<p align="center">
  <h1 align="center">☁️ BesPIN</h1>
  <p align="center">
    <strong>Platform Infrastructure Native</strong><br />
    No account. No auth token. No carbonite. Just <code>docker compose up</code>.
  </p>
</p>

<p align="center">
  <a href="#quick-start">Quick Start</a> ·
  <a href="#features">Features</a> ·
  <a href="#supported-services">Services</a> ·
  <a href="#sdk-integration">SDKs</a> ·
  <a href="#testcontainers">Testcontainers</a> ·
  <a href="#migrating-from-localstack">Migration</a> ·
  <a href="https://floci.io/floci/">Docs</a>
</p>

---

## What is BesPIN?

**BesPIN** (**B**ackend **E**mulator for **S**erverless **P**latform **I**nfrastructure **N**atively) is a fork of [Floci](https://github.com/floci-io/floci) — a free, open-source local AWS emulator for development, testing, and CI.

It gives you AWS-shaped services on your machine without requiring a cloud account, an auth token, or paid feature gates. Point your AWS SDK, CLI, Terraform, CDK, OpenTofu, or test suite at `http://localhost:4566` and keep your existing workflows.

> *Named after Bespin, the gas giant from Star Wars where Cloud City floats among the clouds — just like your local infrastructure.*
> *LocalStack froze their community edition. Han Solo was frozen in carbonite. Coincidence? We think not.*

## Quick Start

Run BesPIN with Docker Compose:

```yaml
services:
  bespin:
    image: floci/floci:latest
    ports:
      - "4566:4566"
```

```bash
docker compose up
```

Export the AWS environment variables:

```bash
export AWS_ENDPOINT_URL=http://localhost:4566
export AWS_DEFAULT_REGION=us-east-1
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
```

Use your existing AWS tools normally:

```bash
aws s3 mb s3://my-bucket
aws dynamodb create-table \
  --table-name demo-table \
  --attribute-definitions AttributeName=pk,AttributeType=S \
  --key-schema AttributeName=pk,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
aws dynamodb list-tables
```

All AWS services are available at `http://localhost:4566`. Any region works. Credentials can be any non-empty values.

<details>
<summary>With Lambda, ElastiCache and RDS (Docker socket required)</summary>

```yaml
services:
  bespin:
    image: floci/floci:latest
    ports:
      - "4566:4566"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - ./data:/app/data
    environment:
      BESPIN_STORAGE_MODE: hybrid
```

</details>

## Features

<details open>
<summary><strong>Local AWS without the cloud account</strong></summary>

Run AWS-compatible services locally without an AWS account, auth token, or paid feature gates.

</details>

<details>
<summary><strong>Real Docker where fidelity matters</strong></summary>

Lambda, RDS, Neptune, ElastiCache, MSK, ECS, EC2, EKS, OpenSearch, and CodeBuild use real Docker-backed execution instead of shallow mocks.

</details>

<details>
<summary><strong>Drop-in AWS compatibility</strong></summary>

Point standard AWS clients at `http://localhost:4566`. Existing credentials, regions, SDKs, CLI commands, and IaC workflows stay familiar.

</details>

<details>
<summary><strong>Fast enough for CI</strong></summary>

The native image starts in milliseconds and keeps idle memory low, making it practical for local development and test pipelines.

</details>

<details>
<summary><strong>Configurable persistence</strong></summary>

Choose from in-memory, persistent, hybrid, and write-ahead log storage depending on the durability profile you need.

</details>

## Why BesPIN?

LocalStack's community edition [sunset in March 2026](https://blog.localstack.cloud/the-road-ahead-for-localstack/), requiring auth tokens and freezing security updates. Like Han Solo in carbonite — functional in appearance, but stuck.

BesPIN is the no-strings-attached alternative, forked from Floci.

| Capability | BesPIN | LocalStack Community |
|---|:---:|:---:|
| Auth token required | No | Yes |
| Security updates | Yes | Frozen |
| Startup time | ~24 ms | ~3.3 s |
| Idle memory | ~13 MiB | ~143 MiB |
| Docker image size | ~90 MB | ~1.0 GB |
| License | MIT | Restricted |
| API Gateway v2 / HTTP API | Yes | No |
| Cognito | Yes | No |
| RDS, ElastiCache, MSK | Real Docker | No |
| Neptune (graph DB + Gremlin WebSocket) | Real Docker | No |
| ECS, EC2, EKS | Real Docker | No |
| CodeBuild | Real Docker execution | No |
| Native binary | ~40 MB | No |

**52 AWS services. Broad coverage. Free forever.**

## Architecture Overview

```mermaid
flowchart LR
    Client["AWS SDK / CLI"]
    subgraph BesPIN ["BesPIN, port 4566"]
        Router["HTTP Router\nJAX-RS / Vert.x"]
        subgraph Stateless ["Stateless Services"]
            A["SSM · SQS · SNS\nIAM · STS · KMS\nSecrets Manager · SES\nCognito · Kinesis\nEventBridge · Scheduler · AppConfig\nCloudWatch · Step Functions\nCloudFormation · ACM · Config\nAPI Gateway · ELB v2 · Auto Scaling\nCodeDeploy · Backup · Bedrock Runtime · Route53 · Transfer"]
        end
        subgraph Stateful ["Stateful Services"]
            B["S3 · DynamoDB\nDynamoDB Streams"]
        end
        subgraph Containers ["Container Services"]
            C["Lambda\nElastiCache\nRDS\nNeptune\nECS\nEC2\nMSK\nEKS\nOpenSearch\nCodeBuild"]
            D["Athena -> bespin-duck\nDuckDB sidecar"]
        end
        Router --> Stateless
        Router --> Stateful
        Router --> Containers
        Stateless & Stateful --> Store[("StorageBackend\nmemory · hybrid · persistent · wal")]
    end
    Docker["Docker Engine"]
    Client -->|"HTTP :4566\nAWS wire protocol"| Router
    Containers -->|"Docker API\nIAM / SigV4 auth"| Docker
```

## Supported Services

BesPIN supports local emulation for application services, data services, eventing, identity, infrastructure, billing, and container-backed workloads.

| Category | Services |
|---|---|
| Core app services | S3, SQS, SNS, DynamoDB, Lambda, IAM, STS, KMS, Secrets Manager |
| Events and workflows | EventBridge, EventBridge Pipes, EventBridge Scheduler, Step Functions, CloudWatch Logs, CloudWatch Metrics |
| API and identity | API Gateway REST, API Gateway v2, Cognito, ACM, Route53 |
| Containers and compute | ECS, EC2, EKS, CodeBuild, CodeDeploy, Auto Scaling, ELB v2 |
| Graph database | Neptune |
| Data and analytics | Athena, Glue, Firehose, OpenSearch, Textract |
| Messaging and transfer | SES, SES v2, Kinesis, Transfer Family |
| Cost and billing | Pricing, Cost Explorer, Cost and Usage Reports, BCM Data Exports |
| Backup and config | AWS Backup, AWS Config, AppConfig, AppConfigData, CloudFormation |

For operation-level compatibility, see the [Floci Services Overview](https://floci.io/floci/services/).

<details>
<summary>Detailed service notes</summary>

| Service | How it works | Notable features |
|---|---|---|
| SSM Parameter Store | In-process | Version history, labels, SecureString, tagging |
| SSM Run Command | In-process | SendCommand, GetCommandInvocation, ListCommands, CancelCommand, agent polling via ec2messages |
| SQS | In-process | Standard and FIFO queues, DLQ, visibility timeout, batch operations, tagging |
| SNS | In-process | Topics, subscriptions, SQS, Lambda and HTTP delivery, tagging |
| S3 | In-process | Versioning, multipart upload, pre-signed URLs, Object Lock, event notifications |
| DynamoDB | In-process | GSI, LSI, Query, Scan, TTL, transactions, batch operations |
| DynamoDB Streams | In-process | Shard iterators, records, Lambda event source mapping trigger |
| Lambda | Real Docker | Runtime environment, execution model, warm container pool, aliases, Function URLs |
| API Gateway REST | In-process | Resources, methods, stages, Lambda proxy, MOCK integrations, AWS integrations |
| API Gateway v2 | In-process | HTTP APIs, routes, integrations, JWT authorizers, stages |
| IAM | In-process | Users, roles, groups, policies, instance profiles, access keys |
| STS | In-process | AssumeRole, WebIdentity, SAML, GetFederationToken, GetSessionToken |
| Cognito | In-process | User pools, app clients, auth flows, JWKS and OpenID well-known endpoints |
| KMS | In-process | Encrypt, decrypt, sign, verify, data keys, aliases |
| Kinesis | In-process | Streams, shards, enhanced fan-out, split and merge |
| Secrets Manager | In-process | Versioning, resource policies, tagging |
| Step Functions | In-process | ASL execution, task tokens, execution history |
| CloudFormation | In-process | Stacks, change sets, resource provisioning |
| EventBridge | In-process | Custom buses, rules, SQS, SNS and Lambda targets |
| EventBridge Pipes | In-process | Poller-based integration connecting SQS, Kinesis, DynamoDB, and MSK sources to targets with optional filtering |
| EventBridge Scheduler | In-process | Schedule groups, schedules, flexible time windows, retry policies, DLQs |
| CloudWatch Logs | In-process | Log groups, streams, ingestion, filtering |
| CloudWatch Metrics | In-process | Custom metrics, statistics, alarms |
| ElastiCache | Real Docker | Redis / Valkey protocol, IAM auth, SigV4 validation |
| RDS | Real Docker | PostgreSQL, MySQL, MariaDB, IAM auth, JDBC-compatible engines |
| Neptune | Real Docker | Graph DB via TinkerPop Gremlin Server; RDS-shaped control plane; Gremlin WebSocket on port 8182 with SigV4 proxy |
| MSK | Real Docker | Kafka-compatible broker via Redpanda |
| Athena | In-process with DuckDB sidecar | Real SQL execution over S3 and Glue-backed views |
| Glue | In-process | Data Catalog, Schema Registry, tables consumed by Athena |
| Data Firehose | In-process | Streaming delivery, NDJSON flush to S3 |
| ECS | Real Docker | Clusters, task definitions, tasks, services, capacity providers, task sets |
| EC2 | Real Docker | RunInstances launches containers, SSH key injection, UserData, IMDS, VPC resources |
| ACM | In-process | Certificate issuance and validation lifecycle |
| ECR | In-process with real registry | Repositories, docker push / pull, image-backed Lambda functions |
| SES | In-process | Send email, raw email, identity verification, DKIM, templates |
| SES v2 | In-process | REST JSON API, identities, DKIM, account sending, templates |
| OpenSearch | Real Docker | Domain CRUD, tags, versions, instance types, upgrade stubs |
| AppConfig | In-process | Applications, environments, profiles, hosted versions, deployments |
| AppConfigData | In-process | Configuration sessions and dynamic configuration retrieval |
| Bedrock Runtime | In-process stub | Dummy Converse and InvokeModel responses for local development |
| EKS | Real Docker, mock mode available | k3s clusters with live Kubernetes API server |
| ELB v2 | In-process | ALB, NLB, target groups, listeners, routing rules, Lambda targets, tags |
| CodeBuild | In-process with real Docker | Real buildspec execution, CloudWatch logs, S3 artifacts |
| CodeDeploy | In-process with Lambda traffic shifting | Deployment groups, configs, lifecycle hooks, auto-rollback |
| Auto Scaling | In-process with reconciler | Launch configs, ASGs, desired capacity reconciliation, lifecycle hooks |
| AWS Backup | In-process | Vaults, backup plans, selections, simulated job lifecycle, recovery points |
| AWS Config | In-process | Config rules, configuration recorders, delivery channels, conformance packs, tagging |
| Route53 | In-process | Hosted zones, SOA and NS records, resource record sets, change tracking, tagging |
| Transfer Family | In-process | Server lifecycle, user management, SSH key import, tagging |
| Textract | In-process stub | API-compatible stubs, dummy block data, async job simulation |
| Pricing | In-process with static snapshot | Product discovery, attributes, price list files, pagination |
| Cost Explorer | In-process | Cost synthesized from BesPIN resource state and pricing snapshots |
| Cost and Usage Reports | In-process with bespin-duck sidecar | CUR 2.0 and FOCUS 1.2 columns, account-scoped storage, Parquet emission |
| BCM Data Exports | In-process | Export lifecycle, executions, update and delete operations |

</details>

## Real Docker Integration

BesPIN uses real Docker containers when in-process emulation would reduce fidelity. This applies to stateful databases, connection-heavy protocols, runtimes, and build systems.

| Service | Default image | What is real |
|---|---|---|
| Lambda | `public.ecr.aws/lambda/<runtime>` | AWS runtime environment, execution model, warm container pool |
| ElastiCache | `valkey/valkey:8` | Redis / Valkey protocol, ACL-based IAM auth, SigV4 validation |
| RDS PostgreSQL | `postgres:16-alpine` | PostgreSQL engine, IAM auth, JDBC-compatible access |
| RDS MySQL / Aurora | `mysql:8.0` | MySQL engine, IAM auth, JDBC-compatible access |
| RDS MariaDB | `mariadb:11` | MariaDB engine, IAM auth, JDBC-compatible access |
| Neptune | `tinkerpop/gremlin-server:3.7.3` | TinkerPop Gremlin Server; Gremlin WebSocket on port 8182; SigV4 auth proxy |
| MSK | `redpandadata/redpanda:latest` | Kafka-compatible broker via Redpanda |
| EC2 | AMI-mapped Linux images | Linux containers, SSH key injection, UserData, IMDS, IAM credentials |
| ECS | User-specified task image | Container lifecycle, start, stop, health checks |
| EKS | `rancher/k3s:latest` | Kubernetes API server via k3s |
| CodeBuild | User-specified environment image | Buildspec execution, log streaming, S3 artifact upload |
| OpenSearch | `opensearchproject/opensearch:2` | Full OpenSearch engine with REST API |
| ECR | `registry:2` | OCI-compatible registry for docker push and docker pull |

Docker-backed services require the Docker socket:

```bash
docker run -d --name bespin \
  -p 4566:4566 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -u root \
  floci/floci:latest
```

### Overriding default images

| Variable | Default |
|---|---|
| `BESPIN_SERVICES_ELASTICACHE_DEFAULT_IMAGE` | `valkey/valkey:8` |
| `BESPIN_SERVICES_RDS_DEFAULT_POSTGRES_IMAGE` | `postgres:16-alpine` |
| `BESPIN_SERVICES_RDS_DEFAULT_MYSQL_IMAGE` | `mysql:8.0` |
| `BESPIN_SERVICES_RDS_DEFAULT_MARIADB_IMAGE` | `mariadb:11` |
| `BESPIN_SERVICES_MSK_DEFAULT_IMAGE` | `redpandadata/redpanda:latest` |
| `BESPIN_SERVICES_OPENSEARCH_DEFAULT_IMAGE` | `opensearchproject/opensearch:2` |
| `BESPIN_SERVICES_NEPTUNE_DEFAULT_IMAGE` | `tinkerpop/gremlin-server:3.7.3` |
| `BESPIN_SERVICES_EKS_DEFAULT_IMAGE` | `rancher/k3s:latest` |
| `BESPIN_SERVICES_ECR_REGISTRY_IMAGE` | `registry:2` |
| `BESPIN_ECR_BASE_URI` | `public.ecr.aws` |

## Persistence and Storage Modes

BesPIN can trade speed for durability depending on the workflow. Configure the default mode with `BESPIN_STORAGE_MODE`, or override storage per service.

| Mode | Behavior | Best for | Durability |
|---|---|---|:---:|
| `memory` | Entirely in RAM. Data is lost when the container stops. | CI and ephemeral tests | None |
| `persistent` | Loaded at startup and flushed to disk immediately on every write operation. | Simple local state preservation with immediate persistence | Medium |
| `hybrid` | In-memory performance with periodic async flushing every 5 seconds. | Local development | Good |
| `wal` | Write-ahead log. Every mutation is logged before responding. | Maximum durability | Highest |

Use `memory` for fast test runs. Use `hybrid` when you want state preserved across container restarts without much overhead.

## Multi-Account Isolation

BesPIN supports per-account resource isolation with no extra setup. If `AWS_ACCESS_KEY_ID` is exactly 12 digits, BesPIN uses it as the account ID. Resources created by one account are invisible to another.

```bash
AWS_ACCESS_KEY_ID=111111111111 aws sqs create-queue --queue-name orders
AWS_ACCESS_KEY_ID=222222222222 aws sqs create-queue --queue-name orders
```

Any other key format, such as `test` or `AKIA...`, causes BesPIN to fall back to `BESPIN_DEFAULT_ACCOUNT_ID`, which defaults to `000000000000`.

## SDK Integration

Point your existing AWS SDK at `http://localhost:4566`.

<details>
<summary><strong>Java, AWS SDK v2</strong></summary>

```java
var client = DynamoDbClient.builder()
    .endpointOverride(URI.create("http://localhost:4566"))
    .region(Region.US_EAST_1)
    .credentialsProvider(StaticCredentialsProvider.create(
        AwsBasicCredentials.create("test", "test")))
    .build();
client.createTable(b -> b
    .tableName("demo-table")
    .billingMode(BillingMode.PAY_PER_REQUEST)
    .attributeDefinitions(
        AttributeDefinition.builder().attributeName("pk").attributeType(ScalarAttributeType.S).build())
    .keySchema(
        KeySchemaElement.builder().attributeName("pk").keyType(KeyType.HASH).build()));
System.out.println(client.listTables().tableNames());
```

</details>

<details>
<summary><strong>Python, boto3</strong></summary>

```python
import boto3
client = boto3.client(
    "ssm",
    endpoint_url="http://localhost:4566",
    region_name="us-east-1",
    aws_access_key_id="test",
    aws_secret_access_key="test",
)
client.put_parameter(
    Name="/demo/app/message",
    Value="hello from bespin",
    Type="String",
    Overwrite=True,
)
response = client.get_parameter(Name="/demo/app/message")
print(response["Parameter"]["Value"])
```

</details>

<details>
<summary><strong>Node.js, AWS SDK v3</strong></summary>

```javascript
import { SQSClient, SendMessageCommand } from "@aws-sdk/client-sqs";
const client = new SQSClient({
  endpoint: "http://localhost:4566",
  region: "us-east-1",
  credentials: { accessKeyId: "test", secretAccessKey: "test" },
});
await client.send(
  new SendMessageCommand({
    QueueUrl: "http://localhost:4566/000000000000/demo-queue",
    MessageBody: "hello from bespin",
  }),
);
```

</details>

<details>
<summary><strong>Go, AWS SDK v2</strong></summary>

```go
package main

import (
    "context"
    "fmt"
    "log"
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/credentials"
    "github.com/aws/aws-sdk-go-v2/service/s3"
)

func main() {
    cfg, err := config.LoadDefaultConfig(context.TODO(),
        config.WithRegion("us-east-1"),
        config.WithCredentialsProvider(
            credentials.NewStaticCredentialsProvider("test", "test", ""),
        ),
        config.WithBaseEndpoint("http://localhost:4566"),
    )
    if err != nil {
        log.Fatal(err)
    }
    client := s3.NewFromConfig(cfg, func(o *s3.Options) {
        o.UsePathStyle = true
    })
    out, err := client.ListBuckets(context.TODO(), nil)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(out.Buckets)
}
```

</details>

<details>
<summary><strong>Rust, AWS SDK</strong></summary>

```rust
use aws_sdk_secretsmanager::config::{Credentials, Region};
use aws_sdk_secretsmanager::Client;

#[tokio::main]
async fn main() -> Result<(), Box<dyn std::error::Error>> {
    let config = aws_config::defaults(aws_config::BehaviorVersion::latest())
        .region(Region::new("us-east-1"))
        .credentials_provider(Credentials::new("test", "test", None, None, "bespin"))
        .endpoint_url("http://localhost:4566")
        .load()
        .await;
    let client = Client::new(&config);
    client
        .create_secret()
        .name("demo/secret")
        .secret_string("hello from bespin")
        .send()
        .await?;
    Ok(())
}
```

</details>

<details>
<summary><strong>Bash, AWS CLI</strong></summary>

```bash
export AWS_ACCESS_KEY_ID=test
export AWS_SECRET_ACCESS_KEY=test
export AWS_DEFAULT_REGION=us-east-1
aws --endpoint-url http://localhost:4566 s3 mb s3://my-bucket
aws --endpoint-url http://localhost:4566 s3 ls
```

</details>

## Testcontainers

BesPIN is wire-compatible with Floci's Testcontainers modules. Use the upstream packages directly — they work without any changes.

| Language | Package | Latest | Registry | Source |
|---|---|---|---|---|
| Java | `io.floci:testcontainers-floci` | `1.4.0` | [Maven Central](https://mvnrepository.com/artifact/io.floci/testcontainers-floci) | [GitHub](https://github.com/floci-io/testcontainers-floci) |
| Node.js | `@floci/testcontainers` | `0.1.0` | [npm](https://www.npmjs.com/package/@floci/testcontainers) | [GitHub](https://github.com/floci-io/testcontainers-floci-node) |
| Python | `testcontainers-floci` | `0.1.1` | [PyPI](https://pypi.org/project/testcontainers-floci/) | [GitHub](https://github.com/floci-io/testcontainers-floci-python) |
| Go | In progress | In progress | N/A | [GitHub](https://github.com/floci-io/testcontainers-floci-go) |

<details>
<summary><strong>Java</strong></summary>

```xml
<dependency>
    <groupId>io.floci</groupId>
    <artifactId>testcontainers-floci</artifactId>
    <version>1.4.0</version>
    <scope>test</scope>
</dependency>
```

```java
@Testcontainers
class S3IntegrationTest {
    @Container
    static FlociContainer bespin = new FlociContainer();

    @Test
    void shouldCreateBucket() {
        S3Client s3 = S3Client.builder()
                .endpointOverride(URI.create(bespin.getEndpoint()))
                .region(Region.of(bespin.getRegion()))
                .credentialsProvider(StaticCredentialsProvider.create(
                        AwsBasicCredentials.create(bespin.getAccessKey(), bespin.getSecretKey())))
                .forcePathStyle(true)
                .build();
        s3.createBucket(b -> b.bucket("my-bucket"));
    }
}
```

</details>

<details>
<summary><strong>Node.js / TypeScript</strong></summary>

```sh
npm install --save-dev @floci/testcontainers
```

```ts
import { FlociContainer } from "@floci/testcontainers";
import { S3Client, CreateBucketCommand } from "@aws-sdk/client-s3";

describe("S3", () => {
  let bespin: FlociContainer;

  beforeAll(async () => {
    bespin = await new FlociContainer().start();
  });

  afterAll(async () => {
    await bespin.stop();
  });

  it("creates a bucket", async () => {
    const s3 = new S3Client({
      endpoint: bespin.getEndpoint(),
      region: bespin.getRegion(),
      credentials: {
        accessKeyId: bespin.getAccessKey(),
        secretAccessKey: bespin.getSecretKey(),
      },
      forcePathStyle: true,
    });
    await s3.send(new CreateBucketCommand({ Bucket: "my-bucket" }));
  });
});
```

</details>

<details>
<summary><strong>Python</strong></summary>

```sh
pip install testcontainers-floci
```

```python
import boto3
from testcontainers_floci import FlociContainer

def test_s3_create_bucket():
    with FlociContainer() as bespin:
        s3 = boto3.client(
            "s3",
            endpoint_url=bespin.get_endpoint(),
            region_name=bespin.get_region(),
            aws_access_key_id=bespin.get_access_key(),
            aws_secret_access_key=bespin.get_secret_key(),
        )
        s3.create_bucket(Bucket="my-bucket")
```

</details>

## Migrating from LocalStack

BesPIN is a drop-in replacement for LocalStack Community. The port, credentials, SDK configuration, and CLI endpoint pattern work the same way.

```yaml
# Before
image: localstack/localstack

# After
image: floci/floci:latest
```

LocalStack environment variables are translated automatically:

| LocalStack | BesPIN equivalent |
|---|---|
| `LOCALSTACK_HOST` | `BESPIN_HOSTNAME` |
| `PERSISTENCE=1` | `BESPIN_STORAGE_MODE=persistent` |
| `LAMBDA_DOCKER_NETWORK` | `BESPIN_SERVICES_LAMBDA_DOCKER_NETWORK` |
| `LAMBDA_REMOVE_CONTAINERS=1` | `BESPIN_SERVICES_LAMBDA_EPHEMERAL=true` |
| `DEBUG=1` | `QUARKUS_LOG_LEVEL=DEBUG` |

Init scripts mounted under `/etc/localstack/init/` run unchanged. The `/_localstack/init` and `/_localstack/health` endpoints are still served.

See the [Floci migration guide](https://floci.io/floci/getting-started/migrate-from-localstack/) for full details.

## Image Tags

BesPIN uses the upstream Floci image directly. Every tag combines a variant and a channel.

| Channel | Standard | Compat with AWS CLI and boto3 |
|---|---|---|
| Release, floating | `latest` | `latest-compat` |
| Release, pinned | `x.y.z` | `x.y.z-compat` |
| Nightly, floating | `nightly` | `nightly-compat` |
| Nightly, dated | `nightly-mmddyyyy` | `nightly-mmddyyyy-compat` |

```yaml
# Recommended
image: floci/floci:latest

# Includes AWS CLI and boto3
image: floci/floci:latest-compat

# Pinned release
image: floci/floci:1.5.11
```

## Configuration

All settings are overridable through environment variables with the `BESPIN_` prefix (mapped to the underlying `FLOCI_` variables).

| Variable | Default | Description |
|---|---|---|
| `BESPIN_PORT` | `4566` | Port exposed by the BesPIN API |
| `BESPIN_DEFAULT_REGION` | `us-east-1` | Default AWS region |
| `BESPIN_DEFAULT_ACCOUNT_ID` | `000000000000` | Default AWS account ID |
| `BESPIN_BASE_URL` | `http://localhost:4566` | Base URL used when BesPIN returns service URLs |
| `BESPIN_HOSTNAME` | Unset | Hostname used in returned URLs when BesPIN runs inside Docker Compose |
| `BESPIN_STORAGE_MODE` | `memory` | Storage mode: `memory`, `persistent`, `hybrid`, or `wal` |
| `BESPIN_STORAGE_PERSISTENT_PATH` | `./data` | Directory used for persisted state |
| `BESPIN_ECR_BASE_URI` | `public.ecr.aws` | ECR base URI used when pulling container images |

### Multi-container Docker Compose

When your application runs in a different container, set `BESPIN_HOSTNAME` to the BesPIN service name so returned URLs resolve correctly.

```yaml
services:
  bespin:
    image: floci/floci:latest
    ports:
      - "4566:4566"
    environment:
      - BESPIN_HOSTNAME=bespin

  my-app:
    environment:
      - AWS_ENDPOINT_URL=http://bespin:4566
    depends_on:
      - bespin
```

## License

MIT. Fork it, embed it, extend it. No "community edition" sunset. No carbonite.

> *Based on [Floci](https://github.com/floci-io/floci) by [@hectorvent](https://github.com/hectorvent) and the floci-io team.*
