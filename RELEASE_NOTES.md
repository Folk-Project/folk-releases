### What's new

Managed-broker jobs drivers — RabbitMQ, AWS SQS, NATS/JetStream, beanstalkd,
Apache Kafka and Google Cloud Pub/Sub ([#39](https://github.com/Folk-Project/folk-releases/issues/39)–[#44](https://github.com/Folk-Project/folk-releases/issues/44)).

The jobs plugin gains six managed-broker backends alongside the existing
memory / redis / embedded drivers. Each is a Cargo feature (off by default), so
a build only links the brokers it uses; the default build stays
`["memory", "redis"]`.

**At-least-once delivery.** The driver contract now leases a message
(`reserve`) and settles it with `ack` (success) or `nack` (terminal failure)
instead of a destructive pop. Broker drivers keep a message in-flight until it
is acked and re-deliver after the lease/visibility window if a worker crashes —
so jobs are no longer lost on crash. memory / redis / embedded keep their
previous at-most-once behaviour (their `ack`/`nack` are no-ops). The in-process
retry/backoff and the app-level `dead_letter_queue` work unchanged across every
driver; without an app-level DLQ, a terminal failure is routed to the broker's
native dead-letter facility (RabbitMQ DLX, beanstalk bury, …).

**Drivers.**

```toml
# RabbitMQ — feature "rabbitmq". Manual ack/nack, prefetch, optional DLX.
[jobs.connections.rabbitmq]
url = "amqp://guest:guest@rabbit:5672/%2f"
prefetch = 10
  [jobs.connections.rabbitmq.queues.emails]
  concurrency = 8

# AWS SQS — feature "sqs". Standard + FIFO (*.fifo), LocalStack via `endpoint`.
[jobs.connections.sqs]
region = "us-east-1"
endpoint = "http://localstack:4566"
  [jobs.connections.sqs.queues.default]

# NATS/JetStream — feature "nats". Durable pull consumers.
[jobs.connections.nats]
url = "nats://nats:4222"
stream = "folk"
  [jobs.connections.nats.queues.events]

# beanstalkd — feature "beanstalk". Tube = queue, native delay + TTR.
[jobs.connections.beanstalk]
host = "beanstalkd"
port = 11300
  [jobs.connections.beanstalk.queues.default]

# Apache Kafka — feature "kafka". Topic = queue, consumer group + manual commit.
[jobs.connections.kafka]
brokers = "kafka:9092"
group_id = "folk"
  [jobs.connections.kafka.queues.events]

# Google Cloud Pub/Sub — feature "pubsub". Topic = queue, pull subscription.
[jobs.connections.pubsub]
project_id = "my-project"
endpoint = "pubsub:8085"           # emulator; empty = real Pub/Sub via ADC
  [jobs.connections.pubsub.queues.events]
```

Select drivers per build in `folk.build.toml`:

```toml
[[plugin]]
crate_name = "folk-plugin-jobs"
version = "0.5"
config_key = "jobs"
features = ["rabbitmq", "sqs"]
```

A connection section whose driver is not compiled in still **fails fast** at
startup with an actionable message. The `[<connection>.]<queue>` addressing from
PHP is unchanged — `->onQueue("rabbitmq.emails")` targets a specific backend.

**Known limitations.** SQS native delay is capped at 900s; RabbitMQ / NATS /
Kafka / Pub/Sub delay is a deferred publish (lost if the server crashes inside
the delay window); Kafka and Pub/Sub do not report queue depth (metric 0). All
six drivers were smoke-tested against real broker containers.

No `folk-api` / `folk-core` / `folk-ext` changes — the `Driver` trait is
internal to the jobs plugin.

**Versions.** folk-plugin-jobs 0.4.0 → **0.5.0**.
