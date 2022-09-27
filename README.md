# vfps

A [very fast](#e2e-load-testing) and [resource-efficient](#resource-efficiency) pseudonym service.

Supports horizontal service replication for highly-available deployments.

## Run it

> **Warning**
> Using the provided docker-compose.yaml is not a production-ready deployment but merely
> used to get started and testing quickly.
> It sets strict and low resource limits, uses the `latest` tag, runs database migrations as part
> of the startup, and uses the default password for an included, unoptimized PostgreSQL deployment.

```sh
docker compose -f docker-compose.yaml --profile=test up
```

Visit <http://localhost:8080/> to view the OpenAPI specification of the Vfps API:

![Screenshot of the OpenAPI specification](docs/img/openapi.png)

You can use the JSON-transcoded REST API described via OpenAPI or interact with the service using gRPC.
For example, using [grpcurl](https://github.com/fullstorydev/grpcurl) to create a new namespace:

```sh
grpcurl \
  -plaintext \
  -import-path src/Vfps/ \
  -proto src/Vfps/Protos/vfps/api/v1/namespaces.proto \
  -d '{"name": "test", "pseudonymGenerationMethod": "PSEUDONYM_GENERATION_METHOD_SECURE_RANDOM_BASE64URL_ENCODED", "pseudonymLength": 32}' \
  127.0.0.1:8081 \
  vfps.api.v1.NamespaceService/Create
```

And to create a new pseudonym inside this namespace:

```sh
grpcurl \
  -plaintext \
  -import-path src/Vfps/ \
  -proto src/Vfps/Protos/vfps/api/v1/pseudonyms.proto \
  -d '{"namespace": "test", "originalValue": "to be pseudonymized"}' \
  127.0.0.1:8081 \
  vfps.api.v1.PseudonymService/Create
```

## Production-grade deployment

See <https://github.com/chgl/charts/tree/master/charts/vfps> for a production-grade deployment on Kubernetes via Helm.

## Observability

The service exports metrics in Prometheus format on `/metrics`.
Health-, readiness-, and liveness-probes are exposed at `/healthz`, `/readyz`, and `/livez` respectively.

## Development

### Prerequisites

- .NET 7.0: <https://dotnet.microsoft.com/en-us/download/dotnet>
- Docker CLI 20.10.17: <https://www.docker.com/>
- Docker Compose: <https://docs.docker.com/compose/install/>

### Build & run

Start an empty PostgreSQL database for development (optionally add `-d` to run in the background):

```sh
docker compose -f docker-compose.yaml up
```

Restore dependencies and run in Debug mode:

```sh
dotnet restore
dotnet run -c Debug --project=src/Vfps
```

Open <https://localhost:8080/> to see the OpenAPI UI for the JSON-transcoded gRPC services.
You can also use [grpcurl](https://github.com/fullstorydev/grpcurl) to interact with the API:

> **Info**
> In development mode gRPC reflection is enabled and used by grpcurl by default.

```sh
grpcurl -plaintext \
    -d '{"name": "test", "pseudonymGenerationMethod": "PSEUDONYM_GENERATION_METHOD_SECURE_RANDOM_BASE64URL_ENCODED", "pseudonymLength": 32}' \
    127.0.0.1:8081 \
    vfps.api.v1.NamespaceService/Create

grpcurl -plaintext \
    -d '{"namespace": "test", "originalValue": "a test value"}' \
    127.0.0.1:8081 \
    vfps.api.v1.PseudonymService/Create
```

#### Run unit tests

```sh
dotnet test src/Vfps.Tests \
  --configuration=Release \
  --collect:"XPlat Code Coverage" \
  --results-directory=./coverage \
  -l "console;verbosity=detailed" \
  --settings=src/Vfps.Tests/runsettings.xml
```

#### Generate Code coverage report

If not installed, install the report generation too:

```sh
dotnet tool install -g dotnet-reportgenerator-globaltool
```

```sh
reportgenerator -reports:"./coverage/*/coverage.cobertura.xml" -targetdir:"coveragereport" -reporttypes:Html
# remove the coverage directory so successive runs won't cause issues with their random GUID.
# See <https://github.com/microsoft/vstest/issues/2378>
rm -rf coverage/
```

### Build container images

#### Main VFPS service

```sh
docker build -t ghcr.io/chgl/vfps:latest .
```

#### VFPS database migration container

```sh
docker build -t ghcr.io/chgl/vfps-migrations:latest --target=migrations .
```

## Benchmarks

### Micro benchmarks

The pseudonym generation methods are continuously benchmarked. Results are viewable at <https://chgl.github.io/vfps/dev/bench/>.

### E2E load testing

Create a pseudonym namespace used for benchmarking:

```sh
grpcurl \
  -plaintext \
  -import-path src/Vfps/ \
  -proto src/Vfps/Protos/vfps/api/v1/namespaces.proto \
  -d '{"name": "benchmark", "pseudonymGenerationMethod": "PSEUDONYM_GENERATION_METHOD_SECURE_RANDOM_BASE64URL_ENCODED", "pseudonymLength": 32}' \
  127.0.0.1:8081 \
  vfps.api.v1.NamespaceService/Create
```

Generate 100.000 pseudonyms in the namespace from random original values:

```sh
ghz -n 100000 \
    --insecure \
    --import-paths src/Vfps/ \
    --proto src/Vfps/Protos/vfps/api/v1/pseudonyms.proto \
    --call vfps.api.v1.PseudonymService/Create \
    -d '{"originalValue": "{{randomString 32}}", "namespace": "benchmark"}' \
    127.0.0.1:8081
```

Sample output running on

```console
OS=Windows 11 (10.0.22000.978/21H2)
12th Gen Intel Core i9-12900K, 1 CPU, 24 logical and 16 physical cores
32GiB of DDR4 4800MHz RAM
Samsung SSD 980 Pro 1TiB
PostgreSQL running in WSL2 VM on the same machine.
.NET SDK=7.0.100-rc.1.22431.12
```

```console
Summary:
  Count:        100000
  Total:        16.68 s
  Slowest:      187.81 ms
  Fastest:      2.52 ms
  Average:      8.00 ms
  Requests/sec: 5993.51

Response time histogram:
  2.522   [1]     |
  21.051  [99748] |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  39.580  [201]   |
  58.109  [0]     |
  76.639  [0]     |
  95.168  [0]     |
  113.697 [0]     |
  132.226 [0]     |
  150.755 [0]     |
  169.285 [0]     |
  187.814 [50]    |

Latency distribution:
  10 % in 6.26 ms
  25 % in 6.91 ms
  50 % in 7.72 ms
  75 % in 8.93 ms
  90 % in 9.57 ms
  95 % in 10.01 ms
  99 % in 11.86 ms

Status code distribution:
  [OK]   100000 responses
```

### Resource efficiency

The sample deployment described in [docker-compose.yaml](docker-compose.yaml) sets strict resource
limits for both the CPU (1 CPU) and memory (max 128MiB). Even under these constraints > 1k RPS are
possible, although with significantly decreased P99 latencies:

```console
Summary:
  Count:        100000
  Total:        73.99 s
  Slowest:      268.06 ms
  Fastest:      5.26 ms
  Average:      36.69 ms
  Requests/sec: 1351.51

Response time histogram:
  5.257   [1]     |
  31.537  [57298] |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  57.817  [21327] |∎∎∎∎∎∎∎∎∎∎∎∎∎∎∎
  84.097  [17685] |∎∎∎∎∎∎∎∎∎∎∎∎
  110.377 [3395]  |∎∎
  136.656 [243]   |
  162.936 [0]     |
  189.216 [1]     |
  215.496 [0]     |
  241.776 [0]     |
  268.055 [50]    |

Latency distribution:
  10 % in 14.62 ms
  25 % in 18.47 ms
  50 % in 29.46 ms
  75 % in 47.53 ms
  90 % in 71.96 ms
  95 % in 79.95 ms
  99 % in 97.22 ms

Status code distribution:
  [OK]   100000 responses
```
