# Why My Spark Container Keeps Exiting — Docker PID 1 and the Daemon Trap

I spent an embarrassing amount of time staring at my terminal, watching Spark
containers start and immediately die. Three different attempts, three different
failure modes, all in the same afternoon. If you're setting up Spark inside
Docker and your container just... vanishes, this post is for you.

---

## The Setup

I'm building a CMS Medicare streaming pipeline — pulling hospital charge data
from the CMS public API, pushing it through Kafka, processing it with Spark
Structured Streaming, and landing the results in Snowflake. The whole stack
runs in Docker Compose. Kafka and ZooKeeper came up without a hitch. Spark did
not.

Here's what my `docker-compose.yml` looked like at the start:

```yaml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on: [zookeeper]
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1

  spark:
    image: bitnami/spark:3.5
    depends_on: [kafka]
    environment:
      SPARK_MODE: master

  spark-worker:
    image: bitnami/spark:3.5
    depends_on: [spark]
    environment:
      SPARK_MODE: worker
      SPARK_MASTER_URL: spark://spark:7077
```

Looked reasonable enough. It wasn't.

---

## Attempt 1 — The Image That No Longer Exists

```
Error response from daemon: failed to resolve reference
"docker.io/bitnami/spark:3.5": not found
```

`bitnami/spark:3.5` had been pulled from Docker Hub. I tried `3.5.3`. Gone.
Tried `bitnami/spark:3`. Also gone. The entire Bitnami Spark image line had
been removed or reorganized with no notice.

This is the first thing worth remembering before we even get to the real
problem: **third-party images on Docker Hub can disappear at any time.** There
is no deprecation warning, no migration guide, no email. One day the image is
there, the next it isn't. For anything that needs to be reproducible — CI
pipelines, team environments, production deployments — you either pin to a
verified digest or mirror the image in a private registry.

I switched to the Apache official image: `apache/spark:3.5.1-python3`. That
one pulled without issues, so I moved on.

---

## Attempt 2 — Wrong Environment Variables

I updated the image name but kept the same environment variable:

```yaml
spark:
  image: apache/spark:3.5.1-python3
  environment:
    SPARK_MODE: master
```

`docker-compose up -d` reported all containers as "Started." But `docker ps`
only showed two running containers — Kafka and ZooKeeper. The Spark containers
had already exited.

The problem here is a common assumption: if two images contain the same
software, they must work the same way. They don't. **`SPARK_MODE` is a
Bitnami-specific environment variable.** The Apache official image has never
heard of it.

Bitnami's image ships with a custom entrypoint script that reads `SPARK_MODE`
at startup and decides whether to launch a master or a worker process. It's a
convenience layer that Bitnami built on top of vanilla Spark to make their
image easier to configure. The Apache official image has none of this. Its
default entrypoint (`/opt/entrypoint.sh`) simply executes whatever command you
pass to it. If you don't pass a meaningful command, it finishes and exits.

The lesson: switching between images from different publishers is not just
swapping the `image:` field. Different publishers package the same software
with different entrypoints, different environment variables, and different
directory layouts. Before you can use an image correctly, you need to
understand how *that specific image* expects to be started.

---

## Attempt 3 — The Real Trap: `start-master.sh`

OK, so the Apache image needs an explicit command. Spark comes bundled with
`start-master.sh`. That seems like the right tool:

```yaml
spark:
  image: apache/spark:3.5.1-python3
  command: /opt/spark/sbin/start-master.sh
```

Same result. `docker-compose up -d` reported "Started." `docker ps` showed no
Spark container. This one took longer to debug, because the failure reason is
less obvious.

The container was starting. Spark Master was launching inside it. And then
everything was shutting down within a fraction of a second. To understand why,
you need to know one foundational rule about how Docker containers work.

---

## The Core Rule: Docker Containers Live and Die with PID 1

Every container has a main process — the one specified by `CMD`, `ENTRYPOINT`,
or `command` in your Compose file. Inside the container, this process is
assigned **PID 1**. When PID 1 exits, the container exits. This is not
configurable. It is how Docker is designed to work:

```
PID 1 is running  →  container is running
PID 1 exits       →  container exits immediately
```

Now look at what `start-master.sh` actually does internally (simplified):

```bash
#!/bin/bash
nohup java -cp $SPARK_CLASSPATH org.apache.spark.deploy.master.Master &
echo "Master started."
exit 0
```

See that `&` at the end of the `java` command? That's the problem. It puts the
Spark Master process into the **background**. The shell script — which is
running as PID 1 — spawns a child Java process, prints a confirmation message,
and then calls `exit 0`. The moment it does that, Docker kills the container,
taking the background Java process with it.

Here's the exact timeline:

```
t=0.0s  Container starts; PID 1 = start-master.sh (bash process)
t=0.1s  Bash forks a Java process (Spark Master) and sends it to the background
t=0.2s  Bash script finishes → exit 0 → PID 1 terminates
t=0.2s  Docker detects PID 1 exit → tears down the container
t=0.2s  The background Java process is a child of PID 1; it gets killed too
```

Spark Master was alive for about 0.2 seconds. Which is why `docker-compose up
-d` showed "Started" — the container did start. It just immediately exited
afterward.

This is what I call the daemon trap. `start-master.sh` was written for
bare-metal servers and virtual machines, where you start a background daemon,
let the script exit, and the daemon keeps running because the OS keeps it
alive. Docker doesn't work that way. Docker is watching PID 1 and only PID 1.
The moment that process is gone, everything inside the container is gone.

---

## Why Kafka and ZooKeeper Didn't Have This Problem

Once I understood the PID 1 rule, I got curious about why Confluent's images
worked fine from the very beginning. The answer is in how their entrypoint
scripts are written. Here's the key line from `cp-kafka`'s startup (simplified):

```bash
exec kafka-server-start /etc/kafka/server.properties
```

That `exec` makes all the difference. In bash, `exec` **replaces the current
process** with the specified command rather than forking a child. The shell
doesn't start Kafka and then wait around — the shell *becomes* Kafka. Kafka
inherits PID 1, runs in the foreground, and blocks forever. The container stays
alive as long as Kafka is running.

Compare that to the Spark approach:

| Image | What PID 1 Does | Result |
|---|---|---|
| `cp-kafka` | `exec kafka-server-start` — foreground, blocks indefinitely | ✅ Container stays alive |
| `cp-zookeeper` | `exec zookeeper-server-start` — foreground, blocks indefinitely | ✅ Container stays alive |
| `apache/spark` + `start-master.sh` | Forks Java to background with `&`, script exits | ❌ Container exits immediately |

The entire difference comes down to `&` versus `exec`. Background daemon versus
foreground process. Two characters with completely opposite outcomes in Docker.

---

## Four Ways to Fix It

### Fix A: Keep the Container Alive with `tail -f /dev/null`

This is the approach I ended up using:

```yaml
spark:
  image: apache/spark:3.5.1-python3
  command: ["tail", "-f", "/dev/null"]
  volumes:
    - ./spark-apps:/opt/spark-apps
```

`tail -f /dev/null` is a command that never finishes. It watches `/dev/null`
for new content that will never arrive. PID 1 stays alive, and so does the
container. When I need to submit a Spark job, I use `docker exec`:

```bash
docker exec my-spark-container \
  /opt/spark/bin/spark-submit \
  /opt/spark-apps/my_job.py
```

The container becomes a persistent, reachable environment rather than a
self-starting service. Simple, reliable, and good enough for development and
portfolio projects.

**When to use it:** local development, one-off job submission, situations where
you don't need Spark Master/Worker to auto-start.

### Fix B: Run Spark Master Directly in the Foreground

Instead of going through `start-master.sh`, call the Java class directly. This
skips the wrapper script entirely and runs the Master process in the foreground
as PID 1:

```yaml
command: >
  bash -c "
  /opt/spark/bin/spark-class org.apache.spark.deploy.master.Master
  --host spark --port 7077 --webui-port 8080
  "
```

No background daemon, no premature exit. The Master is PID 1, runs in the
foreground, and the container lives as long as the Master does.

**When to use it:** when you actually need a running Spark Master/Worker
cluster inside Docker, not just a container to submit jobs into.

### Fix C: Write a Custom Entrypoint Script

This approach starts the daemon the normal way, then uses a blocking process to
keep PID 1 alive:

```bash
#!/bin/bash
# custom-entrypoint.sh
/opt/spark/sbin/start-master.sh   # starts Spark Master in the background
tail -f /opt/spark/logs/*         # blocks forever and streams logs to stdout
```

```yaml
volumes:
  - ./custom-entrypoint.sh:/opt/custom-entrypoint.sh
command: bash /opt/custom-entrypoint.sh
```

`start-master.sh` launches the daemon as usual. Then `tail -f` takes over as
the blocking process, keeping the bash script (PID 1) alive. You also get
Spark's log output forwarded to `docker logs` as a side benefit.

**When to use it:** when you want Spark to auto-start inside the container and
also want log output accessible via Docker.

### Fix D: Use an Image Designed for Docker

Some images handle all of this correctly out of the box — `jupyter/pyspark-notebook`
is one example. Their entrypoints are built around `exec` and foreground
processes from the start. You don't have to think about any of this.

The tradeoff is that you're depending on a third party to maintain the image
and keep it available, which — as Attempt 1 demonstrated — isn't always a safe
assumption.

**When to use it:** quick prototyping, when you don't need control over the
Spark version or configuration.

---

## Summary

Here's the short version of everything above:

- Docker containers exit when PID 1 exits. Always.
- `start-master.sh` backgrounds the Spark process with `&` and then exits,
  which kills the container.
- Confluent's images use `exec`, which makes the service itself become PID 1,
  keeping the container alive.
- The fix is to ensure PID 1 is a foreground process that never returns:
  `tail -f /dev/null`, a direct class invocation, or a custom entrypoint.

The broader point is that **Docker containers are not virtual machines.** On a
VM, running a daemon in the background and exiting the startup script is
completely normal. In Docker, the startup script *is* the container. The
lifecycle of your service is tied directly to the lifecycle of PID 1.

Once you understand that rule, most "why does my container keep exiting"
questions answer themselves. And you'll probably start noticing `exec` in
well-written entrypoint scripts everywhere — because whoever wrote them already
knew this.

Three patterns to watch for in any startup script:

- **`command &`** — background execution, PID 1 will exit shortly after
- **`exec command`** — replaces PID 1, container lives as long as the process does
- **`nohup command &`** — classic daemon pattern, same problem as `&` in Docker

Spot any of these and you'll know immediately whether a container will survive
startup or not.