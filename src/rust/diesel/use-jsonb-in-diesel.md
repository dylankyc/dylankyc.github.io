# Diesel JSONB Example

- [Into](#into)
- [Install dependencies](#install-dependencies)
- [Write sql for migration](#write-sql-for-migration)
- [run migration](#run-migration)
- [Create database models](#create-database-models)
- [Add more dependencies](#add-more-dependencies)
- [Modify main.rs](#modify-mainrs)
- [Build](#build)
  - [Build and fail](#build-and-fail)
  - [Fix: import `orders` from `schema` module](#fix-import-orders-from-schema-module)
  - [Fix: change `total_amount` type from `f64` to `BigDecimal`](#fix-change-total_amount-type-from-f64-to-bigdecimal)
  - [Fix: Change `metadata` type from `Value` to `Option`](#fix-change-metadata-type-from-value-to-option)
- [Create orders](#create-orders)
- [Query orders](#query-orders)
- [Update order](#update-order)
- [Delete order](#delete-order)
- [Summary](#summary)
- [Refs](#refs)

# Into

This is a simple example of using Diesel with JSONB in a Rust project.

# Install dependencies

First, let's create a new project and add dependencies:

```bash
# Init project
cargo init order-diesel-jsonb-example

# Add dependencies
cd order-diesel-jsonb-example
cargo add diesel -F postgres
cargo add dotenvy

# Install diesel cli
cargo install diesel_cli

# Tell diesel where to find the database
# echo DATABASE_URL=postgres://username:password@localhost/diesel_demo > .env
echo DATABASE_URL=postgres://localhost/diesel_demo > .env

# Create postgres database
createdb diesel_demo
psql diesel_demo

# setup diesel and run migrations
diesel setup
diesel migration generate create_orders

# Output:
# Creating migrations/2024-12-16-120623_create_orders/up.sql
# Creating migrations/2024-12-16-120623_create_orders/down.sql
```

# Write sql for migration

As diesel documents say:

> Migrations allow us to evolve the database schema over time. Each migration consists of an up.sql file to apply the changes and a down.sql file to revert them.

diesel will create `migrations` directory after running `diesel migration generate create_orders`. It will create `up.sql` and `down.sql` files.

Let's write some sql in `migrations/2024-12-16-120623_create_orders/up.sql`:

```sql
-- Your SQL goes here
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL,
  total_amount DECIMAL(10, 2) NOT NULL,
  order_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  metadata JSONB
);

CREATE INDEX idx_order_metadata ON orders USING gin (metadata);
```

And `migrations/2024-12-16-120623_create_orders/down.sql`:

```sql
DROP TABLE orders;
```

# run migration

Run `diesel migration run` will create table in postgres:

```
dylan@/tmp:diesel_demo> \d orders;
+--------------+-----------------------------+------------------------------------------------------+
| Column       | Type                        | Modifiers                                            |
|--------------+-----------------------------+------------------------------------------------------|
| id           | integer                     |  not null default nextval('orders_id_seq'::regclass) |
| user_id      | integer                     |  not null                                            |
| total_amount | numeric(10,2)               |  not null                                            |
| order_date   | timestamp without time zone |  not null default CURRENT_TIMESTAMP                  |
| metadata     | jsonb                       |                                                      |
+--------------+-----------------------------+------------------------------------------------------+
Indexes:
    "orders_pkey" PRIMARY KEY, btree (id)
    "idx_order_metadata" gin (metadata)

Time: 0.034s
```

Also, `diesel migration generate create_orders` will generate the following code in `src/schema.rs`:

```
// @generated automatically by Diesel CLI.

diesel::table! {
    orders (id) {
        id -> Int4,
        user_id -> Int4,
        total_amount -> Numeric,
        order_date -> Timestamp,
        metadata -> Nullable<Jsonb>,
    }
}
```

Schema defines a Rust module representing the table structure.

The key components for the schema are:

- Table Structure: Each table is represented by a struct, typically referenced as `users::table`.
- Column Definitions: Each column is represented by a struct implementing the `Expression` trait to specify SQL types.
- DSL Module: Provides a convenient syntax for queries, making them less verbose than writing SQL directly.

By leveraging Diesel's powerful features, you can easily interact with the database, perform complex queries, and manage your data efficiently.

```rust
use diesel::prelude::*;
use dotenvy::dotenv;
use diesel::pg::PgConnection;
use diesel::r2d2::ConnectionManager;
use diesel::r2d2::Pool;

use crate::schema::orders;

pub type PgPool = Pool<ConnectionManager<PgConnection>>;

pub fn establish_connection() -> PgPool {
    dotenv().ok();

    let database_url = std::env::var("DATABASE_URL").expect("DATABASE_URL must be set");
    let manager = ConnectionManager::<PgConnection>::new(database_url);
    Pool::builder()
        .build(manager)
        .expect("Failed to create pool.")
}

pub fn create_order(conn: &PgConnection, new_order: NewOrder) -> Result<Order, diesel::result::Error> {
    use crate::schema::orders;

    diesel::insert_into(orders::table)
        .values(&new_order)
        .get_result(conn)
}
```

# Create database models

Now, let's create database models in `src/models.rs`.

```rust
use diesel::prelude::*;
use serde::{Deserialize, Serialize};
use chrono::NaiveDateTime;
use serde_json::Value;

#[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
#[diesel(table_name = orders)]
#[diesel(check_for_backend(diesel::pg::Pg))]
pub struct Order {
    pub id: i32,
    pub user_id: i32,
    pub total_amount: f64,
    pub order_date: NaiveDateTime,
    pub metadata: Value,  // This will use JSONB
}

#[derive(Insertable, Deserialize)]
#[diesel(table_name = orders)]
pub struct NewOrder {
    pub user_id: i32,
    pub total_amount: f64,
    pub metadata: Value,
}
```

Here are some notes about the code:

- `#[derive(Queryable)]` will generate all of the code needed to load a `Order` struct from a SQL query.

- `#[derive(Selectable)]` will generate code to construct a matching select clause based on your model type based on the table defined via `#[diesel(table_name = orders)]`.

- `#[diesel(check_for_backend(diesel::pg::Pg))` (or `sqlite::SQLite` or `mysql::MySQL`) adds additional compile time checks to verify that all field types in your struct are compatible with their corresponding SQL side expressions. This part is optional, but it greatly improves the generated compiler error messages.

If any types are not compatible with column type in database, Diesel will emit a compile-time error.

# Add more dependencies

```bash
cargo add anyhow serde_json
cargo add chrono -F serde
cargo add serde -F derive
```

Here is the output:

```bash
> cargo add anyhow serde_json
    Updating crates.io index
      Adding anyhow v1.0.94 to dependencies
             Features:
             + std
             - backtrace
      Adding serde_json v1.0.133 to dependencies
             Features:
             + std
             - alloc
             - arbitrary_precision
             - float_roundtrip
             - indexmap
             - preserve_order
             - raw_value
             - unbounded_depth
    Updating crates.io index
    Blocking waiting for file lock on package cache
     Locking 6 packages to latest compatible versions
      Adding anyhow v1.0.94
      Adding memchr v2.7.4
      Adding ryu v1.0.18
      Adding serde v1.0.216
      Adding serde_derive v1.0.216
      Adding serde_json v1.0.133
> cargo add chrono -F serde
    Updating crates.io index
      Adding chrono v0.4.39 to dependencies
             Features:
             + alloc
             + android-tzdata
             + clock
             + iana-time-zone
             + js-sys
             + now
             + oldtime
             + serde
             + std
             + wasm-bindgen
             + wasmbind
             + winapi
             + windows-targets
             - __internal_bench
             - arbitrary
             - libc
             - pure-rust-locales
             - rkyv
             - rkyv-16
             - rkyv-32
             - rkyv-64
             - rkyv-validation
             - unstable-locales
    Updating crates.io index
    Blocking waiting for file lock on package cache
     Locking 31 packages to latest compatible versions
      Adding android-tzdata v0.1.1
      Adding android_system_properties v0.1.5
      Adding autocfg v1.4.0
      Adding bumpalo v3.16.0
      Adding cc v1.2.4
      Adding cfg-if v1.0.0
      Adding chrono v0.4.39
      Adding core-foundation-sys v0.8.7
      Adding iana-time-zone v0.1.61
      Adding iana-time-zone-haiku v0.1.2
      Adding js-sys v0.3.76
      Adding libc v0.2.168
      Adding log v0.4.22
      Adding num-traits v0.2.19
      Adding once_cell v1.20.2
      Adding shlex v1.3.0
      Adding wasm-bindgen v0.2.99
      Adding wasm-bindgen-backend v0.2.99
      Adding wasm-bindgen-macro v0.2.99
      Adding wasm-bindgen-macro-support v0.2.99
      Adding wasm-bindgen-shared v0.2.99
      Adding windows-core v0.52.0
      Adding windows-targets v0.52.6
      Adding windows_aarch64_gnullvm v0.52.6
      Adding windows_aarch64_msvc v0.52.6
      Adding windows_i686_gnu v0.52.6
      Adding windows_i686_gnullvm v0.52.6
      Adding windows_i686_msvc v0.52.6
      Adding windows_x86_64_gnu v0.52.6
      Adding windows_x86_64_gnullvm v0.52.6
      Adding windows_x86_64_msvc v0.52.6
> cargo add serde -F derive
    Updating crates.io index
      Adding serde v1.0.216 to dependencies
             Features:
             + derive
             + serde_derive
             + std
             - alloc
             - rc
             - unstable
    Blocking waiting for file lock on package cache
    Blocking waiting for file lock on package cache
```

# Modify main.rs

Now, let's modify `main.rs` to create a new order and print the result:

```rust
mod models;
mod schema;
mod db;

use diesel::pg::PgConnection;
use diesel::Connection;
use dotenvy::dotenv;
use std::env;

fn establish_connection() -> PgConnection {
    dotenv().ok();
    let database_url = env::var("DATABASE_URL")
        .expect("DATABASE_URL must be set");
    PgConnection::establish(&database_url)
        .expect(&format!("Error connecting to {}", database_url))
}

fn main() {
    let conn = &mut establish_connection();

    // Example usage
    let new_order = models::NewOrder {
        user_id: 1,
        total_amount: 99.99,
        metadata: serde_json::json!({
            "items": ["book", "pen"],
            "shipping_method": "express",
            "gift_wrap": true
        }),
    };

    match db::create_order(conn, new_order) {
        Ok(order) => println!("Created order: {:?}", order),
        Err(e) => eprintln!("Error creating order: {}", e),
    }
}
```

# Build

## Build and fail

Now, let's build the project and see the error:

```bash
> cargo build
   Compiling core-foundation-sys v0.8.7
   Compiling itoa v1.0.14
   Compiling memchr v2.7.4
   Compiling pq-sys v0.6.3
   Compiling num-traits v0.2.19
   Compiling byteorder v1.5.0
   Compiling bitflags v2.6.0
   Compiling serde v1.0.216
   Compiling ryu v1.0.18
   Compiling anyhow v1.0.94
   Compiling dotenvy v0.15.7
   Compiling iana-time-zone v0.1.61
   Compiling diesel v2.2.6
   Compiling serde_json v1.0.133
   Compiling chrono v0.4.39
   Compiling order-diesel-jsonb-example v0.1.0 (
error[E0433]: failed to resolve: use of undeclared crate or module `orders`
 --> src/models.rs:7:23
  |
7 | #[diesel(table_name = orders)]
  |                       ^^^^^^ use of undeclared crate or module `orders`
  |
help: a struct with a similar name exists
  |
7 | #[diesel(table_name = Order)]
  |                       ~~~~~
help: consider importing this struct through its public re-export
  |
1 + use crate::schema::orders::dsl::orders;
  |

error[E0433]: failed to resolve: use of undeclared crate or module `orders`
  --> src/models.rs:18:23
   |
18 | #[diesel(table_name = orders)]
   |                       ^^^^^^ use of undeclared crate or module `orders`
   |
help: a struct with a similar name exists
   |
18 | #[diesel(table_name = Order)]
   |                       ~~~~~
help: consider importing this struct through its public re-export
   |
1  + use crate::schema::orders::dsl::orders;
   |

warning: unused import: `serde_json::json`
 --> src/db.rs:3:5
  |
3 | use serde_json::json;
  |     ^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_imports)]` on by default

error[E0277]: the trait bound `&NewOrder: diesel::Insertable<table>` is not satisfied
   --> src/db.rs:10:17
    |
10  |         .values(&new_order)
    |          ------ ^^^^^^^^^^ the trait `diesel::Insertable<table>` is not implemented for `&NewOrder`
    |          |
    |          required by a bound introduced by this call
    |
note: required by a bound in `IncompleteInsertStatement::<T, Op>::values`
   --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_builder/insert_statement/mod.rs:115:12
    |
113 |     pub fn values<U>(self, records: U) -> InsertStatement<T, U::Values, Op>
    |            ------ required by a bound in this associated function
114 |     where
115 |         U: Insertable<T>,
    |            ^^^^^^^^^^^^^ required by this bound in `IncompleteInsertStatement::<T, Op>::values`

error[E0277]: the trait bound `(i32, i32, f64, NaiveDateTime, Value): FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not satisfied
    --> src/db.rs:11:21
     |
11   |         .get_result(conn)
     |          ---------- ^^^^ the trait `FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not implemented for `(i32, i32, f64, NaiveDateTime, Value)`, which is required by `InsertStatement<table, _>: LoadQuery<'_, _, _>`
     |          |
     |          required by a bound introduced by this call
     |
     = help: the following other types implement trait `FromStaticSqlRow<ST, DB>`:
               `(T0,)` implements `FromStaticSqlRow<(ST0,), __DB>`
               `(T1, T0)` implements `FromStaticSqlRow<(ST1, ST0), __DB>`
               `(T1, T2, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST0), __DB>`
               `(T1, T2, T3, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST0), __DB>`
               `(T1, T2, T3, T4, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T7, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST7, ST0), __DB>`
             and 24 others
note: required for `models::Order` to implement `diesel::Queryable<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
    --> src/models.rs:6:10
     |
6    | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
     |          ^^^^^^^^^ unsatisfied trait bound introduced in this `derive` macro
...
9    | pub struct Order {
     |            ^^^^^
     = note: required for `models::Order` to implement `FromSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
     = note: required for `(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>)` to implement `load_dsl::private::CompatibleType<models::Order, Pg>`
     = note: required for `InsertStatement<table, _>` to implement `LoadQuery<'_, diesel::PgConnection, models::Order>`
note: required by a bound in `get_result`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1722:15
     |
1720 |     fn get_result<'query, U>(self, conn: &mut Conn) -> QueryResult<U>
     |        ---------- required by a bound in this associated function
1721 |     where
1722 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::get_result`
     = note: this error originates in the derive macro `Queryable` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `(i32, i32, f64, NaiveDateTime, Value): FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not satisfied
    --> src/db.rs:17:16
     |
17   |         .first(conn)
     |          ----- ^^^^ the trait `FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not implemented for `(i32, i32, f64, NaiveDateTime, Value)`, which is required by `SelectStatement<FromClause<table>, query_builder::select_clause::DefaultSelectClause<FromClause<table>>, query_builder::distinct_clause::NoDistinctClause, query_builder::where_clause::WhereClause<diesel::expression::grouped::Grouped<diesel::expression::operators::Eq<columns::id, diesel::expression::bound::Bound<Integer, i32>>>>, query_builder::order_clause::NoOrderClause, LimitOffsetClause<LimitClause<diesel::expression::bound::Bound<BigInt, i64>>, NoOffsetClause>>: LoadQuery<'_, _, _>`
     |          |
     |          required by a bound introduced by this call
     |
     = help: the following other types implement trait `FromStaticSqlRow<ST, DB>`:
               `(T0,)` implements `FromStaticSqlRow<(ST0,), __DB>`
               `(T1, T0)` implements `FromStaticSqlRow<(ST1, ST0), __DB>`
               `(T1, T2, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST0), __DB>`
               `(T1, T2, T3, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST0), __DB>`
               `(T1, T2, T3, T4, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T7, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST7, ST0), __DB>`
             and 24 others
note: required for `models::Order` to implement `diesel::Queryable<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
    --> src/models.rs:6:10
     |
6    | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
     |          ^^^^^^^^^ unsatisfied trait bound introduced in this `derive` macro
...
9    | pub struct Order {
     |            ^^^^^
     = note: required for `models::Order` to implement `FromSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
     = note: required for `(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>)` to implement `load_dsl::private::CompatibleType<models::Order, Pg>`
     = note: required for `SelectStatement<FromClause<table>, DefaultSelectClause<FromClause<table>>, NoDistinctClause, ..., ..., ...>` to implement `LoadQuery<'_, diesel::PgConnection, models::Order>`
note: required by a bound in `first`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1779:22
     |
1776 |     fn first<'query, U>(self, conn: &mut Conn) -> QueryResult<U>
     |        ----- required by a bound in this associated function
...
1779 |         Limit<Self>: LoadQuery<'query, Conn, U>,
     |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::first`
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-0af85acd9e7b4e2f.long-type-11474480310476070344.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: this error originates in the derive macro `Queryable` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `(i32, i32, f64, NaiveDateTime, Value): FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not satisfied
    --> src/db.rs:24:15
     |
24   |         .load(conn)
     |          ---- ^^^^ the trait `FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not implemented for `(i32, i32, f64, NaiveDateTime, Value)`, which is required by `SelectStatement<FromClause<table>, query_builder::select_clause::DefaultSelectClause<FromClause<table>>, query_builder::distinct_clause::NoDistinctClause, query_builder::where_clause::WhereClause<diesel::expression::grouped::Grouped<diesel::expression::operators::Eq<columns::user_id, diesel::expression::bound::Bound<Integer, i32>>>>>: LoadQuery<'_, _, _>`
     |          |
     |          required by a bound introduced by this call
     |
     = help: the following other types implement trait `FromStaticSqlRow<ST, DB>`:
               `(T0,)` implements `FromStaticSqlRow<(ST0,), __DB>`
               `(T1, T0)` implements `FromStaticSqlRow<(ST1, ST0), __DB>`
               `(T1, T2, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST0), __DB>`
               `(T1, T2, T3, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST0), __DB>`
               `(T1, T2, T3, T4, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T7, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST7, ST0), __DB>`
             and 24 others
note: required for `models::Order` to implement `diesel::Queryable<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
    --> src/models.rs:6:10
     |
6    | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
     |          ^^^^^^^^^ unsatisfied trait bound introduced in this `derive` macro
...
9    | pub struct Order {
     |            ^^^^^
     = note: required for `models::Order` to implement `FromSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
     = note: required for `(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>)` to implement `load_dsl::private::CompatibleType<models::Order, Pg>`
     = note: required for `SelectStatement<FromClause<table>, DefaultSelectClause<FromClause<table>>, NoDistinctClause, WhereClause<...>>` to implement `LoadQuery<'_, diesel::PgConnection, models::Order>`
note: required by a bound in `diesel::RunQueryDsl::load`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1542:15
     |
1540 |     fn load<'query, U>(self, conn: &mut Conn) -> QueryResult<Vec<U>>
     |        ---- required by a bound in this associated function
1541 |     where
1542 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::load`
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-0af85acd9e7b4e2f.long-type-14102635225686123243.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: this error originates in the derive macro `Queryable` (in Nightly builds, run with -Z macro-backtrace for more info)

Some errors have detailed explanations: E0277, E0433.
For more information about an error, try `rustc --explain E0277`.
warning: `order-diesel-jsonb-example` (bin "order-diesel-jsonb-example") generated 1 warning
error: could not compile `order-diesel-jsonb-example` (bin "order-diesel-jsonb-example") due to 6 previous errors; 1 warning emitted
```

## Fix: import `orders` from `schema` module

Although there are a lot of errors, the rust compiler is able to find the error and give us some hints.

```bash
error[E0433]: failed to resolve: use of undeclared crate or module `orders`
 --> src/models.rs:7:23
  |
7 | #[diesel(table_name = orders)]
  |                       ^^^^^^ use of undeclared crate or module `orders`
  |
help: a struct with a similar name exists
  |
7 | #[diesel(table_name = Order)]
  |                       ~~~~~
help: consider importing this struct through its public re-export
  |
1 + use crate::schema::orders::dsl::orders;
```

Howerver, the hint is not correcty, We should import `orders` from `schema` module(`use crate::schema::orders`), not `use crate::schema::orders::dsl::orders`.

Let's import `orders` from `schema` module:

```rust
use crate::schema::orders;
```

Or follow the IDE:

![Import orders](https://github.com/dylankyc/dylankyc.github.io/blob/main/.img/import-orders.png?raw=true)

![Import orders](https://github.com/dylankyc/dylankyc.github.io/blob/main/.img/import-orders-from-schema.png?raw=true)

Also, we should add `chrono` and `serde_json` features for `diesel` dependency:

```bash
cargo add diesel -F chrono,serde_json
    Updating crates.io index
      Adding diesel v2.2.6 to dependencies
             Features:
             + 32-column-tables
             + chrono
             + postgres
             + postgres_backend
             + serde_json
             + with-deprecated
             - 128-column-tables
             - 64-column-tables
             - __with_asan_tests
             - extras
             - huge-tables
             - i-implement-a-third-party-backend-and-opt-into-breaking-changes
             - ipnet-address
             - large-tables
             - mysql
             - mysql_backend
             - mysqlclient-src
             - network-address
             - numeric
             - pq-src
             - quickcheck
             - r2d2
             - returning_clauses_for_sqlite_3_35
             - sqlite
             - time
             - unstable
             - uuid
             - without-deprecated
```

Let's build it again.

```bash
> cargo build
   Compiling diesel v2.2.6
   Compiling order-diesel-jsonb-example v0.1.0
warning: unused import: `serde_json::json`
 --> src/db.rs:3:5
  |
3 | use serde_json::json;
  |     ^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_imports)]` on by default

error[E0277]: the trait bound `Value: FromSqlRow<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>` is not satisfied
  --> src/models.rs:16:19
   |
16 |     pub metadata: Value,  // This will use JSONB
   |                   ^^^^^ the trait `FromSql<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>` is not implemented for `Value`, which is required by `Value: FromSqlRow<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>`
   |
   = note: double check your type mappings via the documentation of `diesel::sql_types::Nullable<diesel::sql_types::Jsonb>`
   = note: `diesel::sql_query` requires the loading target to column names for loading values.
           You need to provide a type that explicitly derives `diesel::deserialize::QueryableByName`
   = help: the following other types implement trait `FromSql<A, DB>`:
             `Value` implements `FromSql<Json, Pg>`
             `Value` implements `FromSql<diesel::sql_types::Jsonb, Pg>`
   = note: required for `Value` to implement `diesel::Queryable<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>`
   = note: required for `Value` to implement `FromSqlRow<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>`
   = help: see issue #48214

error[E0277]: the trait bound `f64: FromSqlRow<diesel::sql_types::Numeric, Pg>` is not satisfied
  --> src/models.rs:14:23
   |
14 |     pub total_amount: f64,
   |                       ^^^ the trait `FromSql<diesel::sql_types::Numeric, Pg>` is not implemented for `f64`, which is required by `f64: FromSqlRow<diesel::sql_types::Numeric, Pg>`
   |
   = note: double check your type mappings via the documentation of `diesel::sql_types::Numeric`
   = note: `diesel::sql_query` requires the loading target to column names for loading values.
           You need to provide a type that explicitly derives `diesel::deserialize::QueryableByName`
   = help: the trait `FromSql<Double, Pg>` is implemented for `f64`
   = help: for that trait implementation, expected `Double`, found `diesel::sql_types::Numeric`
   = note: required for `f64` to implement `diesel::Queryable<diesel::sql_types::Numeric, Pg>`
   = note: required for `f64` to implement `FromSqlRow<diesel::sql_types::Numeric, Pg>`
   = help: see issue #48214

error[E0277]: the trait bound `f64: diesel::Expression` is not satisfied
 --> src/models.rs:8:33
  |
8 | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
  |                                 ^^^^^^^^^^ the trait `diesel::Expression` is not implemented for `f64`, which is required by `f64: AsExpression<diesel::sql_types::Numeric>`
  |
  = help: the following other types implement trait `diesel::Expression`:
            &'a T
            (T0, T1)
            (T0, T1, T2)
            (T0, T1, T2, T3)
            (T0, T1, T2, T3, T4)
            (T0, T1, T2, T3, T4, T5)
            (T0, T1, T2, T3, T4, T5, T6)
            (T0, T1, T2, T3, T4, T5, T6, T7)
          and 137 others
  = note: required for `f64` to implement `AsExpression<diesel::sql_types::Numeric>`
  = note: this error originates in the derive macro `Insertable` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `f64: diesel::Expression` is not satisfied
 --> src/models.rs:8:33
  |
8 | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
  |                                 ^^^^^^^^^^ the trait `diesel::Expression` is not implemented for `f64`, which is required by `&'insert f64: AsExpression<diesel::sql_types::Numeric>`
  |
  = help: the following other types implement trait `diesel::Expression`:
            &'a T
            (T0, T1)
            (T0, T1, T2)
            (T0, T1, T2, T3)
            (T0, T1, T2, T3, T4)
            (T0, T1, T2, T3, T4, T5)
            (T0, T1, T2, T3, T4, T5, T6)
            (T0, T1, T2, T3, T4, T5, T6, T7)
          and 137 others
  = note: required for `&'insert f64` to implement `diesel::Expression`
  = note: required for `&'insert f64` to implement `AsExpression<diesel::sql_types::Numeric>`
  = note: this error originates in the derive macro `Insertable` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `f64: diesel::Expression` is not satisfied
  --> src/models.rs:19:10
   |
19 | #[derive(Insertable, Deserialize)]
   |          ^^^^^^^^^^ the trait `diesel::Expression` is not implemented for `f64`, which is required by `f64: AsExpression<diesel::sql_types::Numeric>`
   |
   = help: the following other types implement trait `diesel::Expression`:
             &'a T
             (T0, T1)
             (T0, T1, T2)
             (T0, T1, T2, T3)
             (T0, T1, T2, T3, T4)
             (T0, T1, T2, T3, T4, T5)
             (T0, T1, T2, T3, T4, T5, T6)
             (T0, T1, T2, T3, T4, T5, T6, T7)
           and 137 others
   = note: required for `f64` to implement `AsExpression<diesel::sql_types::Numeric>`
   = note: this error originates in the derive macro `Insertable` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `f64: diesel::Expression` is not satisfied
  --> src/models.rs:19:10
   |
19 | #[derive(Insertable, Deserialize)]
   |          ^^^^^^^^^^ the trait `diesel::Expression` is not implemented for `f64`, which is required by `&'insert f64: AsExpression<diesel::sql_types::Numeric>`
   |
   = help: the following other types implement trait `diesel::Expression`:
             &'a T
             (T0, T1)
             (T0, T1, T2)
             (T0, T1, T2, T3)
             (T0, T1, T2, T3, T4)
             (T0, T1, T2, T3, T4, T5)
             (T0, T1, T2, T3, T4, T5, T6)
             (T0, T1, T2, T3, T4, T5, T6, T7)
           and 137 others
   = note: required for `&'insert f64` to implement `diesel::Expression`
   = note: required for `&'insert f64` to implement `AsExpression<diesel::sql_types::Numeric>`
   = note: this error originates in the derive macro `Insertable` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `f64: diesel::Expression` is not satisfied
  --> src/models.rs:14:9
   |
14 |     pub total_amount: f64,
   |         ^^^^^^^^^^^^ the trait `diesel::Expression` is not implemented for `f64`, which is required by `f64: AsExpression<diesel::sql_types::Numeric>`
   |
   = help: the following other types implement trait `diesel::Expression`:
             &'a T
             (T0, T1)
             (T0, T1, T2)
             (T0, T1, T2, T3)
             (T0, T1, T2, T3, T4)
             (T0, T1, T2, T3, T4, T5)
             (T0, T1, T2, T3, T4, T5, T6)
             (T0, T1, T2, T3, T4, T5, T6, T7)
           and 137 others
   = note: required for `f64` to implement `AsExpression<diesel::sql_types::Numeric>`

error[E0277]: the trait bound `f64: diesel::Expression` is not satisfied
  --> src/models.rs:14:9
   |
14 |     pub total_amount: f64,
   |         ^^^^^^^^^^^^ the trait `diesel::Expression` is not implemented for `f64`, which is required by `&'insert f64: AsExpression<diesel::sql_types::Numeric>`
   |
   = help: the following other types implement trait `diesel::Expression`:
             &'a T
             (T0, T1)
             (T0, T1, T2)
             (T0, T1, T2, T3)
             (T0, T1, T2, T3, T4)
             (T0, T1, T2, T3, T4, T5)
             (T0, T1, T2, T3, T4, T5, T6)
             (T0, T1, T2, T3, T4, T5, T6, T7)
           and 137 others
   = note: required for `&'insert f64` to implement `diesel::Expression`
   = note: required for `&'insert f64` to implement `AsExpression<diesel::sql_types::Numeric>`

error[E0277]: the trait bound `f64: diesel::Expression` is not satisfied
  --> src/models.rs:23:9
   |
23 |     pub total_amount: f64,
   |         ^^^^^^^^^^^^ the trait `diesel::Expression` is not implemented for `f64`, which is required by `f64: AsExpression<diesel::sql_types::Numeric>`
   |
   = help: the following other types implement trait `diesel::Expression`:
             &'a T
             (T0, T1)
             (T0, T1, T2)
             (T0, T1, T2, T3)
             (T0, T1, T2, T3, T4)
             (T0, T1, T2, T3, T4, T5)
             (T0, T1, T2, T3, T4, T5, T6)
             (T0, T1, T2, T3, T4, T5, T6, T7)
           and 137 others
   = note: required for `f64` to implement `AsExpression<diesel::sql_types::Numeric>`

error[E0277]: the trait bound `f64: diesel::Expression` is not satisfied
  --> src/models.rs:23:9
   |
23 |     pub total_amount: f64,
   |         ^^^^^^^^^^^^ the trait `diesel::Expression` is not implemented for `f64`, which is required by `&'insert f64: AsExpression<diesel::sql_types::Numeric>`
   |
   = help: the following other types implement trait `diesel::Expression`:
             &'a T
             (T0, T1)
             (T0, T1, T2)
             (T0, T1, T2, T3)
             (T0, T1, T2, T3, T4)
             (T0, T1, T2, T3, T4, T5)
             (T0, T1, T2, T3, T4, T5, T6)
             (T0, T1, T2, T3, T4, T5, T6, T7)
           and 137 others
   = note: required for `&'insert f64` to implement `diesel::Expression`
   = note: required for `&'insert f64` to implement `AsExpression<diesel::sql_types::Numeric>`

error[E0277]: the trait bound `f64: diesel::Expression` is not satisfied
  --> src/db.rs:10:10
   |
10 |         .values(&new_order)
   |          ^^^^^^ the trait `diesel::Expression` is not implemented for `f64`, which is required by `&f64: AsExpression<diesel::sql_types::Numeric>`
   |
   = help: the following other types implement trait `diesel::Expression`:
             &'a T
             (T0, T1)
             (T0, T1, T2)
             (T0, T1, T2, T3)
             (T0, T1, T2, T3, T4)
             (T0, T1, T2, T3, T4, T5)
             (T0, T1, T2, T3, T4, T5, T6)
             (T0, T1, T2, T3, T4, T5, T6, T7)
           and 137 others
   = note: required for `&f64` to implement `diesel::Expression`
   = note: required for `&f64` to implement `AsExpression<diesel::sql_types::Numeric>`

error[E0277]: the trait bound `f64: AppearsOnTable<NoFromClause>` is not satisfied
    --> src/db.rs:11:21
     |
11   |         .get_result(conn)
     |          ---------- ^^^^ the trait `AppearsOnTable<NoFromClause>` is not implemented for `f64`, which is required by `InsertStatement<table, query_builder::insert_statement::ValuesClause<(DefaultableColumnInsertValue<ColumnInsertValue<columns::user_id, diesel::expression::bound::Bound<Integer, &i32>>>, DefaultableColumnInsertValue<ColumnInsertValue<columns::total_amount, &f64>>, DefaultableColumnInsertValue<ColumnInsertValue<columns::metadata, diesel::expression::bound::Bound<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, &Value>>>), table>>: LoadQuery<'_, _, _>`
     |          |
     |          required by a bound introduced by this call
     |
     = help: the following other types implement trait `AppearsOnTable<QS>`:
               `&'a T` implements `AppearsOnTable<QS>`
               `(T0, T1)` implements `AppearsOnTable<QS>`
               `(T0, T1, T2)` implements `AppearsOnTable<QS>`
               `(T0, T1, T2, T3)` implements `AppearsOnTable<QS>`
               `(T0, T1, T2, T3, T4)` implements `AppearsOnTable<QS>`
               `(T0, T1, T2, T3, T4, T5)` implements `AppearsOnTable<QS>`
               `(T0, T1, T2, T3, T4, T5, T6)` implements `AppearsOnTable<QS>`
               `(T0, T1, T2, T3, T4, T5, T6, T7)` implements `AppearsOnTable<QS>`
             and 137 others
     = note: required for `&f64` to implement `AppearsOnTable<NoFromClause>`
     = note: required for `DefaultableColumnInsertValue<ColumnInsertValue<columns::total_amount, &f64>>` to implement `InsertValues<_, table>`
     = note: 1 redundant requirement hidden
     = note: required for `(DefaultableColumnInsertValue<ColumnInsertValue<user_id, Bound<Integer, &i32>>>, ..., ...)` to implement `InsertValues<_, table>`
     = note: required for `ValuesClause<(DefaultableColumnInsertValue<ColumnInsertValue<user_id, Bound<Integer, &i32>>>, ..., ...), ...>` to implement `QueryFragment<_>`
     = note: 1 redundant requirement hidden
     = note: required for `InsertStatement<table, ValuesClause<(DefaultableColumnInsertValue<...>, ..., ...), ...>, ..., ...>` to implement `QueryFragment<_>`
     = note: required for `InsertStatement<table, ValuesClause<(DefaultableColumnInsertValue<ColumnInsertValue<..., ...>>, ..., ...), ...>>` to implement `LoadQuery<'_, _, _>`
note: required by a bound in `get_result`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1722:15
     |
1720 |     fn get_result<'query, U>(self, conn: &mut Conn) -> QueryResult<U>
     |        ---------- required by a bound in this associated function
1721 |     where
1722 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::get_result`
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-f4ea1891b9fe6645.long-type-12046924592425831098.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-f4ea1891b9fe6645.long-type-16463581489430962252.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-f4ea1891b9fe6645.long-type-8702231718444061151.txt'
     = note: consider using `--verbose` to print the full type name to the console

error[E0271]: type mismatch resolving `<Pg as SqlDialect>::InsertWithDefaultKeyword == NotSpecialized`
    --> src/db.rs:11:21
     |
11   |         .get_result(conn)
     |          ---------- ^^^^ expected `NotSpecialized`, found `IsoSqlDefaultKeyword`
     |          |
     |          required by a bound introduced by this call
     |
     = note: required for `DefaultableColumnInsertValue<ColumnInsertValue<columns::total_amount, &f64>>` to implement `QueryFragment<Pg>`
     = note: required for `DefaultableColumnInsertValue<ColumnInsertValue<columns::total_amount, &f64>>` to implement `InsertValues<Pg, table>`
     = note: 3 redundant requirements hidden
     = note: required for `InsertStatement<table, ValuesClause<(DefaultableColumnInsertValue<...>, ..., ...), ...>, ..., ...>` to implement `QueryFragment<Pg>`
     = note: required for `InsertStatement<table, ValuesClause<(DefaultableColumnInsertValue<ColumnInsertValue<..., ...>>, ..., ...), ...>>` to implement `LoadQuery<'_, diesel::PgConnection, _>`
note: required by a bound in `get_result`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1722:15
     |
1720 |     fn get_result<'query, U>(self, conn: &mut Conn) -> QueryResult<U>
     |        ---------- required by a bound in this associated function
1721 |     where
1722 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::get_result`
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-f4ea1891b9fe6645.long-type-12046924592425831098.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-f4ea1891b9fe6645.long-type-9860913280699406432.txt'
     = note: consider using `--verbose` to print the full type name to the console

error[E0277]: the trait bound `(i32, i32, f64, NaiveDateTime, Value): FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not satisfied
    --> src/db.rs:11:21
     |
11   |         .get_result(conn)
     |          ---------- ^^^^ the trait `FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not implemented for `(i32, i32, f64, NaiveDateTime, Value)`, which is required by `InsertStatement<table, query_builder::insert_statement::ValuesClause<(DefaultableColumnInsertValue<ColumnInsertValue<columns::user_id, diesel::expression::bound::Bound<Integer, &i32>>>, DefaultableColumnInsertValue<ColumnInsertValue<columns::total_amount, &f64>>, DefaultableColumnInsertValue<ColumnInsertValue<columns::metadata, diesel::expression::bound::Bound<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, &Value>>>), table>>: LoadQuery<'_, _, _>`
     |          |
     |          required by a bound introduced by this call
     |
     = help: the following other types implement trait `FromStaticSqlRow<ST, DB>`:
               `(T0,)` implements `FromStaticSqlRow<(ST0,), __DB>`
               `(T1, T0)` implements `FromStaticSqlRow<(ST1, ST0), __DB>`
               `(T1, T2, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST0), __DB>`
               `(T1, T2, T3, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST0), __DB>`
               `(T1, T2, T3, T4, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T7, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST7, ST0), __DB>`
             and 24 others
note: required for `models::Order` to implement `diesel::Queryable<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
    --> src/models.rs:8:10
     |
8    | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
     |          ^^^^^^^^^ unsatisfied trait bound introduced in this `derive` macro
...
11   | pub struct Order {
     |            ^^^^^
     = note: required for `models::Order` to implement `FromSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
     = note: required for `(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>)` to implement `load_dsl::private::CompatibleType<models::Order, Pg>`
     = note: required for `InsertStatement<table, ValuesClause<(DefaultableColumnInsertValue<ColumnInsertValue<..., ...>>, ..., ...), ...>>` to implement `LoadQuery<'_, diesel::PgConnection, models::Order>`
note: required by a bound in `get_result`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1722:15
     |
1720 |     fn get_result<'query, U>(self, conn: &mut Conn) -> QueryResult<U>
     |        ---------- required by a bound in this associated function
1721 |     where
1722 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::get_result`
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-f4ea1891b9fe6645.long-type-12046924592425831098.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: this error originates in the derive macro `Queryable` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `(i32, i32, f64, NaiveDateTime, Value): FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not satisfied
    --> src/db.rs:17:16
     |
17   |         .first(conn)
     |          ----- ^^^^ the trait `FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not implemented for `(i32, i32, f64, NaiveDateTime, Value)`, which is required by `SelectStatement<FromClause<table>, query_builder::select_clause::DefaultSelectClause<FromClause<table>>, query_builder::distinct_clause::NoDistinctClause, query_builder::where_clause::WhereClause<diesel::expression::grouped::Grouped<diesel::expression::operators::Eq<columns::id, diesel::expression::bound::Bound<Integer, i32>>>>, query_builder::order_clause::NoOrderClause, LimitOffsetClause<LimitClause<diesel::expression::bound::Bound<BigInt, i64>>, NoOffsetClause>>: LoadQuery<'_, _, _>`
     |          |
     |          required by a bound introduced by this call
     |
     = help: the following other types implement trait `FromStaticSqlRow<ST, DB>`:
               `(T0,)` implements `FromStaticSqlRow<(ST0,), __DB>`
               `(T1, T0)` implements `FromStaticSqlRow<(ST1, ST0), __DB>`
               `(T1, T2, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST0), __DB>`
               `(T1, T2, T3, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST0), __DB>`
               `(T1, T2, T3, T4, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T7, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST7, ST0), __DB>`
             and 24 others
note: required for `models::Order` to implement `diesel::Queryable<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
    --> src/models.rs:8:10
     |
8    | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
     |          ^^^^^^^^^ unsatisfied trait bound introduced in this `derive` macro
...
11   | pub struct Order {
     |            ^^^^^
     = note: required for `models::Order` to implement `FromSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
     = note: required for `(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>)` to implement `load_dsl::private::CompatibleType<models::Order, Pg>`
     = note: required for `SelectStatement<FromClause<table>, DefaultSelectClause<FromClause<table>>, NoDistinctClause, ..., ..., ...>` to implement `LoadQuery<'_, diesel::PgConnection, models::Order>`
note: required by a bound in `first`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1779:22
     |
1776 |     fn first<'query, U>(self, conn: &mut Conn) -> QueryResult<U>
     |        ----- required by a bound in this associated function
...
1779 |         Limit<Self>: LoadQuery<'query, Conn, U>,
     |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::first`
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-f4ea1891b9fe6645.long-type-13104014775257459898.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: this error originates in the derive macro `Queryable` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `(i32, i32, f64, NaiveDateTime, Value): FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not satisfied
    --> src/db.rs:24:15
     |
24   |         .load(conn)
     |          ---- ^^^^ the trait `FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not implemented for `(i32, i32, f64, NaiveDateTime, Value)`, which is required by `SelectStatement<FromClause<table>, query_builder::select_clause::DefaultSelectClause<FromClause<table>>, query_builder::distinct_clause::NoDistinctClause, query_builder::where_clause::WhereClause<diesel::expression::grouped::Grouped<diesel::expression::operators::Eq<columns::user_id, diesel::expression::bound::Bound<Integer, i32>>>>>: LoadQuery<'_, _, _>`
     |          |
     |          required by a bound introduced by this call
     |
     = help: the following other types implement trait `FromStaticSqlRow<ST, DB>`:
               `(T0,)` implements `FromStaticSqlRow<(ST0,), __DB>`
               `(T1, T0)` implements `FromStaticSqlRow<(ST1, ST0), __DB>`
               `(T1, T2, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST0), __DB>`
               `(T1, T2, T3, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST0), __DB>`
               `(T1, T2, T3, T4, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T7, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST7, ST0), __DB>`
             and 24 others
note: required for `models::Order` to implement `diesel::Queryable<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
    --> src/models.rs:8:10
     |
8    | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
     |          ^^^^^^^^^ unsatisfied trait bound introduced in this `derive` macro
...
11   | pub struct Order {
     |            ^^^^^
     = note: required for `models::Order` to implement `FromSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
     = note: required for `(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>)` to implement `load_dsl::private::CompatibleType<models::Order, Pg>`
     = note: required for `SelectStatement<FromClause<table>, DefaultSelectClause<FromClause<table>>, NoDistinctClause, WhereClause<...>>` to implement `LoadQuery<'_, diesel::PgConnection, models::Order>`
note: required by a bound in `diesel::RunQueryDsl::load`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1542:15
     |
1540 |     fn load<'query, U>(self, conn: &mut Conn) -> QueryResult<Vec<U>>
     |        ---- required by a bound in this associated function
1541 |     where
1542 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::load`
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-f4ea1891b9fe6645.long-type-451606845660155404.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: this error originates in the derive macro `Queryable` (in Nightly builds, run with -Z macro-backtrace for more info)

Some errors have detailed explanations: E0271, E0277.
For more information about an error, try `rustc --explain E0271`.
warning: `order-diesel-jsonb-example` (bin "order-diesel-jsonb-example") generated 1 warning
error: could not compile `order-diesel-jsonb-example` (bin "order-diesel-jsonb-example") due to 24 previous errors; 1 warning emitted
```

## Fix: change `total_amount` type from `f64` to `BigDecimal`

That's a lot of errors, let's fix it step by step.

First, let's see the error for `total_amount` field

```bash
error[E0277]: the trait bound `f64: FromSqlRow<diesel::sql_types::Numeric, Pg>` is not satisfied
  --> src/models.rs:14:23
   |
14 |     pub total_amount: f64,
   |                       ^^^ the trait `FromSql<diesel::sql_types::Numeric, Pg>` is not implemented for `f64`, which is required by `f64: FromSqlRow<diesel::sql_types::Numeric, Pg>`
   |
   = note: double check your type mappings via the documentation of `diesel::sql_types::Numeric`
   = note: `diesel::sql_query` requires the loading target to column names for loading values.
           You need to provide a type that explicitly derives `diesel::deserialize::QueryableByName`
   = help: the trait `FromSql<Double, Pg>` is implemented for `f64`
   = help: for that trait implementation, expected `Double`, found `diesel::sql_types::Numeric`
   = note: required for `f64` to implement `diesel::Queryable<diesel::sql_types::Numeric, Pg>`
   = note: required for `f64` to implement `FromSqlRow<diesel::sql_types::Numeric, Pg>`
   = help: see issue #48214

error[E0277]: the trait bound `f64: diesel::Expression` is not satisfied
 --> src/models.rs:8:33
  |
8 | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
  |                                 ^^^^^^^^^^ the trait `diesel::Expression` is not implemented for `f64`, which is required by `f64: AsExpression<diesel::sql_types::Numeric>`
  |
  = help: the following other types implement trait `diesel::Expression`:
            &'a T
            (T0, T1)
            (T0, T1, T2)
            (T0, T1, T2, T3)
            (T0, T1, T2, T3, T4)
            (T0, T1, T2, T3, T4, T5)
            (T0, T1, T2, T3, T4, T5, T6)
            (T0, T1, T2, T3, T4, T5, T6, T7)
          and 137 others
  = note: required for `f64` to implement `AsExpression<diesel::sql_types::Numeric>`
  = note: this error originates in the derive macro `Insertable` (in Nightly builds, run with -Z macro-backtrace for more info)
```

Notice, in migration file, `total_amount` column type is `Decimal`, and diesel will use `Numeric` type in it's SQL query.

Here is the source code of `diesel::sql_types::Numeric`

```rust
/// The arbitrary precision numeric SQL type.
///
/// This type is only supported on PostgreSQL and MySQL.
/// On SQLite, [`Double`] should be used instead.
///
/// ### [`ToSql`](crate::serialize::ToSql) impls
///
/// - [`bigdecimal::BigDecimal`] with `feature = ["numeric"]`
///
/// ### [`FromSql`](crate::deserialize::FromSql) impls
///
/// - [`bigdecimal::BigDecimal`] with `feature = ["numeric"]`
///
/// [`bigdecimal::BigDecimal`]: /bigdecimal/struct.BigDecimal.html
#[derive(Debug, Clone, Copy, Default, QueryId, SqlType)]
#[diesel(postgres_type(oid = 1700, array_oid = 1231))]
#[diesel(mysql_type(name = "Numeric"))]
#[diesel(sqlite_type(name = "Double"))]
pub struct Numeric;
```

We should tell diesel to use `numeric` type.

```sql
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL,
  total_amount DECIMAL(10, 2) NOT NULL, -- 👈👈👈👈👈👈 🙋
  order_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  metadata JSONB
);
```

Let's enable `numeric` feature to `diesel` dependency to correctly deserialize `total_amount` field.

```bash
cargo add diesel -F numeric
    Updating crates.io index
      Adding diesel v2.2.6 to dependencies
             Features:
             + 32-column-tables
             + chrono
             + numeric
             + postgres
             + postgres_backend
             + serde_json
             + with-deprecated
             - 128-column-tables
             - 64-column-tables
             - __with_asan_tests
             - extras
             - huge-tables
             - i-implement-a-third-party-backend-and-opt-into-breaking-changes
             - ipnet-address
             - large-tables
             - mysql
             - mysql_backend
             - mysqlclient-src
             - network-address
             - pq-src
             - quickcheck
             - r2d2
             - returning_clauses_for_sqlite_3_35
             - sqlite
             - time
             - unstable
             - uuid
             - without-deprecated
     Locking 4 packages to latest compatible versions
      Adding bigdecimal v0.4.7
      Adding libm v0.2.11
      Adding num-bigint v0.4.6
      Adding num-integer v0.1.46
```

Now, let's build it again.

Still lots of errors:

```bash
   Compiling order-diesel-jsonb-example v0.1.0
warning: unused import: `serde_json::json`
 --> src/db.rs:3:5
  |
3 | use serde_json::json;
  |     ^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_imports)]` on by default

error[E0277]: the trait bound `Value: FromSqlRow<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>` is not satisfied
  --> src/models.rs:18:19
   |
18 |     pub metadata: Value,  // This will use JSONB
   |                   ^^^^^ the trait `FromSql<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>` is not implemented for `Value`, which is required by `Value: FromSqlRow<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>`
   |
   = note: double check your type mappings via the documentation of `diesel::sql_types::Nullable<diesel::sql_types::Jsonb>`
   = note: `diesel::sql_query` requires the loading target to column names for loading values.
           You need to provide a type that explicitly derives `diesel::deserialize::QueryableByName`
   = help: the following other types implement trait `FromSql<A, DB>`:
             `Value` implements `FromSql<Json, Pg>`
             `Value` implements `FromSql<diesel::sql_types::Jsonb, Pg>`
   = note: required for `Value` to implement `diesel::Queryable<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>`
   = note: required for `Value` to implement `FromSqlRow<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>`
   = help: see issue #48214

error[E0277]: the trait bound `(i32, i32, BigDecimal, NaiveDateTime, Value): FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not satisfied
    --> src/db.rs:11:21
     |
11   |         .get_result(conn)
     |          ---------- ^^^^ the trait `FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not implemented for `(i32, i32, BigDecimal, NaiveDateTime, Value)`, which is required by `InsertStatement<table, query_builder::insert_statement::ValuesClause<(DefaultableColumnInsertValue<ColumnInsertValue<columns::user_id, diesel::expression::bound::Bound<Integer, &i32>>>, DefaultableColumnInsertValue<ColumnInsertValue<columns::total_amount, diesel::expression::bound::Bound<diesel::sql_types::Numeric, &BigDecimal>>>, DefaultableColumnInsertValue<ColumnInsertValue<columns::metadata, diesel::expression::bound::Bound<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, &Value>>>), table>>: LoadQuery<'_, _, _>`
     |          |
     |          required by a bound introduced by this call
     |
     = help: the following other types implement trait `FromStaticSqlRow<ST, DB>`:
               `(T0,)` implements `FromStaticSqlRow<(ST0,), __DB>`
               `(T1, T0)` implements `FromStaticSqlRow<(ST1, ST0), __DB>`
               `(T1, T2, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST0), __DB>`
               `(T1, T2, T3, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST0), __DB>`
               `(T1, T2, T3, T4, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T7, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST7, ST0), __DB>`
             and 24 others
note: required for `models::Order` to implement `diesel::Queryable<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
    --> src/models.rs:9:10
     |
9    | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
     |          ^^^^^^^^^ unsatisfied trait bound introduced in this `derive` macro
...
12   | pub struct Order {
     |            ^^^^^
     = note: required for `models::Order` to implement `FromSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
     = note: required for `(Integer, Integer, Numeric, Timestamp, Nullable<Jsonb>)` to implement `load_dsl::private::CompatibleType<models::Order, Pg>`
     = note: required for `InsertStatement<table, ValuesClause<(..., ..., ...), ...>>` to implement `LoadQuery<'_, diesel::PgConnection, models::Order>`
note: required by a bound in `get_result`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1722:15
     |
1720 |     fn get_result<'query, U>(self, conn: &mut Conn) -> QueryResult<U>
     |        ---------- required by a bound in this associated function
1721 |     where
1722 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::get_result`
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-cdb3b656aa15b263.long-type-9790004947608049719.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-cdb3b656aa15b263.long-type-2452195376562117312.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: this error originates in the derive macro `Queryable` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `(i32, i32, BigDecimal, NaiveDateTime, Value): FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not satisfied
    --> src/db.rs:17:16
     |
17   |         .first(conn)
     |          ----- ^^^^ the trait `FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not implemented for `(i32, i32, BigDecimal, NaiveDateTime, Value)`, which is required by `SelectStatement<FromClause<table>, query_builder::select_clause::DefaultSelectClause<FromClause<table>>, query_builder::distinct_clause::NoDistinctClause, query_builder::where_clause::WhereClause<diesel::expression::grouped::Grouped<diesel::expression::operators::Eq<columns::id, diesel::expression::bound::Bound<Integer, i32>>>>, query_builder::order_clause::NoOrderClause, LimitOffsetClause<LimitClause<diesel::expression::bound::Bound<diesel::sql_types::BigInt, i64>>, NoOffsetClause>>: LoadQuery<'_, _, _>`
     |          |
     |          required by a bound introduced by this call
     |
     = help: the following other types implement trait `FromStaticSqlRow<ST, DB>`:
               `(T0,)` implements `FromStaticSqlRow<(ST0,), __DB>`
               `(T1, T0)` implements `FromStaticSqlRow<(ST1, ST0), __DB>`
               `(T1, T2, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST0), __DB>`
               `(T1, T2, T3, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST0), __DB>`
               `(T1, T2, T3, T4, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T7, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST7, ST0), __DB>`
             and 24 others
note: required for `models::Order` to implement `diesel::Queryable<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
    --> src/models.rs:9:10
     |
9    | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
     |          ^^^^^^^^^ unsatisfied trait bound introduced in this `derive` macro
...
12   | pub struct Order {
     |            ^^^^^
     = note: required for `models::Order` to implement `FromSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
     = note: required for `(Integer, Integer, Numeric, Timestamp, Nullable<Jsonb>)` to implement `load_dsl::private::CompatibleType<models::Order, Pg>`
     = note: required for `SelectStatement<FromClause<table>, DefaultSelectClause<...>, ..., ..., ..., ...>` to implement `LoadQuery<'_, diesel::PgConnection, models::Order>`
note: required by a bound in `first`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1779:22
     |
1776 |     fn first<'query, U>(self, conn: &mut Conn) -> QueryResult<U>
     |        ----- required by a bound in this associated function
...
1779 |         Limit<Self>: LoadQuery<'query, Conn, U>,
     |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::first`
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-cdb3b656aa15b263.long-type-12865645728958808655.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-cdb3b656aa15b263.long-type-2452195376562117312.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: this error originates in the derive macro `Queryable` (in Nightly builds, run with -Z macro-backtrace for more info)

error[E0277]: the trait bound `(i32, i32, BigDecimal, NaiveDateTime, Value): FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not satisfied
    --> src/db.rs:24:15
     |
24   |         .load(conn)
     |          ---- ^^^^ the trait `FromStaticSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>` is not implemented for `(i32, i32, BigDecimal, NaiveDateTime, Value)`, which is required by `SelectStatement<FromClause<table>, query_builder::select_clause::DefaultSelectClause<FromClause<table>>, query_builder::distinct_clause::NoDistinctClause, query_builder::where_clause::WhereClause<diesel::expression::grouped::Grouped<diesel::expression::operators::Eq<columns::user_id, diesel::expression::bound::Bound<Integer, i32>>>>>: LoadQuery<'_, _, _>`
     |          |
     |          required by a bound introduced by this call
     |
     = help: the following other types implement trait `FromStaticSqlRow<ST, DB>`:
               `(T0,)` implements `FromStaticSqlRow<(ST0,), __DB>`
               `(T1, T0)` implements `FromStaticSqlRow<(ST1, ST0), __DB>`
               `(T1, T2, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST0), __DB>`
               `(T1, T2, T3, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST0), __DB>`
               `(T1, T2, T3, T4, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST0), __DB>`
               `(T1, T2, T3, T4, T5, T6, T7, T0)` implements `FromStaticSqlRow<(ST1, ST2, ST3, ST4, ST5, ST6, ST7, ST0), __DB>`
             and 24 others
note: required for `models::Order` to implement `diesel::Queryable<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
    --> src/models.rs:9:10
     |
9    | #[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
     |          ^^^^^^^^^ unsatisfied trait bound introduced in this `derive` macro
...
12   | pub struct Order {
     |            ^^^^^
     = note: required for `models::Order` to implement `FromSqlRow<(Integer, Integer, diesel::sql_types::Numeric, diesel::sql_types::Timestamp, diesel::sql_types::Nullable<diesel::sql_types::Jsonb>), Pg>`
     = note: required for `(Integer, Integer, Numeric, Timestamp, Nullable<Jsonb>)` to implement `load_dsl::private::CompatibleType<models::Order, Pg>`
     = note: required for `SelectStatement<FromClause<table>, DefaultSelectClause<FromClause<...>>, ..., ...>` to implement `LoadQuery<'_, diesel::PgConnection, models::Order>`
note: required by a bound in `diesel::RunQueryDsl::load`
    --> /Users/dylan/.cargo/registry/src/index.crates.io-6f17d22bba15001f/diesel-2.2.6/src/query_dsl/mod.rs:1542:15
     |
1540 |     fn load<'query, U>(self, conn: &mut Conn) -> QueryResult<Vec<U>>
     |        ---- required by a bound in this associated function
1541 |     where
1542 |         Self: LoadQuery<'query, Conn, U>,
     |               ^^^^^^^^^^^^^^^^^^^^^^^^^^ required by this bound in `RunQueryDsl::load`
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-cdb3b656aa15b263.long-type-17907481964813576637.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: the full name for the type has been written to '/Users/dylan/projects/order-diesel-jsonb-example/target/debug/deps/order_diesel_jsonb_example-cdb3b656aa15b263.long-type-2452195376562117312.txt'
     = note: consider using `--verbose` to print the full type name to the console
     = note: this error originates in the derive macro `Queryable` (in Nightly builds, run with -Z macro-backtrace for more info)

For more information about this error, try `rustc --explain E0277`.
warning: `order-diesel-jsonb-example` (bin "order-diesel-jsonb-example") generated 1 warning
error: could not compile `order-diesel-jsonb-example` (bin "order-diesel-jsonb-example") due to 4 previous errors; 1 warning emitted
```

## Fix: Change `metadata` type from `Value` to `Option<Value>`

Next, let's see the error for `metadata` field.

```bash
> cargo build
   Compiling order-diesel-jsonb-example v0.1.0
warning: unused import: `serde_json::json`
 --> src/db.rs:3:5
  |
3 | use serde_json::json;
  |     ^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_imports)]` on by default

error[E0277]: the trait bound `Value: FromSqlRow<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>` is not satisfied
  --> src/models.rs:18:19
   |
18 |     pub metadata: Value,  // This will use JSONB
   |                   ^^^^^ the trait `FromSql<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>` is not implemented for `Value`, which is required by `Value: FromSqlRow<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>`
   |
   = note: double check your type mappings via the documentation of `diesel::sql_types::Nullable<diesel::sql_types::Jsonb>`
   = note: `diesel::sql_query` requires the loading target to column names for loading values.
           You need to provide a type that explicitly derives `diesel::deserialize::QueryableByName`
   = help: the following other types implement trait `FromSql<A, DB>`:
             `Value` implements `FromSql<Json, Pg>`
             `Value` implements `FromSql<diesel::sql_types::Jsonb, Pg>`
   = note: required for `Value` to implement `diesel::Queryable<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>`
   = note: required for `Value` to implement `FromSqlRow<diesel::sql_types::Nullable<diesel::sql_types::Jsonb>, Pg>`
   = help: see issue #48214
```

The reason for that error is that we declare `metadata` as `NOT NULL` in migration sql file, which is not aligned with `metadata` type in model defination. The fix is easy by changing datetype for `metadata` from `Value` to `Option<Value>`.

```rust
#[derive(Queryable, Selectable, Insertable, Serialize, Deserialize, Debug)]
#[diesel(table_name = orders)]
#[diesel(check_for_backend(diesel::pg::Pg))]
pub struct Order {
    pub id: i32,
    pub user_id: i32,
    // pub total_amount: f64,
    pub total_amount: BigDecimal,
    pub order_date: NaiveDateTime,
    // pub metadata: Value,  // This will use JSONB ❌
    pub metadata: Option<Value>,  // This will use JSONB ✅
}
```

`metadata` column type:

```sql
CREATE TABLE orders (
  id SERIAL PRIMARY KEY,
  user_id INTEGER NOT NULL,
  total_amount DECIMAL(10, 2) NOT NULL,
  order_date TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  metadata JSONB -- 👈👈👈👈👈👈 🙋
);
```

Now, let's build it.

```bash
   Compiling order-diesel-jsonb-example v0.1.0
warning: unused import: `serde_json::json`
 --> src/db.rs:3:5
  |
3 | use serde_json::json;
  |     ^^^^^^^^^^^^^^^^
  |
  = note: `#[warn(unused_imports)]` on by default

warning: function `get_order_by_id` is never used
  --> src/db.rs:15:8
   |
15 | pub fn get_order_by_id(conn: &mut PgConnection, order_id: i32) -> Result<Order> {
   |        ^^^^^^^^^^^^^^^
   |
   = note: `#[warn(dead_code)]` on by default

warning: function `get_orders_by_user` is never used
  --> src/db.rs:21:8
   |
21 | pub fn get_orders_by_user(conn: &mut PgConnection, user_id_param: i32) -> Result<Vec<Order>> {
   |        ^^^^^^^^^^^^^^^^^^

warning: `order-diesel-jsonb-example` (bin "order-diesel-jsonb-example") generated 3 warnings (run `cargo fix --bin "order-diesel-jsonb-example"` to apply 1 suggestion)
```

🎉🎉🎉

We could also add `NOT NULL` to `metadata` column type and keep `metadata` as `Value` in model definition.

# Create orders

Finally, let's create some orders and see the result in database.

```bash
cargo run --bin order-diesel-jsonb-example
```

Connect the database using `psql diesel_demo` and see the result:

```sql
dylan@/tmp:diesel_demo> \d
+--------+----------------------------+----------+-------+
| Schema | Name                       | Type     | Owner |
|--------+----------------------------+----------+-------|
| public | __diesel_schema_migrations | table    | dylan  |
| public | orders                     | table    | dylan  |
| public | orders_id_seq              | sequence | dylan  |
+--------+----------------------------+----------+-------+
SELECT 3
Time: 0.008s
dylan@/tmp:diesel_demo> select * from orders;
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------+
| id | user_id | total_amount | order_date                 | metadata                                                                    |
|----+---------+--------------+----------------------------+-----------------------------------------------------------------------------|
| 1  | 1       | 0.80         | 2024-12-17 03:05:10.732408 | {"items": ["book", "pen"], "gift_wrap": true, "shipping_method": "express"} |
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------+
SELECT 1
Time: 0.006s
```

# Query orders

As a next step, let's see how to query orders.

Below is the query result for `SELECT * FROM orders WHERE metadata @> '{"address": "Article Circle Expressway 2"}'`:

```sql
dylan@/tmp:diesel_demo> select * from orders;
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------+
| id | user_id | total_amount | order_date                 | metadata                                                                                                              |
|----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------|
| 1  | 1       | 0.80         | 2024-12-17 03:05:10.732408 | {"items": ["book", "pen"], "gift_wrap": true, "shipping_method": "express"}                                           |
| 2  | 1       | 0.80         | 2024-12-17 03:08:16.591275 | {"items": ["book", "pen"], "gift_wrap": true, "shipping_method": "express"}                                           |
| 3  | 1       | 0.80         | 2024-12-17 05:46:41.173109 | {"items": ["book", "pen"], "address": "123 Main St, Anytown, USA", "gift_wrap": true, "shipping_method": "express"}   |
| 4  | 1       | 0.80         | 2024-12-17 05:47:40.956483 | {"items": ["book", "pen"], "address": "Article Circle Expressway 2", "gift_wrap": true, "shipping_method": "express"} |
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------+
SELECT 4
Time: 0.006s
dylan@/tmp:diesel_demo>
Time: 0.000s
dylan@/tmp:diesel_demo>
Time: 0.000s
dylan@/tmp:diesel_demo> SELECT * FROM orders WHERE metadata @> '{"address": "Article Circle Expressway 2"}';
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------+
| id | user_id | total_amount | order_date                 | metadata                                                                                                              |
|----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------|
| 4  | 1       | 0.80         | 2024-12-17 05:47:40.956483 | {"items": ["book", "pen"], "address": "Article Circle Expressway 2", "gift_wrap": true, "shipping_method": "express"} |
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------+
SELECT 1
Time: 0.013s
```

Let's see how we can archieve the same result using Diesel.

We can use operators for jsonb types:

```rust
pub fn get_orders_by_address(
    conn: &mut PgConnection, metadata: &serde_json::Value,
) -> QueryResult<Vec<Order>> {
    use crate::schema::orders::dsl::{metadata as orders_metadata, orders};
    let query = orders.filter(orders_metadata.contains(metadata));
    let debug = diesel::debug_query::<diesel::pg::Pg, _>(&query);
    println!("The insert query: {:#?}", debug);
    query.get_results(conn)
}
```

The code above uses `contains` jsonb operator(`@>`) to query the orders by metadata. You can use `{"address": "Article Circle Expressway 2"}` as the creteria.

Now, let's modify `main.rs` to call the `get_orders_by_address` method.

```rust

fn main() {
    let conn = &mut establish_connection();

    // // Example usage
    // let new_order = models::NewOrder {
    //     user_id: 1,
    //     // total_amount: 99.99,
    //     total_amount: BigDecimal::from_str("0.80").unwrap(),
    //     metadata: serde_json::json!({
    //         "items": ["book", "pen"],
    //         "shipping_method": "express",
    //         "gift_wrap": true,
    //         "address": "Article Circle Expressway 2",
    //         // "address": "123 Main St, Anytown, USA",
    //     }),
    // };

    // match db::create_order(conn, new_order) {
    //     Ok(order) => println!("Created order: {:?}", order),
    //     Err(e) => eprintln!("Error creating order: {}", e),
    // }

    let metadata_address: serde_json::Value = serde_json::json!({"address": "Article Circle Expressway 2"});
    match db::get_orders_by_address(conn, &metadata_address) {
        Ok(orders) => println!("Orders by address: {:#?}", orders),
        Err(e) => eprintln!("Error getting orders by address: {}", e),
    }
}
```

Running the query using `contains` jsonb operator(`@>`) gives us the same result. Notice the `@>` operator in the SQL query.

```bash
    Finished `dev` profile [unoptimized + debuginfo] target(s) in 1.46s
     Running `target/debug/order-diesel-jsonb-example`
The insert query: Query {
    sql: "SELECT \"orders\".\"id\", \"orders\".\"user_id\", \"orders\".\"total_amount\", \"orders\".\"order_date\", \"orders\".\"metadata\" FROM \"orders\" WHERE (\"orders\".\"metadata\" @> $1)",
    binds: [
        Object {
            "address": String("Article Circle Expressway 2"),
        },
    ],
}
Orders by address: [
    Order {
        id: 4,
        user_id: 1,
        total_amount: BigDecimal("80e-2"),
        order_date: 2024-12-17T05:47:40.956483,
        metadata: Some(
            Object {
                "address": String("Article Circle Expressway 2"),
                "gift_wrap": Bool(true),
                "items": Array [
                    String("book"),
                    String("pen"),
                ],
                "shipping_method": String("express"),
            },
        ),
    },
]
```

Below is the source code for `contains` method in [diesel source code](https://github.com/diesel-rs/diesel/blob/b705023d85e6ef76292a427aa70f21c28a5902ff/diesel/src/pg/expression/expression_methods.rs#L2771).

````rust
/// PostgreSQL specific methods present on JSONB expressions.
#[cfg(feature = "postgres_backend")]
pub trait PgJsonbExpressionMethods: Expression + Sized {
    /// Creates a PostgreSQL `@>` expression.
    ///
    /// This operator checks whether left hand side JSONB value contains right hand side JSONB value
    ///
    /// # Example
    ///
    /// ```rust
    /// # include!("../../doctest_setup.rs");
    /// #
    /// # table! {
    /// #    contacts {
    /// #        id -> Integer,
    /// #        name -> VarChar,
    /// #        address -> Jsonb,
    /// #    }
    /// # }
    /// #
    /// # fn main() {
    /// #     run_test().unwrap();
    /// # }
    /// #
    /// # #[cfg(feature = "serde_json")]
    /// # fn run_test() -> QueryResult<()> {
    /// #     use self::contacts::dsl::*;
    /// #     let conn = &mut establish_connection();
    /// #     diesel::sql_query("DROP TABLE IF EXISTS contacts").execute(conn).unwrap();
    /// #     diesel::sql_query("CREATE TABLE contacts (
    /// #         id SERIAL PRIMARY KEY,
    /// #         name VARCHAR NOT NULL,
    /// #         address JSONB NOT NULL
    /// #     )").execute(conn).unwrap();
    /// #
    /// let easter_bunny_address: serde_json::Value = serde_json::json!({
    ///     "street": "123 Carrot Road",
    ///     "province": "Easter Island",
    ///     "region": "Valparaíso",
    ///     "country": "Chile",
    ///     "postcode": "88888",
    /// });
    /// diesel::insert_into(contacts)
    ///     .values((name.eq("Bunny"), address.eq(&easter_bunny_address)))
    ///     .execute(conn)?;
    ///
    /// let country_chile: serde_json::Value = serde_json::json!({"country": "Chile"});
    /// let contains_country_chile = contacts.select(address.contains(&country_chile)).get_result::<bool>(conn)?;
    /// assert!(contains_country_chile);
    /// #     Ok(())
    /// # }
    /// # #[cfg(not(feature = "serde_json"))]
    /// # fn run_test() -> QueryResult<()> {
    /// #     Ok(())
    /// # }
    /// ```
    fn contains<T>(self, other: T) -> dsl::Contains<Self, T>
    where
        Self::SqlType: SqlType,
        T: AsExpression<Self::SqlType>,
    {
        Grouped(Contains::new(self, other.as_expression()))
    }
}
````

# Update order

Now, let's update the order by filtering the order by address.

```rust
// write update order by address function
pub fn update_order_by_address(
    conn: &mut PgConnection, address: &str, new_amount: BigDecimal,
) -> QueryResult<usize> {
    use crate::schema::orders::dsl::{metadata, orders, total_amount};

    let query = diesel::update(orders)
        .filter(metadata.contains(json!({ "address": address })))
        .set(total_amount.eq(new_amount));

    let debug = diesel::debug_query::<diesel::pg::Pg, _>(&query);
    println!("The update query: {:#?}", debug);

    query.execute(conn)
}
```

This function will update the order by filtering the order by address.

Now, let's modify `main.rs` to call the `update_order_by_address` method.

```rust
fn main() {
    let conn = &mut establish_connection();
    // // Example usage
    // let new_order = models::NewOrder {
    //     user_id: 1,
    //     // total_amount: 99.99,
    //     total_amount: BigDecimal::from_str("0.80").unwrap(),
    //     metadata: serde_json::json!({
    //         "items": ["book", "pen"],
    //         "shipping_method": "express",
    //         "gift_wrap": true,
    //         "address": "Article Circle Expressway 2",
    //         // "address": "123 Main St, Anytown, USA",
    //     }),
    // };

    // match db::create_order(conn, new_order) {
    //     Ok(order) => println!("Created order: {:?}", order),
    //     Err(e) => eprintln!("Error creating order: {}", e),
    // }


    // Query
    // let metadata_address: serde_json::Value = serde_json::json!({"address": "Article Circle Expressway 2"});
    // match db::get_orders_by_address(conn, &metadata_address) {
    //     Ok(orders) => println!("Orders by address: {:#?}", orders),
    //     Err(e) => eprintln!("Error getting orders by address: {}", e),
    // }

    // Update
    let address = "Article Circle Expressway 2";
    let new_amount = BigDecimal::from_f64(1234.56).unwrap();
    match db::update_order_by_address(conn, address, new_amount) {
        Ok(orders) => println!("Orders by address: {:#?}", orders),
        Err(e) => eprintln!("Error getting orders by address: {}", e),
    }
}
```

Below is the data changes after the update:

```sql
dylan@/tmp:diesel_demo> SELECT * FROM orders WHERE metadata @> '{"address": "Article Circle Expressway 2"}';
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------+
| id | user_id | total_amount | order_date                 | metadata                                                                                                              |
|----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------|
| 4  | 1       | 0.80         | 2024-12-17 05:47:40.956483 | {"items": ["book", "pen"], "address": "Article Circle Expressway 2", "gift_wrap": true, "shipping_method": "express"} |
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------+
SELECT 1
Time: 0.017s
dylan@/tmp:diesel_demo>
Time: 0.000s
dylan@/tmp:diesel_demo>
Time: 0.000s
dylan@/tmp:diesel_demo>
Time: 0.000s
dylan@/tmp:diesel_demo> SELECT * FROM orders WHERE metadata @> '{"address": "Article Circle Expressway 2"}';
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------+
| id | user_id | total_amount | order_date                 | metadata                                                                                                              |
|----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------|
| 4  | 1       | 1234.56      | 2024-12-17 05:47:40.956483 | {"items": ["book", "pen"], "address": "Article Circle Expressway 2", "gift_wrap": true, "shipping_method": "express"} |
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------+
SELECT 1
Time: 0.007s
```

# Delete order

Let's write the code to delete the order by filtering the order by address.

```rust
// write delete order by address function
pub fn delete_order_by_address(
    conn: &mut PgConnection, address: &str,
) -> QueryResult<usize> {
    use crate::schema::orders::dsl::{metadata, orders};

    let query = diesel::delete(orders)
        .filter(metadata.contains(json!({ "address": address })));

    let debug = diesel::debug_query::<diesel::pg::Pg, _>(&query);
    println!("The delete query: {:#?}", debug);

    query.execute(conn)
}
```

Let's modify `main.rs` to call the `delete_order_by_address` method.

```rust
fn main() {
    let conn = &mut establish_connection();

    // // Example usage
    // let new_order = models::NewOrder {
    //     user_id: 1,
    //     // total_amount: 99.99,
    //     total_amount: BigDecimal::from_str("0.80").unwrap(),
    //     metadata: serde_json::json!({
    //         "items": ["book", "pen"],
    //         "shipping_method": "express",
    //         "gift_wrap": true,
    //         "address": "Article Circle Expressway 2",
    //         // "address": "123 Main St, Anytown, USA",
    //     }),
    // };

    // match db::create_order(conn, new_order) {
    //     Ok(order) => println!("Created order: {:?}", order),
    //     Err(e) => eprintln!("Error creating order: {}", e),
    // }


    // Query
    // let metadata_address: serde_json::Value = serde_json::json!({"address": "Article Circle Expressway 2"});
    // match db::get_orders_by_address(conn, &metadata_address) {
    //     Ok(orders) => println!("Orders by address: {:#?}", orders),
    //     Err(e) => eprintln!("Error getting orders by address: {}", e),
    // }

    // Update
    // let address = "Article Circle Expressway 2";
    // let new_amount = BigDecimal::from_f64(1234.56).unwrap();
    // match db::update_order_by_address(conn, address, new_amount) {
    //     Ok(orders) => println!("Orders by address: {:#?}", orders),
    //     Err(e) => eprintln!("Error getting orders by address: {}", e),
    // }

    // Delete
    let address = "Article Circle Expressway 2";
    match db::delete_order_by_address(conn, address) {
        Ok(orders) => println!("Orders by address: {:#?}", orders),
        Err(e) => eprintln!("Error getting orders by address: {}", e),
    }
}
```

Below is the data changes after the delete:

```sql
dylan@/tmp:diesel_demo> SELECT * FROM orders WHERE metadata @> '{"address": "Article Circle Expressway 2"}';
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------+
| id | user_id | total_amount | order_date                 | metadata                                                                                                              |
|----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------|
| 4  | 1       | 1234.56      | 2024-12-17 05:47:40.956483 | {"items": ["book", "pen"], "address": "Article Circle Expressway 2", "gift_wrap": true, "shipping_method": "express"} |
+----+---------+--------------+----------------------------+-----------------------------------------------------------------------------------------------------------------------+
SELECT 1
Time: 0.007s
dylan@/tmp:diesel_demo>
Time: 0.000s
dylan@/tmp:diesel_demo>
Time: 0.000s
dylan@/tmp:diesel_demo>
Time: 0.000s
dylan@/tmp:diesel_demo> SELECT * FROM orders WHERE metadata @> '{"address": "Article Circle Expressway 2"}';
+----+---------+--------------+------------+----------+
| id | user_id | total_amount | order_date | metadata |
|----+---------+--------------+------------+----------|
+----+---------+--------------+------------+----------+
SELECT 0
Time: 0.006s
```

# Summary

In this demo, we have seen how to setup diesel and run migrations, write sql for migration, create orders, query orders, update orders, delete orders.

We learned how to use Diesel to query the orders by metadata using the `@>` jsonb operator.

We have also seen how to update the order by filtering the order by address and delete the order by filtering the order by address.

# Refs

Diesel schema
https://diesel.rs/guides/schema-in-depth.html

Diesel getting started
https://diesel.rs/guides/getting-started
