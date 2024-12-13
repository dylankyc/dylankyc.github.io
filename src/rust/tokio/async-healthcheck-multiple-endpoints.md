# Async Healthcheck Multiple Endpoints

- [Intro](#intro)
- [Code](#code)
- [Code explain](#code-explain)

## Intro

Today, I'll show you how to use tokio to do healthcheck for multiple endpoints.

The architecture is simple:

- Initialized vector of healthcheck endpoints
- Spawn futures to do health check

## Code

```rust
// use tokio::time::sleep;
use serde::{Deserialize, Serialize};
use std::fs::File;
use std::io::BufReader;
use std::time::Duration;
use tokio::select;
use tokio::sync::mpsc;
use tokio::time;
use tokio::time::{interval, sleep};

#[derive(Debug, Deserialize, Serialize, Clone)]
struct Config {
    interval: u64,
    url: String,
}

async fn check_url(config: Config) {
    loop {
        println!("In check_url loop");
        let url = &config.url;
        match reqwest::get(url).await {
            Err(e) => println!("Error: Failed to access {}: {}", config.url, e),
            Ok(response) => {
                // println!("{response:?}");
                if !response.status().is_success() {
                    println!(
                        "Error: {} returned status code {}",
                        config.url,
                        response.status()
                    );
                }

                println!("check for {url} OK");
            }
        }
        sleep(Duration::from_secs(config.interval)).await;
    }
}

#[tokio::main]
async fn main() {
    // Load configuration from file
    // let file = File::open("config.json").expect("Failed to open config file");
    // let reader = BufReader::new(file);
    // let configs: Vec<Config> =
    //     serde_json::from_reader(reader).expect("Failed to parse config file");

    let configs = vec![
        Config { interval: 10, url: "http://www.baidu.com".to_string() },
        Config { interval: 10, url: "http://www.qq.com".to_string() },
    ];

    // Create a shared timer
    // let mut ticker = interval(Duration::from_secs(1));

    // let mut interval =
    //     time::interval(time::Duration::from_millis(consume_interval));

    // Create a task for each URL and spawn it
    //
    // NOTE: we don't need to run in loop in spawn, check_url already has loop
    // for config in configs {
    //     // let mut tick = ticker.tick();
    //     tokio::spawn(async move {
    //         let mut ticker = interval(Duration::from_secs(1));
    //         loop {
    //             select! {
    //                 // _ = tick => {
    //                 _ = ticker.tick() => {
    //                     println!("1s ...");
    //                     check_url(config.clone()).await;
    //                 }
    //             }
    //         }
    //     });
    // }

    for config in configs {
        tokio::spawn(async move {
            println!("spawn check future ...");
            check_url(config.clone()).await;
        });
    }

    println!("Infinite loop");
    // Keep the program running so that other tasks can continue to run
    time::sleep(Duration::from_secs(2000)).await;
    // loop {}
}
```

## Code explain

1. Load configuration from file or hard code the configuration

We can hard code the configuration or load configuration from file.

```rust
// let file = File::open("config.json").expect("Failed to open config file");
// let reader = BufReader::new(file);
// let configs: Vec<Config> =
//     serde_json::from_reader(reader).expect("Failed to parse config file");

let configs = vec![
    Config { interval: 10, url: "http://www.baidu.com".to_string() },
    Config { interval: 10, url: "http://www.qq.com".to_string() },
];
```

2. Create a task for each URL and spawn it

```rust
for config in configs {
    tokio::spawn(async move {
        println!("spawn check future ...");
        check_url(config.clone()).await;
    });
}
```

3. Keep the program running so that other tasks can continue to run

```rust
time::sleep(Duration::from_secs(2000)).await;
// loop {}  // Keep the program running so that other tasks can continue to run
```
