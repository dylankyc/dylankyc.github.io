# How to organise application Error in actix-web application

- [Into](#into)
- [Actix-web http layer](#actix-web-http-layer)
- [Solution](#solution)
  - [Solution 1: implement `From` trait manually](#solution-1-implement-from-trait-manually)
  - [Solution 2: use thiserror crate](#solution-2-use-thiserror-crate)

# Into

In this article, we will learn how to organise application Error in actix-web application.

Every backend application has a dao layer(or data access object) and a http layer.

For dao layer, usually we use `lib::Result` as return type. Below is an example of fetching all users when using `clickhosue-rs` crate:

```rust
use clickhouse::error::{Error, Result};

pub async fn get_all_users(client: &Client) -> Result<Vec<User>> {
    let users = client
        .query("SELECT ?fields FROM users")
        .fetch_all::<User>()
        .await?;

    Ok(users)
}
```

Notice, the `Result` is not `std::result::Result` type, it is a type that use `clickhouse::error::Error` as the error type in `Result` generic type.

```rust
use std::{error::Error as StdError, fmt, io, result, str::Utf8Error};

use serde::{de, ser};

/// A result with a specified [`Error`] type.
pub type Result<T, E = Error> = result::Result<T, E>;


/// Represents all possible errors.
#[derive(Debug, thiserror::Error)]
#[non_exhaustive]
#[allow(missing_docs)]
pub enum Error {
    #[error("invalid params: {0}")]
    InvalidParams(#[source] Box<dyn StdError + Send + Sync>),
    #[error("network error: {0}")]
    Network(#[source] Box<dyn StdError + Send + Sync>),
    #[error("compression error: {0}")]
    Compression(#[source] Box<dyn StdError + Send + Sync>),
    #[error("decompression error: {0}")]
    Decompression(#[source] Box<dyn StdError + Send + Sync>),
    #[error("no rows returned by a query that expected to return at least one row")]
    RowNotFound,
    #[error("sequences must have a known size ahead of time")]
    SequenceMustHaveLength,
    #[error("`deserialize_any` is not supported")]
    DeserializeAnyNotSupported,
    #[error("not enough data, probably a row type mismatches a database schema")]
    NotEnoughData,
    #[error("string is not valid utf8")]
    InvalidUtf8Encoding(#[from] Utf8Error),
    #[error("tag for enum is not valid")]
    InvalidTagEncoding(usize),
    #[error("a custom error message from serde: {0}")]
    Custom(String),
    #[error("bad response: {0}")]
    BadResponse(String),
    #[error("timeout expired")]
    TimedOut,

    // Internally handled errors, not part of public API.
    // XXX: move to another error?
    #[error("internal error: too small buffer, need another {0} bytes")]
    #[doc(hidden)]
    TooSmallBuffer(usize),
}
```

# Actix-web http layer

Actix-web framework requires the error type is `actix_web::error::Error` if `actix_web::Result` is used as the return type.

`Result` type in actix-web :

```rust
pub use self::error::Error;
pub use self::internal::*;
pub use self::response_error::ResponseError;
pub(crate) use macros::{downcast_dyn, downcast_get_type_id};

/// A convenience [`Result`](std::result::Result) for Actix Web operations.
///
/// This type alias is generally used to avoid writing out `actix_http::Error` directly.
pub type Result<T, E = Error> = std::result::Result<T, E>;
```

`Error` type in actix-web :

```rust
/// General purpose Actix Web error.
///
/// An Actix Web error is used to carry errors from `std::error` through actix in a convenient way.
/// It can be created through converting errors with `into()`.
///
/// Whenever it is created from an external object a response error is created for it that can be
/// used to create an HTTP response from it this means that if you have access to an actix `Error`
/// you can always get a `ResponseError` reference from it.
pub struct Error {
    cause: Box<dyn ResponseError>,
}
```

Normally, we write actix-web handler and call dao functions like this:

```rust
pub async fn get_all_users(data: web::Data<AppState>) -> actix_web::Result<impl Responder> {
    let db = &data.db;

    // call dao function to fetch data
    let users = users::get_all_users(db).await?;
    Ok(web::Json(users))
}
```

If you write a http handler like this and call function in dao(i.e. `dao::get_all_users`) function, which returns an error from the dao crate, error will happen.

```rust
   --> src/user/http.rs:348:54
    |
348 |     let users = users::get_all(db).await?;
    |                                         ^ the trait `ResponseError` is not implemented for `clickhouse::error::Error`
    |
    = help: the following other types implement trait `ResponseError`:
              AppError
              BlockingError
              Box<(dyn StdError + 'static)>
              HttpError
              Infallible
              InvalidHeaderValue
              JsonPayloadError
              PathError
            and 17 others
    = note: required for `actix_web::Error` to implement `std::convert::From<clickhouse::error::Error>`
    = note: required for `Result<_, actix_web::Error>` to implement `FromResidual<Result<Infallible, clickhouse::error::Error>>`
```

It means that you cannot convert `clickhouse::error::Error` to `actix_web::Error`.

The reason is that the error type in dao function is `clickhouse::error::Error`, not `actix_web::Error`. While in http handler, the error type is `actix_web::Error` . You have to implement `From` trait as rust compiler told you.

# Solution

## Solution 1: implement `From` trait manually

One possible solution is to define application error and implement `From` trait, which will convert `clickhouse::error::Error` to `AppError`:

```rust
// Define your application error in enum
#[derive(Debug)]
pub enum AppError {
    ClickhouseError(ClickhouseError),
    // ... other error variants
}

impl ResponseError for AppError {
    fn error_response(&self) -> HttpResponse {
        HttpResponse::InternalServerError().body(self.to_string())
        // We can also use match to handle specific error define in clickhouse::error::Error
        // match *self {
        //     AppError::ClickhouseError(ref err) => match err {
        //         ClickhouseError::Timeout(err) => HttpResponse::InternalServerError()
        //             .body(format!("Clickhouse server error: {}", err)),
        //         ClickhouseError::Network(err) => {
        //             HttpResponse::BadRequest().body(format!("Clickhouse client error: {}", err))
        //         }
        //         _ => HttpResponse::InternalServerError().body("Unknown error"),
        //     }, // ... handle other error variants
        // }
    }
}

use std::fmt;

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            AppError::ClickhouseError(ref err) => {
                write!(f, "Clickhouse error: {}", err)
            } // ... handle other error variants
        }
    }
}

impl From<ClickhouseError> for AppError {
    fn from(error: ClickhouseError) -> Self {
        AppError::ClickhouseError(error)
    }
}
```

Finally, you need to modify actix-web handler:

```rust
use clickhouse::error::{Error, Result};

// Previous function signature
// pub async fn get_all_users(data: web::Data<AppState>) -> actix_web::Result<impl Responder> {
// Pass AppError to handler's return type
pub async fn get_all_users(data: web::Data<AppState>) -> actix_web::Result<impl Responder, AppError> {
    let db = &data.db;

    // call dao function to fetch data
    let users = users::get_all_users(db).await?;
    Ok(web::Json(users))
}
```

🎉🎉🎉

## Solution 2: use thiserror crate

Another solution is to use thiserror crate.

This crate will automatically implement `From` trait, which will do the conversion.

```rust
use thiserror::Error;
use clickhouse::error::Error as ClickhouseError;
use actix_web::{HttpResponse, ResponseError};

#[derive(Debug, Error)]
pub enum AppError {
    #[error("Clickhouse error: {0}")]
    ClickhouseError(#[from] ClickhouseError),

    // You can add more error variants as needed
    #[error("Database connection error")]
    DatabaseConnectionError,

    #[error("Internal server error: {0}")]
    InternalError(String),
}

impl ResponseError for AppError {
    fn error_response(&self) -> HttpResponse {
        match self {
            AppError::ClickhouseError(_) => {
                HttpResponse::InternalServerError().body(self.to_string())
            }
            AppError::DatabaseConnectionError => {
                HttpResponse::ServiceUnavailable().body(self.to_string())
            }
            AppError::InternalError(_) => {
                HttpResponse::InternalServerError().body(self.to_string())
            }
        }
    }
}
```

Here is the key improvements with `thiserror`:

- The `#[derive(Error)]` automatically implements `std::error::Error`.
- `#[error("...")]` provides a convenient way to implement `Display` trait.
- `#[from]` attribute automatically implements `From` trait for error conversion.
- The code is more concise and readable.
- You can easily add more error variants with custom error messages.

Benefits of this approach:

- Automatic error conversion
- Clear, descriptive error messages
- Easy to extend with new error types
- Consistent error handling across the application

The `ResponseError` implementation allows you to:

- Map different error types to appropriate HTTP status codes
- Provide meaningful error responses
- Easily customize error handling for different error variants

Note: Make sure to import necessary types and traits from the appropriate modules (actix-web, thiserror, etc.).

Below is the example of using thiserror crate:

```rust
// use actix_web::{get, post, web, App, HttpRequest, HttpResponse, HttpServer, Responder};
use actix_web::{ web, App,  HttpServer, Responder};
use clickhouse::Client;
use clickhouse_example::{dao::get_all_users, error::AppError};

// This struct represents state
pub(crate) struct AppState {
    pub app_name: String,
    pub db: Client,
}

// NOTE: This function is not working because of error type mismatch
// async fn get_users(
//     data: web::Data<AppState>,
// ) ->  actix_web::Result<impl Responder> {
//     let db = &data.db;

//     // call dao function to fetch data
//     let users = get_all_users(db).await?;
//     Ok(web::Json(users))
// }

// Handler function
pub(crate) async fn get_users(data: web::Data<AppState>) -> actix_web::Result<impl Responder, AppError> {
    let db = &data.db;

    // call dao function to fetch data
    let users = get_all_users(db).await?;
    Ok(web::Json(users))
}

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let url = "http://localhost:8123";
    let database = "default";
    let user = "test";
    let password  = "secret";
    let client = Client::default()
    .with_url(url)
    .with_user(user)
    .with_password(password)
    .with_database(database);

    HttpServer::new(move || {
        App::new()
        .app_data(web::Data::new(AppState {db:client.clone(), app_name: "My App".into() }))
            .route("/users", web::get().to(get_users))
    })
    .bind(("127.0.0.1", 8080))?
    .run()
    .await
}
```
