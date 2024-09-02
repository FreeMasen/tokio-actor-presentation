# Tokio Mutex

$web-only$

To start, let's just swap out the Mutex and see what happens.

$web-only-end$

```rust
use serde::Serialize;
use serde_json::Value;
// We've dropped `sync::Mutex` here
use std::{
    collections::BTreeMap,
    sync::Arc,
    time::{Duration, Instant},
};
// This is our new import
use tokio::sync::Mutex;

async fn main() {
    let db = Arc::new(Mutex::new(Database::default()));
    tokio::task::spawn({
        let db = db.clone();
        async move {
            loop {
                let guard = db.lock().await;
                let value = guard.get("my-key");
                tokio::time::sleep(Duration::from_secs(1)).await;
                dbg!(value);
            }
        }
    });

    tokio::task::spawn(async move {
        let start = Instant::now();
        loop {
            db.lock()
                .await
                .insert("my-key", &start.elapsed().as_millis());
            tokio::time::sleep(Duration::from_millis(350)).await;
        }
    })
    .await
    .unwrap();
}
```

$web-only$

That worked! But what exactly is happening? Let's add some print statements to see if we can figure
out the ordering of operations.

$web-only-end$

```rust
async fn main() {
    let db = Arc::new(Mutex::new(Database::default()));
    tokio::task::spawn({
        let db = db.clone();
        async move {
            loop {
                println!("->read-guard");
                let guard = db.lock().await;
                let value = guard.get("my-key");
                println!("->read-sleep");
                tokio::time::sleep(Duration::from_secs(1)).await;
                println!("<-read-sleep");
                dbg!(value);
                println!("<-read-guard")
            }
        }
    });

    tokio::task::spawn(async move {
        let start = Instant::now();
        loop {
            println!("->write-guard");
            db.lock()
                .await
                .insert("my-key", &start.elapsed().as_millis());
            println!("<-write-guard");
            tokio::time::sleep(Duration::from_millis(350)).await;
        }
    })
    .await
    .unwrap();
}
```

$web-only$

When we run this, we get the following output.

$web-only-end$

```text
->read-guard
->read-sleep
->write-guard
<-read-sleep
[src/main.rs:55:17] value = None
<-read-guard
->read-guard
<-write-guard
->read-sleep
->write-guard
<-read-sleep
[src/main.rs:55:17] value = Some(
    Number(1002),
)
<-read-guard
->read-guard
<-write-guard
```

$web-only$

So, looking over our logs it seems like our `get` task locks our `Mutex` and then immeadly sleeps,
`tokio` then selects our `insert` task which attempts to lock our `Mutex` however since our `get`
task already has the `lock` it will yield back to `tokio`. Once our `sleep` finishes it prints out
the value and drops the guard and then immedatly tries to lock the map again. At this point because
the `insert` task called `lock` before our second `lock` in the get task, `tokio` will let our
`insert` task start to make progress which inserts our first value.

$web-only-end$
