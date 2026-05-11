# 调试多容器 Airflow Pipeline：Kafka 网络隔离与 YAML 缩进陷阱

把 Kafka、Spark 和 Snowflake 分别跑通之后，我以为用 Airflow 把它们串起来会是最简单的一步。结果不是。

接下来是一整个下午的容器无声失败、误导性报错，以及那种"修复只需要两个字符，但找到它花了两个小时"的 bug。

这篇文章记录了我在构建 CMS Medicare Streaming Pipeline 的 Airflow DAG 时踩到的两个主要问题。这两个问题都值得认真理解，因为任何时候你想把多个 Docker Compose 项目的服务连接在一起，都会遇到它们。

---

## 背景

我要构建的 pipeline 是这样的：

```
CMS API → Kafka Producer → Kafka → Spark Streaming → Snowflake → dbt → dbt test
```

用一个 Airflow DAG 的四个 task 来编排：

```
run_cms_producer >> run_spark_streaming >> run_dbt_models >> run_dbt_tests
```

麻烦的地方在于：Kafka 和 Spark 在一个 Docker Compose 项目里（`CMS_project`），Airflow 在完全独立的另一个项目里（`DOT_project/airflow`）。两个独立项目，两个独立的 Docker 网络，以及由此带来的一系列问题。

---

## 问题一：`NoBrokersAvailable` —— Kafka 网络隔离

### 错误长什么样

第一个 task `run_cms_producer` 一直立刻失败。Airflow task 日志里的错误：

```
kafka.errors.NoBrokersAvailable: NoBrokersAvailable
```

这个错误的意思是：Kafka 客户端尝试连接 broker，没有收到任何响应，然后放弃了。解决办法看起来很明显：检查 broker 地址。我检查了。看起来没问题。容器名称是对的，端口是开放的。但就是连不上。

### 为什么会发生：Docker Compose 网络隔离

当你在一个目录里运行 `docker-compose up`，Docker 会为这个项目创建一个私有网络。默认情况下，网络名称是文件夹名加上 `_default`：`CMS_project` 的网络叫 `cms_project_default`，Airflow 的网络叫 `airflow_default`。

`cms_project_default` 里的容器可以互相通信。`airflow_default` 里的容器也可以互相通信。但**不同项目的容器默认无法互相访问**——网络是完全隔离的。

我尝试的第一个修复方案是把 Airflow 的容器加入 CMS 的网络：

```yaml
# 在 Airflow 的 docker-compose.yml 里
networks:
  default:
    name: airflow_default
  cms_network:
    external: true
    name: cms_project_default
```

这看起来生效了——`docker inspect` 确认 Airflow scheduler 已经在两个网络里。但 `NoBrokersAvailable` 的错误还是继续出现。

### 真正的问题：Kafka 如何广播自己的地址

这里有一个不那么显眼的机制。Kafka 不只是被动地等待连接。当客户端第一次连接时，Kafka 会返回一个地址列表，告诉客户端后续所有通信应该用哪些地址。这个列表叫做 **advertised listeners（广播监听地址）**。

我的 `CMS_project/docker-compose.yml` 里，Kafka 的配置是这样的：

```yaml
KAFKA_LISTENERS: INTERNAL://0.0.0.0:29092,EXTERNAL://0.0.0.0:9092
KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:29092,EXTERNAL://localhost:9092
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT
KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
```

这里设置了两个 listener：
- `INTERNAL`，端口 29092 —— 给同一个 Docker Compose 项目内的容器用
- `EXTERNAL`，端口 9092 —— 给宿主机上的连接用（比如你的终端、本地 Python 脚本）

当 Airflow 容器尝试连接 `kafka:29092` 时，Kafka 接受了初始连接（因为 Airflow 现在在同一个 Docker 网络里）。但随后 Kafka 返回了这个 listener 对应的广播地址：`kafka:29092`。

问题的根源在这里：`INTERNAL` listener 是为 `cms_project` 内部的容器设计的，这些容器有特定的网络配置，能正确解析 `kafka` 这个 hostname。Airflow 容器来自不同的 Compose 项目，网络解析行为略有差异。连接会在 API 版本检查阶段超时——这是 Kafka 客户端在能做任何事情之前必须完成的第一步。

用一张图来理解三种 listener 的职责：

```
宿主机（你的 Windows 终端）
    ↓ 使用 EXTERNAL listener（localhost:9092）
Kafka 容器
    ↑ 使用 INTERNAL listener（kafka:29092）
cms_project 内的其他容器（Spark 等）

Airflow 容器（来自另一个 Compose 项目）
    ↓ 以前没有专属 listener → 连接失败
    ↓ 加了 DOCKER listener（kafka:39092）→ 连接成功
```

### 解决方案：加第三个专用 listener

最干净的解决方案是加一个专门给跨项目 Docker 容器使用的第三个 listener：

```yaml
KAFKA_LISTENERS: INTERNAL://0.0.0.0:29092,EXTERNAL://0.0.0.0:9092,DOCKER://0.0.0.0:39092
KAFKA_ADVERTISED_LISTENERS: INTERNAL://kafka:29092,EXTERNAL://localhost:9092,DOCKER://kafka:39092
KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: INTERNAL:PLAINTEXT,EXTERNAL:PLAINTEXT,DOCKER:PLAINTEXT
KAFKA_INTER_BROKER_LISTENER_NAME: INTERNAL
```

同时在 `ports` 里暴露新端口：

```yaml
ports:
  - "9092:9092"
  - "39092:39092"
```

现在三个 listener 各司其职：

| Listener | 端口 | 使用者 |
|---|---|---|
| `INTERNAL` | 29092 | `cms_project` 内部容器（Spark 等） |
| `EXTERNAL` | 9092 | 宿主机（终端、本地脚本） |
| `DOCKER` | 39092 | 其他 Docker Compose 项目的容器（Airflow） |

在 Airflow DAG 里，producer task 改用 `DOCKER` listener：

```python
run_producer = BashOperator(
    task_id="run_cms_producer",
    bash_command="KAFKA_BROKER=kafka:39092 python /opt/airflow/cms_producer.py",
    execution_timeout=timedelta(minutes=15),
)
```

在 `cms_producer.py` 里，broker 地址从环境变量读取：

```python
import os
KAFKA_BROKER = os.environ.get("KAFKA_BROKER", "localhost:9092")
```

改完之后，连接立刻成功。

### 如何调试 Kafka 连通性问题

在改任何配置之前，先从出问题的容器内部直接测试连接。不要靠猜端口号或者看网络图：

```bash
# 把测试脚本复制进 Airflow 容器
docker cp test_kafka.py airflow-airflow-scheduler-1:/tmp/test_kafka.py

# 从容器内部运行
docker exec airflow-airflow-scheduler-1 python /tmp/test_kafka.py
```

`test_kafka.py` 的内容：

```python
from kafka import KafkaConsumer
c = KafkaConsumer(bootstrap_servers='kafka:39092', consumer_timeout_ms=3000)
print('Connected OK')
```

这能立刻告诉你网络通路是否畅通，不需要触发完整的 DAG 运行然后等日志。

---

## 问题二：YAML 缩进陷阱

### 错误长什么样

Kafka 连接问题修好之后，`run_cms_producer` 和 `run_spark_streaming` 开始成功运行。但 `run_dbt_models` 失败了：

```
Runtime Error
  Could not find profile named 'cms_dbt'
```

dbt 找不到我添加的 profile。我检查了文件——`cms_dbt` 明明就在 `profiles.yml` 里。至少我以为是这样。

### 为什么会发生：YAML 嵌套是无声的

`profiles.yml` 的实际内容是这样的：

```yaml
dbt_dot_flights:
  outputs:
    dev:
      type: snowflake
      account: "{{ env_var('SNOWFLAKE_ACCOUNT') }}"
      # ... 其他字段 ...
  target: dev
  cms_dbt:                    # ← 问题就在这里
    target: dev
    outputs:
      dev:
        type: snowflake
        account: FZPFTPF-LOB40082
        # ...
```

`cms_dbt` 被缩进在 `dbt_dot_flights` 下面了。在 YAML 里，缩进决定层级结构——所以 `cms_dbt` 不是一个独立的顶层 profile，而是变成了 `dbt_dot_flights` 内部的一个字段。dbt 查找名为 `cms_dbt` 的顶层 key，没找到，就报告说这个 profile 不存在。

正确的写法应该是这样：

```yaml
dbt_dot_flights:
  target: dev
  outputs:
    dev:
      type: snowflake
      # ...

cms_dbt:                      # ← 顶层，没有任何缩进
  target: dev
  outputs:
    dev:
      type: snowflake
      account: FZPFTPF-LOB40082
      # ...
```

区别是两个空格的缩进，肉眼快速扫描文件时几乎看不出来。

### 为什么 YAML 缩进 bug 这么难发现

大多数语法错误会给你一个清晰、明显的报错信息。YAML 缩进错误通常不会——文件在语法上是完全有效的，只是表达的含义和你想的不一样。一个缩进错误的 block 不会导致解析报错，它只会产生错误的数据结构，只有在后续某个地方尝试使用这个数据时才会以逻辑错误的形式暴露出来。

在这个案例里，`profiles.yml` 解析时没有任何报错。dbt 读取了它，构建了 profile 注册表，然后只是没有找到 `cms_dbt` 作为顶层 key——因为它根本不是顶层 key。

### 快速发现 YAML 结构问题的方法

每次修改 YAML 文件之后，用解析器把结果打印出来，直接看实际产生的数据结构：

```bash
python -c "import yaml; import json; print(json.dumps(yaml.safe_load(open('profiles.yml')), indent=2))"
```

这会显示解析器实际看到的数据结构，而不是你以为自己写的东西。缩进错误的 block 会立刻显示为嵌套 key 而不是顶层 key。

对于 `profiles.yml`，顶层 key 应该是各个 profile 的名称。如果你运行上面的命令，看到 `cms_dbt` 嵌套在 `dbt_dot_flights` 里面而不是在顶层，bug 就找到了。

### 第二个 dbt 失败：`dbt_project.yml` 里的拼写错误

修好缩进之后，`run_dbt_models` 再次失败，这次是不同的错误：

```
No materialization 'table2' was found for adapter snowflake!
```

这个很直接——就是个拼写错误。在 `dbt_project.yml` 里：

```yaml
models:
  cms_dbt:
    marts:
      +materialized: table2    # ← 应该是 'table'
```

`table2` 不是 dbt 的合法 materialization 类型。有效的选项只有 `view`、`table`、`incremental` 和 `ephemeral`。一个字符的差别，dbt 就完全找不到对应的实现。

---

## 实际有效的调试工作流

经历了这一切之后，如果下次重来，我会从一开始就用这个方法：

**第一步：写任何 DAG 代码之前，先测试网络连通性。**

不要因为两个容器在同一个网络里就假设它们能互相通信。从源容器内部实际测试一下，再做别的事情：

```bash
docker exec <源容器> python -c "
from kafka import KafkaConsumer
c = KafkaConsumer(bootstrap_servers='<目标>:<端口>', consumer_timeout_ms=3000)
print('OK')
"
```

**第二步：编辑 YAML 文件之后立刻验证。**

每次改动 YAML 文件，就跑一次解析器打印：

```bash
python -c "import yaml; import json; print(json.dumps(yaml.safe_load(open('yourfile.yml')), indent=2))"
```

两秒钟的检查，省掉一个小时的调试。

**第三步：看 Airflow task 级别的日志，不是 DAG 级别的。**

当一个 task 失败时，DAG 级别的日志只会说"failed"。真正的错误在 task 专属的日志文件里。在 Airflow UI 里：点击 task（那个有颜色的方块）→ 点击"Logs"。真正的报错在那里。

**第四步：对于 Kafka，理解三种 listener 类型。**

如果你在 Docker 里跑 Kafka，并且需要从多个环境连接（宿主机、同项目容器、跨项目容器），你需要为每种场景设置独立的 listener。一个 listener 无法干净地服务所有三种使用场景。

---

## 总结

| 问题 | 根本原因 | 解决方案 |
|---|---|---|
| Airflow 报 `NoBrokersAvailable` | 两个 Docker Compose 项目处于独立网络；Kafka 的 `INTERNAL` listener 不是为跨项目连接设计的 | 在专用端口（39092）上添加第三个 `DOCKER` listener，供跨项目容器使用 |
| `Could not find profile 'cms_dbt'` | YAML 缩进错误——`cms_dbt` 被嵌套在 `dbt_dot_flights` 内部而不是作为顶层 key | 修正缩进；用 `yaml.safe_load` 验证结构 |
| `No materialization 'table2'` | `dbt_project.yml` 里的拼写错误 | 把 `table2` 改成 `table` |

Kafka 网络问题是最值得花时间理解的。Docker Compose 网络隔离是有意为之的设计，也是有用的，但当你需要不同项目的服务互相通信时，它会带来真实的麻烦。知道 Kafka 有独立的 listener 类型，而且你可以自定义新的，就能给出一个干净、明确的解决方案，而不是用 `network_mode: host` 这种绕过问题的方式。

YAML 的 bug 事后看起来很尴尬，但确实很容易漏掉。解决方法永远是同一个：不要相信肉眼看 YAML 缩进，相信解析器。