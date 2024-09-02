# Using std

$web-only$

There are 2 types that we can utilize in the standard library's `sync` module, `Arc` and `Mutex`.

`Arc` moves our `Database` onto the heap so it can be accessed across each of our tasks via a
`clone`. `Mutex` provides our synchronization, only one task will be allowed to `lock` the `Mutex`
at any given time.

$web-only-end$

```rust
#[tokio::main]
async fn main() {
    // wrapping the database in an Arc+Mutex will make it shareable across our
    // tasks.
    let db = Arc::new(Mutex::new(Database::default()));
    tokio::task::spawn({
        // Clone the database reference to `move` into our async block
        let db = db.clone();
        async move {
            loop {
                dbg!(db.lock().unwrap().get("my-key"));
                tokio::time::sleep(Duration::from_secs(1)).await;
            }
        }
    });

    tokio::task::spawn(async move {
        let start = Instant::now();
        loop {
            db.lock()
                .unwrap()
                .insert("my-key", &start.elapsed().as_millis());
            tokio::time::sleep(Duration::from_millis(350)).await;
        }
    // We need to await here to make sure we don't exit our application early
    }).await;
}
```

$web-only$

This works as expected (printing out the duration since start with about a 1 second delay between
updates). However, immedatly dropping the the lock in both of our tasks isn't much of a real-world
simulation. Instead, what happens when we add a short dealy before printing in the `get` task?

$web-only-end$

```rust
tokio::task::spawn({
    let db = db.clone();
    async move {
        loop {
            let guard = db.lock().unwrap();
            let value = guard.get("my-key");
            tokio::time::sleep(Duration::from_secs(1)).await;
            dbg!(value);
        }
    }
});
```

$web-only$

This seems like it would be just the same as our last version but cargo complains.

$web-only-end$

```shellsession
error: future cannot be sent between threads safely
   --> src/main.rs:31:5
    |
31  | /     tokio::task::spawn({
32  | |         let db = db.clone();
33  | |         async move {
34  | |             loop {
...   |
40  | |         }
41  | |     });
    | |______^ future created by async block is not `Send`
```

$web-only$

This is because we are `await`ing in our `get` task before dropping the `MutexGuard` returned by
`db.lock()`. Let's try and unroll what the order of operations here would be from when we first call
`lock`.

- `db.lock()` reserves the `Mutex` for our `get` task
- `guard.get` returns a reference to a value in our database
- `tokio::time::sleep` pauses this tasks allowing our `insert` task to run
- `db.lock()` waits for all other `MutexGuard`s to be dropped

And now we've achieved a deadlock since the `insert` task cannot procede until the `get` task
makes progress but the `get` task has yielded control to `tokio` so there is no gaurantee that it
will be resumed. For example if we were using a single threaded runtime the `insert` task would
never yield control back to `tokio` meaning our application will always get stuck.

There are a few ways to deal with this, one would be to switch from `std::sync::Mutex` to
`tokio::sync::Mutex`.

$web-only-end$
