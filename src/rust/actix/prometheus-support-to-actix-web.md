# prometheus support for actix-web project

- [Init an empty actix-web project with tokio runtime](#init-an-empty-actix-web-project-with-tokio-runtime)
- [Add prometheus support actix-web project](#add-prometheus-support-actix-web-project)
- [Enable process features](#enable-process-features)
- [How to use](#how-to-use)
- [Run actix-web server](#run-actix-web-server)
- [Request metrics endpoint](#request-metrics-endpoint)

## Init an empty actix-web project with tokio runtime

```bash
# init project
cargo init actix-web-t
# add dependencies
cargo add actix-web
cargo add tokio --features full
```

## Add prometheus support actix-web project

```bash
cargo add actix_web_prometheus
```

`actix-web-prometheus` is a middleware inspired by and forked from actix-web-prom. By default three metrics are tracked (this assumes the namespace `actix_web_prometheus`):

- `actix_web_prometheus_incoming_requests` (labels: endpoint, method, status): the total number of HTTP requests handled by the actix `HttpServer`.
- `actix_web_prometheus_response_code` (labels: endpoint, method, statuscode, type): Response codes of all HTTP requests handled by the actix `HttpServer`.
- `actix_web_prometheus_response_time` (labels: endpoint, method, status): Total the request duration of all HTTP requests handled by the actix `HttpServer`.

## Enable process features

You could also enable `process` features when adding `actix_web_prometheus` crate, which means process metrics will also be collected.

```bash
cargo add actix_web_prometheus --features process
```

Output:

```bash
    Updating crates.io index
warning: translating `actix_web_prometheus` to `actix-web-prometheus`
      Adding actix-web-prometheus v0.1.2 to dependencies.
             Features:
             + process
```

## How to use

Here is an simple example of how to integrate this middleware into `actix-web` project.

`main.rs`

```rust
use actix_web::{http, web, App, HttpServer, Responder, Result, HttpResponse};
use actix_web_prometheus::PrometheusMetricsBuilder;
use serde::{Deserialize, Serialize};


#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let prometheus = PrometheusMetricsBuilder::new("api")
        .endpoint("/metrics")
        .build()
        .unwrap();

    HttpServer::new(move || {
        App::new()
            .wrap(prometheus.clone())
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```

## Run actix-web server

```bash
cargo run
```

Output:

```bash
warning: unused imports: `HttpResponse`, `Responder`, `Result`, `http`, `web`
 --> src/main.rs:1:17
  |
1 | use actix_web::{http, web, App, HttpServer, Responder, Result, HttpResponse};
  |                 ^^^^  ^^^                   ^^^^^^^^^  ^^^^^^  ^^^^^^^^^^^^
  |
  = note: `#[warn(unused_imports)]` on by default

warning: unused imports: `Deserialize`, `Serialize`
 --> src/main.rs:3:13
  |
3 | use serde::{Deserialize, Serialize};
  |             ^^^^^^^^^^^  ^^^^^^^^^

warning: `actix-web-t` (bin "actix-web-t") generated 2 warnings
    Finished dev [unoptimized + debuginfo] target(s) in 0.26s
     Running `target/debug/actix-web-t`
```

## Request metrics endpoint

Build and run actix-web project, we can send request to `/metrics` endpoint.

```bash
curl 0:8080/metrics
```

Ouput:

```bash
# HELP api_incoming_requests Incoming Requests
# TYPE api_incoming_requests counter
api_incoming_requests{endpoint="/metrics",method="GET",status="200"} 28
# HELP api_response_code Response Codes
# TYPE api_response_code counter
api_response_code{endpoint="/metrics",method="GET",statuscode="200",type="200"} 28
# HELP api_response_time Response Times
# TYPE api_response_time histogram
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="0.005"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="0.01"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="0.025"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="0.05"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="0.1"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="0.25"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="0.5"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="1"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="2.5"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="5"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="10"} 28
api_response_time_bucket{endpoint="/metrics",method="GET",status="200",le="+Inf"} 28
api_response_time_sum{endpoint="/metrics",method="GET",status="200"} 0.03155173499999999
api_response_time_count{endpoint="/metrics",method="GET",status="200"} 28
# HELP process_cpu_seconds_total Total user and system CPU time spent in seconds.
# TYPE process_cpu_seconds_total counter
process_cpu_seconds_total 66
# HELP process_max_fds Maximum number of open file descriptors.
# TYPE process_max_fds gauge
process_max_fds 50000
# HELP process_open_fds Number of open file descriptors.
# TYPE process_open_fds gauge
process_open_fds 23
# HELP process_resident_memory_bytes Resident memory size in bytes.
# TYPE process_resident_memory_bytes gauge
process_resident_memory_bytes 6410240
# HELP process_start_time_seconds Start time of the process since unix epoch in seconds.
# TYPE process_start_time_seconds gauge
process_start_time_seconds 1677656707
# HELP process_threads Number of OS threads in the process.
# TYPE process_threads gauge
process_threads 3
# HELP process_virtual_memory_bytes Virtual memory size in bytes.
# TYPE process_virtual_memory_bytes gauge
process_virtual_memory_bytes 165015552
```

Notice, on MacOS, process metrics are not exported.
