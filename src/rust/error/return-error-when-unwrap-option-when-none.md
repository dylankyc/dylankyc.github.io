# Return error when unwrap Option when None

- [Intro](#intro)
- [Application error](#application-error)
- [Function returns Result](#function-returns-result)
- [Solution](#solution)
  - [Use `match`](#use-match)
  - [Use `ok_or_else`](#use-ok_or_else)

# Intro

In this blog, we will learn how to handle optional values and return errors in Actix-Web Handlers.

In Rust, dealing with optional values (`Option`) and converting them to errors in web handlers is a common task. This blog explores different strategies to handle cases where an expected value is `None`.

When working with optional values, you have several idiomatic Rust approaches:

- Using `match` to Convert `None` to an Error

  - Explicitly match on the `Option` type
  - Explicitly return an error when the value is `None`

- Using `ok_or_else()` Method
  - Provides a concise way to convert `Option` to `Result`
  - Allows lazy error generation
  - Avoids unnecessary error creation if not needed

Let's explore these approaches with practical examples.

# Application error

Suppose we define our application error like this:

```rust
use actix_web::{HttpResponse, ResponseError};
use clickhouse::error::Error as ClickhouseError;
use std::fmt;

#[derive(Debug)]
pub enum AppError {
    ClickhouseError(ClickhouseError),
    ScheduleError(String),
    SQLGenError(String),
}

impl ResponseError for AppError {
    fn error_response(&self) -> HttpResponse {
        HttpResponse::InternalServerError().body(self.to_string())
        // match *self {
        //     AppError::ClickhouseError(ref err) => match err {
        //         ClickhouseError::Server(err) => HttpResponse::InternalServerError()
        //             .body(format!("Clickhouse server error: {}", err)),
        //         ClickhouseError::Client(err) => {
        //             HttpResponse::BadRequest().body(format!("Clickhouse client error: {}", err))
        //         }
        //         _ => HttpResponse::InternalServerError().body("Unknown error"),
        //     }, // ... handle other error variants
        // }
    }
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            AppError::ClickhouseError(ref err) => {
                write!(f, "Clickhouse error: {}", err)
            }
            AppError::ScheduleError(ref err) => {
                write!(f, "Schedule error: {}", err)
            }
            AppError::SQLGenError(ref err) => {
                write!(f, "SQLGen error: {}", err)
            }
        }
    }
}

impl From<ClickhouseError> for AppError {
    fn from(error: ClickhouseError) -> Self {
        AppError::ClickhouseError(error)
    }
}
```

# Function returns Result<String, AppError>

If we have a function which will return `Result<String, AppError>` type:

```rust
fn sql_gen_visualization_barchart(
    query: &VisualizationQuery,
    table_name: &str,
    //) -> Result<String, Box<dyn std::error::Error>> {
    //) -> anyhow::Result<String> {
) -> anyhow::Result<String, AppError> {
    // gen sql for visualization
    let mut sql = String::new();
    let field = query.field.as_ref();
    // ...
}
```

# Solution

If we would like to return error when one of parameters is empty, we can do as follows.

## Use `match`

```rust
let field = match field {
    Some(field) => field,
    None => return Err(AppError::SQLGenError("Field is empty".to_string())),
};
```

By using `match`, we can easily return an error when `field` is None.

## Use `ok_or_else`

If we don't like the `match`, we can leverage `ok_or_else` method, which will do the same way as using `match`.

```rust
let field = query
    .field
    .as_ref()
    .ok_or_else(|| AppError::SQLGenError("Field is empty".to_string()))?;
```

Below is the source code of `ok_or_else` method:

````rust
impl<T> Option<T> {
    /// Transforms the `Option<T>` into a [`Result<T, E>`], mapping [`Some(v)`] to
    /// [`Ok(v)`] and [`None`] to [`Err(err())`].
    ///
    /// [`Ok(v)`]: Ok
    /// [`Err(err())`]: Err
    /// [`Some(v)`]: Some
    ///
    /// # Examples
    ///
    /// ```
    /// let x = Some("foo");
    /// assert_eq!(x.ok_or_else(|| 0), Ok("foo"));
    ///
    /// let x: Option<&str> = None;
    /// assert_eq!(x.ok_or_else(|| 0), Err(0));
    /// ```
    #[inline]
    #[stable(feature = "rust1", since = "1.0.0")]
    pub fn ok_or_else<E, F>(self, err: F) -> Result<T, E>
    where
        F: FnOnce() -> E,
    {
        match self {
            Some(v) => Ok(v),
            None => Err(err()),
        }
    }
}
````
