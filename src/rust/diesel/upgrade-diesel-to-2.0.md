# Upgrade diesel to 2.0

- [An introduction to diesel 2.0](#an-introduction-to-diesel-20)
- [Upgrade to diesel 2.0](#upgrade-to-diesel-20)
  - [Upgrade diesel version in Cargo.toml](#upgrade-diesel-version-in-cargotoml)
  - [Add mut to PgConnection and dao functions](#add-mut-to-pgconnection-and-dao-functions)
  - [Derive attributes](#derive-attributes)
- [Refs](#refs)

# An introduction to diesel 2.0

Diesel 2.0 has breaking changes compared to 1.4.x.

Any code base migrating from Diesel 1.4.x to Diesel 2.0 is expected to be affected at least by the following changes:

- [Diesel now requires a mutable reference to the connection](https://diesel.rs/guides/migration_guide.html#2-0-0-mutable-connection)
- [Changed derive attributes](https://diesel.rs/guides/migration_guide.html#2-0-0-derive-attributes)

# Upgrade to diesel 2.0

## Upgrade diesel version in Cargo.toml

In order to upgrade diesel to 2.0, we need to change dependencies in `Cargo.toml`.

```diff
diff --git a/rust/projects/diesel_2.0_example/Cargo.toml b/rust/projects/diesel_2.0_example/Cargo.toml
index 8f5a1aa..5164334 100644
--- a/rust/projects/diesel_2.0_example/Cargo.toml
+++ b/rust/projects/diesel_2.0_example/Cargo.toml
@@ -5,12 +5,13 @@ authors = ["lichuan <lichuan@mur>"]
 edition = "2018"

 [dependencies]
-diesel = { version = "1.4", features = [
+diesel = { version = "2.1.2", features = [
   "postgres",
   "serde_json",
   "chrono",
   "numeric",
   "64-column-tables",
+  "r2d2",
 ] }
 # r2d2 = "0.8"
 # r2d2-diesel = "1.0"
 dotenv = "0.14"
 # actix-web = "1.0.3"
 chrono = { version = "0.4.7", features = ["serde"] }
```

We change diesel version to `2.1.2` and remove `r2d2` and `r2d2-diesel` dependencies, as they are included in `diesel` crate, by specifying the `r2d2` feature in `diesel` crate `diesel = { version = "2.1.2", features = [ "r2d2" ] }`.

## Add mut to PgConnection and dao functions

Diesel now requires mutable access to the Connection to perform any database interaction. The following changes are required for all usages of any Connection type:

```diff
- let connection = PgConnection::establish_connection("…")?;
- let result = some_query.load(&connection)?;
+ let mut connection = PgConnection::establish_connection("…")?;
+ let result = some_query.load(&mut connection)?;
```

Here are the changes for our own code:

```diff
diff --git a/rust/projects/diesel_2.0_example/src/bin/contacts.rs b/rust/projects/diesel_2.0_example/src/bin/contacts.rs
index a74e29397..6efc8ef4f 100644
--- a/rust/projects/diesel_2.0_example/src/bin/contacts.rs
+++ b/rust/projects/diesel_2.0_example/src/bin/contacts.rs
@@ -16,11 +16,15 @@ fn test_contacts() {
         env::var("DATABASE_URL").unwrap_or(LOCAL_DATABASE_URL.into());
     let pool = db::init_pool(database_url);
     // https://github.com/sfackler/r2d2/issues/37
-    let connection = pool.get().unwrap();
+    let mut conn = pool.get().unwrap();

-    let conn: &PgConnection = &connection;
-    conn.execute("TRUNCATE TABLE contacts").unwrap();
-    conn.execute("alter sequence contacts_id_seq restart;").unwrap();
+    diesel::sql_query("TRUNCATE TABLE contacts").execute(&mut conn).unwrap();
+    diesel::sql_query("alter sequence contacts_id_seq restart;")
+        .execute(&mut conn)
+        .unwrap();
+
+    // conn.execute("TRUNCATE TABLE contacts").unwrap();
+    // conn.execute("alter sequence contacts_id_seq restart;").unwrap();

     let santas_address: serde_json::Value = serde_json::from_str(
         r#"{
@@ -61,7 +65,7 @@ fn test_contacts() {
         },
     ];

-    let contacts = create_contracts(&conn, &new_contacts).unwrap();
+    let contacts = create_contracts(&mut conn, &new_contacts).unwrap();
     println!("{:?}", contacts);

     // let inserted_address = insert_into(contacts)
@@ -75,9 +79,7 @@ fn get_contacts() {
     let database_url =
         env::var("DATABASE_URL").unwrap_or(LOCAL_DATABASE_URL.into());
     let pool = db::init_pool(database_url);
-    let connection = pool.get().unwrap();
-
-    let conn: &PgConnection = &connection;
+    let mut conn = pool.get().unwrap();

     let santas_address: serde_json::Value = serde_json::from_str(
         r#"{
@@ -86,12 +88,13 @@ fn get_contacts() {
     )
     .unwrap();

-    let contacts = get_contacts_by_address(&conn, &santas_address).unwrap();
+    let contacts = get_contacts_by_address(&mut conn, &santas_address).unwrap();
     println!("{:?}", contacts);

     let santas_address2: serde_json::Value = json!(true);

-    let contacts = get_contacts_by_address(&conn, &santas_address2).unwrap();
+    let contacts =
+        get_contacts_by_address(&mut conn, &santas_address2).unwrap();
     println!("{:?}", contacts);
 }

diff --git a/rust/projects/diesel_2.0_example/src/bin/select-limit-offset.rs b/rust/projects/diesel_2.0_example/src/bin/select-limit-offset.rs
index a9c583039..a69489b7a 100644
--- a/rust/projects/diesel_2.0_example/src/bin/select-limit-offset.rs
+++ b/rust/projects/diesel_2.0_example/src/bin/select-limit-offset.rs
@@ -10,13 +10,11 @@ fn select_limit_offset() {
         env::var("DATABASE_URL").unwrap_or(local_database_url.into());
     let pool = db::init_pool(database_url);
     // https://github.com/sfackler/r2d2/issues/37
-    let connection = pool.get().unwrap();
-
-    let _conn: &PgConnection = &connection;
+    let mut conn = pool.get().unwrap();

     let limit = 2;
     let offset = 2;
-    let all = get_select_limit_offset(&connection, limit, offset).unwrap();
+    let all = get_select_limit_offset(&mut conn, limit, offset).unwrap();
     println!("select : {:?} records", all);
 }

@@ -26,17 +24,15 @@ fn select_limit_offset_loop() {
         env::var("DATABASE_URL").unwrap_or(local_database_url.into());
     let pool = db::init_pool(database_url);
     // https://github.com/sfackler/r2d2/issues/37
-    let connection = pool.get().unwrap();
-
-    let _conn: &PgConnection = &connection;
+    let mut conn = pool.get().unwrap();

     let limit = 2;
     let mut offset = 0;
-    let all = get_select_limit_offset(&connection, limit, offset).unwrap();
+    let all = get_select_limit_offset(&mut conn, limit, offset).unwrap();
     println!("select : {:?} records", all);

     loop {
-        if let Ok(res) = get_select_limit_offset(&connection, limit, offset) {
+        if let Ok(res) = get_select_limit_offset(&mut conn, limit, offset) {
             if res.len() == 0 {
                 break;
             }
diff --git a/rust/projects/diesel_2.0_example/src/bin/test-connect-r2d2-pool-actix.rs b/rust/projects/diesel_2.0_example/src/bin/test-connect-r2d2-pool-actix.rs
index 1010a2380..44bce1eb4 100644
--- a/rust/projects/diesel_2.0_example/src/bin/test-connect-r2d2-pool-actix.rs
+++ b/rust/projects/diesel_2.0_example/src/bin/test-connect-r2d2-pool-actix.rs
@@ -2,15 +2,22 @@ extern crate diesel;

 use std::env;

-use actix_web::{web, App, HttpRequest, HttpServer, Responder};
+use actix_web::{web, App, HttpServer, Responder};
 use diesel_example::db;

-fn greet(req: HttpRequest) -> impl Responder {
-    let name = req.match_info().get("name").unwrap_or("World");
-    format!("Hello {}!", &name)
+// TODO: why compile error for actix-web 4?
+// fn greet(req: HttpRequest) -> impl Responder {
+//     let name = req.match_info().get("name").unwrap_or("World");
+//     format!("Hello {}!", &name)
+// }
+
+// #[get("/hello/{name}")]
+async fn greet(name: web::Path<String>) -> impl Responder {
+    format!("Hello {}!", name)
 }

-fn main() {
+#[actix_web::main]
+async fn main() -> std::io::Result<()> {
     let database_url = env::var("DATABASE_URL").expect("set DATABASE_URL");
     let pool = db::init_pool(database_url);
     // https://github.com/sfackler/r2d2/issues/37
@@ -26,5 +33,5 @@ fn main() {
     .bind("127.0.0.1:8000")
     .expect("Can not bind to port 8000")
     .run()
-    .unwrap();
+    .await
 }
diff --git a/rust/projects/diesel_2.0_example/src/bin/test-partial-inserts.rs b/rust/projects/diesel_2.0_example/src/bin/test-partial-inserts.rs
index 1819a9677..0fa4e2650 100644
--- a/rust/projects/diesel_2.0_example/src/bin/test-partial-inserts.rs
+++ b/rust/projects/diesel_2.0_example/src/bin/test-partial-inserts.rs
@@ -1,4 +1,3 @@
-
 use diesel_example::dao::partial_inserts::create_partial_inserts;
 use diesel_example::db;
 use diesel_example::model::partial_inserts::NewPartialInsert;
@@ -10,7 +9,7 @@ fn main() {
         env::var("DATABASE_URL").unwrap_or(local_database_url.into());
     let pool = db::init_pool(database_url);
     // https://github.com/sfackler/r2d2/issues/37
-    let connection = pool.get().unwrap();
+    let mut conn = pool.get().unwrap();

     let v = vec![
         NewPartialInsert { user_id: 5, name: Some("3".to_string()) },
@@ -20,7 +19,7 @@ fn main() {
     // 如果每次插入多个，其中一个报错（比如以上 user_id = 2），整体插入失败（5 不被插入）
     // Err(DatabaseError(UniqueViolation, "duplicate key value violates unique constraint \"ui_partial_inserts_user_id\""))

-    let r = create_partial_inserts(&connection, &v);
+    let r = create_partial_inserts(&mut conn, &v);

     println!("{:?}", r);
 }
diff --git a/rust/projects/diesel_2.0_example/src/bin/timestamp-with-zone.rs b/rust/projects/diesel_2.0_example/src/bin/timestamp-with-zone.rs
index e5a793222..1d3b3e72d 100644
--- a/rust/projects/diesel_2.0_example/src/bin/timestamp-with-zone.rs
+++ b/rust/projects/diesel_2.0_example/src/bin/timestamp-with-zone.rs
@@ -1,7 +1,5 @@
-use chrono::{DateTime};
-use diesel_example::dao::timestamp_with_zone::{
-    create_timestamp_with_zones,
-};
+use chrono::DateTime;
+use diesel_example::dao::timestamp_with_zone::create_timestamp_with_zones;
 use diesel_example::db;
 use diesel_example::model::timestamp_with_zone::NewTimestampWithZone;
 use std::env;
@@ -12,7 +10,7 @@ fn main() {
         env::var("DATABASE_URL").unwrap_or(local_database_url.into());
     let pool = db::init_pool(database_url);
     // https://github.com/sfackler/r2d2/issues/37
-    let connection = pool.get().unwrap();
+    let mut conn = pool.get().unwrap();

     /*
         let mut user_id = 123;
@@ -45,7 +43,7 @@ fn main() {
         NewTimestampWithZone { user_id: 109, created_at: dt.into() },
     ];

-    let inserted = create_timestamp_with_zones(&connection, &zones);
+    let inserted = create_timestamp_with_zones(&mut conn, &zones);
     match inserted {
         Ok(ref v) => {
             println!(
diff --git a/rust/projects/diesel_2.0_example/src/bin/updated-at.rs b/rust/projects/diesel_2.0_example/src/bin/updated-at.rs
index c536b6a9c..27c5a37da 100644
--- a/rust/projects/diesel_2.0_example/src/bin/updated-at.rs
+++ b/rust/projects/diesel_2.0_example/src/bin/updated-at.rs
@@ -10,7 +10,7 @@ fn main() {
         env::var("DATABASE_URL").unwrap_or(local_database_url.into());
     let pool = db::init_pool(database_url);
     // https://github.com/sfackler/r2d2/issues/37
-    let connection = pool.get().unwrap();
+    let mut conn = pool.get().unwrap();

     let user_id = 123;
     let n = NewWeiboFeedCrawlStatus {
@@ -20,11 +20,8 @@ fn main() {
         created_at: Local::now().naive_local(),
         updated_at: None,
     };
-    create_weibo_feed_crawl_status(&connection, &n);
+    create_weibo_feed_crawl_status(&mut conn, &n);
     update_total_page_count_and_next_page_index_by_user_id(
-        user_id,
-        &connection,
-        1,
-        1,
+        user_id, &mut conn, 1, 1,
     );
 }
diff --git a/rust/projects/diesel_2.0_example/src/bin/upsert-if-condition.rs b/rust/projects/diesel_2.0_example/src/bin/upsert-if-condition.rs
index ca7f19d77..807f74272 100644
--- a/rust/projects/diesel_2.0_example/src/bin/upsert-if-condition.rs
+++ b/rust/projects/diesel_2.0_example/src/bin/upsert-if-condition.rs
@@ -10,11 +10,15 @@ fn upsert_on_conflict_id() {
         env::var("DATABASE_URL").unwrap_or(local_database_url.into());
     let pool = db::init_pool(database_url);
     // https://github.com/sfackler/r2d2/issues/37
-    let connection = pool.get().unwrap();
+    let mut conn = pool.get().unwrap();

-    let conn: &PgConnection = &connection;
-    conn.execute("TRUNCATE TABLE upserts_if_condition").unwrap();
-    conn.execute("alter sequence upserts_if_condition_id_seq restart;")
+    // let conn: &mut PgConnection = &mut connection;
+    // conn.execute("TRUNCATE TABLE upserts_if_condition").unwrap();
+    // conn.execute("alter sequence upserts_if_condition_id_seq restart;")
+    //     .unwrap();
+
+    diesel::sql_query("TRUNCATE TABLE upserts_if_condition")
+        .execute(&mut conn)
         .unwrap();

     let u = NewUpsertIfCondition {
@@ -27,7 +31,7 @@ fn upsert_on_conflict_id() {
         next: Some(9),
         email: Some("e".into()),
     };
-    let inserted = create_upserts_if_condition_by_sql(&connection, &u).unwrap();
+    let inserted = create_upserts_if_condition_by_sql(&mut conn, &u).unwrap();
     println!("{} records inserted", inserted.len());
 }
```

## Derive attributes

You should add `#[diesel()]` when using derive attributes.

Below is an example of using `table_name` macro to define `NewTimestampWithZone` struct.

```rust
#[derive(Insertable, Debug)]
// NOTE: For diesel 1.4.x, we use `table_name` macro directly.
// #[table_name = "timestamp_with_zone"]
#[table_name = "timestamp_with_zone"]
pub struct NewTimestampWithZone {
    pub user_id: i64,
    pub created_at: DateTime<Utc>,
}
```

# Refs

Diesel 2.0 migration guide
https://diesel.rs/guides/migration_guide.html
