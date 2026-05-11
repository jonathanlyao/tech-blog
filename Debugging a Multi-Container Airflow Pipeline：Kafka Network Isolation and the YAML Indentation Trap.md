# Debugging a Multi-Container Airflow Pipeline: Kafka Network Isolation and the YAML Indentation Trap

After getting Kafka, Spark, and Snowflake all working individually, I thought wiring them together in Airflow would be the easy part. It was not. What followed was an afternoon of containers failing silently, misleading error messages, and one of those bugs where the fix is two characters but finding it takes two hours.

This post covers the two main issues I hit when building an Airflow DAG to orchestrate a CMS Medicare streaming pipeline. Both problems are worth understanding because they'll come up any time you try to connect services across multiple Docker Compose projects.

---

## The Setup

The pipeline I was building looks like this:

```
CMS API → Kafka Producer → Kafka → Spark Streaming → Snowflake → dbt → dbt test
```

All orchestrated by a single Airflow DAG with four tasks:

```
run_cms_producer >> run_spark_streaming >> run_dbt_models >> run_dbt_tests
```

The tricky part: Kafka and Spark live in one Docker Compose project (`CMS_project`), and Airflow lives in a completely separate Docker Compose project (`DOT_project/airflow`). Two separate projects, two separate Docker networks, and a whole set of problems that come with that.

---

## Problem 1: `NoBrokersAvailable` — Kafka Network Isolation

### What the error looked like

The first task, `run_cms_producer`, kept failing immediately. The error in the Airflow task log:

```
kafka.errors.NoBrokersAvailable: NoBrokersAvailable
```

This error means the Kafka client tried to connect to the broker, got nothing back, and gave up. The fix seems obvious: check the broker address. I did. It looked right. The container names were correct. The ports were open. And yet — nothing.

### Why it happened: Docker Compose network isolation

When you run `docker-compose up` in a directory, Docker creates a private network for that project. By default, it's named after the folder: `cms_project_default` for `CMS_project`, and `airflow_default` for `DOT_project/airflow`.

Containers in `cms_project_default` can talk to each other freely. Containers in `airflow_default` can talk to each other freely. But **containers in different projects cannot reach each other by default** — the networks are completely separate.

The first fix I tried was adding Airflow's containers to the CMS network:

```yaml
# In Airflow's docker-compose.yml
networks:
  default:
    name: airflow_default
  cms_network:
    external: true
    name: cms_project_default
```

This seemed to work — `docker inspect` confirmed the Airflow scheduler was in both networks. But the `NoBrokersAvailable` error kept showing up.

### The real problem: how Kafka advertises itself

Here's where it gets less obvious. Kafka doesn't just sit there and accept connections. When a client first connects, Kafka sends back a list of addresses the client should use for all future communication. This list is called the **advertised listeners**.

In my `CMS_project/docker-compose.yml`, the Kafka configuration looked like this:

```yaml
KAFKA_LISTENERS: INTERNAL://0.0.0.0:29092,EXTERNAL://0.0.0.0:9092
KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:29092,EXTERNAL://localhost:9092
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
```

This sets up two listeners:
- `INTERNAL` on port 29092 — intended for containers inside the same Docker Compose project
- `EXTERNAL` on port 9092 — intended for connections from the host machine (your Windows terminal)

When the Airflow container tried to connect to `kafka:29092`, Kafka initially accepted the connection (because Airflow was now on the same Docker network). But then Kafka sent back its advertised address for that listener: `kafka:29092`. When the Airflow container tried to use that address for subsequent requests, it hit the same broker again — which was fine.

So why was it still failing?

The answer is subtle. Even though Airflow was on the `cms_project_default` network, Kafka was treating it as an **internal** client. The `INTERNAL` listener was designed for containers inside `cms_project` — and those containers have a specific network configuration that resolves `kafka` to the right IP. Airflow's container, being from a different Compose project, had slightly different network resolution behavior. The connection would time out at the API version check stage, which is the very first thing the Kafka client does before it can do anything else.

### The fix: a third dedicated listener

The cleanest solution is to add a third listener specifically for cross-project Docker container connections:

```yaml
KAFKA_LISTENERS: INTERNAL://0.0.0.0:29092,EXTERNAL://0.0.0.0:9092,DOCKER://0.0.0.0:39092
KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:29092,EXTERNAL://localhost:9092,DOCKER://kafka:39092
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,DOCKER:PLAINTEXT
KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
```

And expose the new port:

```yaml
ports:
  - "9092:9092"
  - "39092:39092"
```

Now there are three separate listeners, each for a different kind of client:

| Listener | Port | Who uses it |
|---|---|---|
| `INTERNAL` | 29092 | Containers inside `cms_project` (Spark, etc.) |
| `EXTERNAL` | 9092 | Host machine (your terminal, local Python scripts) |
| `DOCKER` | 39092 | Containers from other Docker Compose projects (Airflow) |

In the Airflow DAG, the producer task now uses the `DOCKER` listener:

```python
run_producer = BashOperator(
    task_id="run_cms_producer",
    bash_command="KAFKA_BROKER=kafka:39092 python /opt/airflow/cms_producer.py",
    execution_timeout=timedelta(minutes=15),
)
```

And in `cms_producer.py`, the broker address reads from the environment variable:

```python
import os
KAFKA_BROKER = os.environ.get("KAFKA_BROKER", "localhost:9092")
```

After this change, the connection worked immediately.

### How to debug Kafka connectivity issues

Before changing any configuration, verify the actual connection from inside the container that's having trouble. Don't guess based on port numbers or network diagrams:

```bash
# Copy a test script into the Airflow container
docker cp test_kafka.py airflow-airflow-scheduler-1:/tmp/test_kafka.py

# Run it from inside the container
docker exec airflow-airflow-scheduler-1 python /tmp/test_kafka.py
```

Where `test_kafka.py` contains:

```python
from kafka import KafkaConsumer
c = KafkaConsumer(bootstrap_servers='kafka:39092', consumer_timeout_ms=3000)
print('Connected OK')
```

This tells you immediately whether the network path is open, without having to trigger a full DAG run and wait for logs.

---

## Problem 2: The YAML Indentation Trap

### What the error looked like

Once the Kafka connection was fixed, `run_cms_producer` and `run_spark_streaming` started succeeding. But `run_dbt_models` failed with:

```
Runtime Error
  Could not find profile named 'cms_dbt'
```

dbt couldn't find the profile I had added. I checked the file — `cms_dbt` was right there in `profiles.yml`. Or so I thought.

### Why it happened: YAML nesting is invisible

Here's what the `profiles.yml` actually contained:

```yaml
dbt_dot_flights:
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      # ... other fields ...
  target: dev
  cms_dbt:                    # ← THIS IS THE PROBLEM
    target: dev
    outputs:
      dev:
        type: snowflake
        account: FZPFTPF-LOB40082
        # ...
```

`cms_dbt` was indented under `dbt_dot_flights`. In YAML, indentation defines the structure — so instead of being a separate top-level profile, `cms_dbt` became a field *inside* the `dbt_dot_flights` profile. dbt looked for a top-level key called `cms_dbt`, didn't find one, and reported that the profile was missing.

What the file should have looked like:

```yaml
dbt_dot_flights:
  target: dev
  outputs:
    dev:
      type: snowflake
      # ...

cms_dbt:                      # ← Top-level, no indentation
  target: dev
  outputs:
    dev:
      type: snowflake
      account: FZPFTPF-LOB40082
      # ...
```

The difference is two spaces of indentation, which is completely invisible when you're skimming a file looking for a specific key.

### Why YAML indentation bugs are so hard to catch

Most syntax errors give you a loud, obvious error message. YAML indentation errors often don't — the file is syntactically valid, it just means something different from what you intended. A misindented block doesn't cause a parse error; it causes incorrect data structure, which only surfaces as a logical error later when something tries to use that data.

In this case, `profiles.yml` parsed without any complaint. dbt read it, built the profile registry, and simply didn't find `cms_dbt` as a top-level key — because it wasn't one.

### The quick way to catch YAML structure bugs

Before you commit any YAML file, dump it as parsed Python to see exactly what structure it produces:

```bash
python -c "import yaml; import json; print(json.dumps(yaml.safe_load(open('profiles.yml')), indent=2))"
```

This shows you the actual data structure the parser sees, not what you think you wrote. A misindented block shows up immediately as a nested key instead of a top-level key.

For `profiles.yml` specifically, the top-level keys should be the profile names. If you run the command above and see `cms_dbt` nested inside `dbt_dot_flights` instead of at the top level, you've found the bug.

### The second dbt failure: a typo in `dbt_project.yml`

After fixing the indentation, `run_dbt_models` failed again with a different error:

```
No materialization 'table2' was found for adapter snowflake!
```

This one was a straightforward typo. In `dbt_project.yml`:

```yaml
models:
  cms_dbt:
    marts:
      +materialized: table2    # ← Should be 'table'
```

`table2` is not a valid dbt materialization type. The valid options are `view`, `table`, `incremental`, and `ephemeral`. One character difference, and dbt can't find the materialization.

---

## The Debugging Workflow That Actually Works

After going through all of this, here's the approach I'd use from the start next time:

**Step 1: Test network connectivity before writing any DAG code.**

Don't assume two containers can talk to each other just because they're on the same network. Actually test it from inside the source container before you do anything else.

```bash
docker exec <source-container> python -c "
from kafka import KafkaConsumer
c = KafkaConsumer(bootstrap_servers='<target>:<port>', consumer_timeout_ms=3000)
print('OK')
"
```

**Step 2: Validate YAML files immediately after editing.**

Any time you touch a YAML file, run the parser dump to verify the structure:

```bash
python -c "import yaml; import json; print(json.dumps(yaml.safe_load(open('yourfile.yml')), indent=2))"
```

Two seconds of checking saves an hour of debugging.

**Step 3: Read the Airflow task logs, not the DAG-level logs.**

When a task fails, the DAG-level logs just say "failed." The actual error is in the task-specific log file. In the Airflow UI: click the task (the colored square) → click "Logs." That's where the real error message is.

**Step 4: For Kafka specifically, understand the three listener types.**

If you're running Kafka in Docker and connecting from multiple environments (host machine, same-project containers, cross-project containers), you need a separate listener for each. One listener cannot serve all three use cases cleanly.

---

## Summary

| Problem | Root Cause | Fix |
|---|---|---|
| `NoBrokersAvailable` from Airflow | Two Docker Compose projects on separate networks; Kafka's `INTERNAL` listener not designed for cross-project connections | Add a third `DOCKER` listener on a dedicated port (39092) for cross-project container access |
| `Could not find profile 'cms_dbt'` | YAML indentation error — `cms_dbt` was nested inside `dbt_dot_flights` instead of being a top-level key | Fix indentation; validate with `yaml.safe_load` |
| `No materialization 'table2'` | Typo in `dbt_project.yml` | Change `table2` to `table` |

The Kafka network issue is the one worth spending time understanding. Docker Compose network isolation is intentional and useful, but it creates real headaches when you need services from different projects to communicate. Knowing that Kafka has separate listener types — and that you can add custom ones — gives you a clean, explicit solution rather than hacks like `network_mode: host`.

The YAML bugs are embarrassing in retrospect, but they're also genuinely easy to miss. The fix is always the same: don't trust your eyes on YAML indentation, trust the parser.