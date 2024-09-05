# Overview

$web-only$

When developing applications with tokio one of the larger challenges is how to manage sharing access
to resources that either need to be mutated or might have some internal state that needs to be
protected. There are a few options for managing this synchronization. To go over this problem and
what solutions are available we are going to use an somewhat contrived example of a rudimentary
in-memory database. The basic structure will look something like the following.

$web-only-end$

```rust
use std::collections::BTreeMap;
use serde_json::Value;
use serde::Serialize;

/// A key value store where all keys are Strings
/// and any associated values backed by json values
#[derive(Debug, Default)]
pub struct Database {
    /// The mapping of keys to values
    map: BTreeMap<String, Value>,
}

impl Database {
    /// Get a reference to the values paired with the key if one exists
    pub fn get(&self, key: impl AsRef<str>) -> Option<&Value> {
        self.map.get(key.as_ref())
    }

    /// Insert a new value into the map, overwriting any previous value of one was present
    pub fn insert(&mut self, key: impl ToString, value: &impl Serialize) {
        let value = serde_json::to_value(value).unwrap();
        self.map.insert(key.to_string(), value);
    }
}
```

$web-only$

Our `Database`, is a simple wrapper around `BTreeMap` that isn't all that interesting yet but it does
impose an issue if we tried to use this as a shared resource across tasks.

$web-only-end$

```rust
use std::time::{Duration, Instant};

#[tokio::main]
async fn main() {
    let mut db = Database::default();
    tokio::task::spawn(async {
        loop {
            dbg!(db.get("my-key"));
            tokio::time::sleep(Duration::from_secs(1)).await;
        }
    });

    tokio::task::spawn(async {
        let start = Instant::now();
        loop {
            db.insert("my-key", start.elapsed().as_millis())
        }
    });
}
```

$web-only$

The above will error with the following message

$web-only-end$

```shell
error[E0502]: cannot borrow `db` as mutable because it is also borrowed as immutable
  --> src/main.rs:38:24
   |
31 |        tokio::task::spawn(async {
   |  ______-__________________-
   | | _____|
   | ||
32 | ||         loop {
33 | ||             dbg!(db.get("my-key"));
   | ||                  -- first borrow occurs due to use of `db` in coroutine
34 | ||             tokio::time::sleep(Duration::from_secs(1)).await;
35 | ||         }
36 | ||     });
   | ||_____-- argument requires that `db` is borrowed for `'static`
   | |______|
   |        immutable borrow occurs here
37 |
38 |        tokio::task::spawn(async {
   |   ________________________^
39 |  |         let start = Instant::now();
40 |  |         loop {
41 |  |             db.insert("my-key", &start.elapsed().as_millis())
   |  |             -- second borrow occurs due to use of `db` in coroutine
42 |  |         }
43 |  |     });
   |  |_____^ mutable borrow
```

$web-only$

Because the task that calls `get` and the task that calls `insert` both need a reference to
this shared database the compiler is complaining that we need somehting to synchronize the insert
and get operations.

In the next chapter we are going to look at how we might achieve this with the standard libaray's
primitives.

$web-only-end$
