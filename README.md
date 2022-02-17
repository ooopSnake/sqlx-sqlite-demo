# Rust sqlx sqlite demo

```rust
#[tokio::main]
async fn main() -> anyhow::Result<()> {
    let pool = sqlite::SqlitePoolOptions::new()
        .max_connections(5)
        .connect("sqlite:local.db?mode=rwc").await?;
    sqlx::query(
        r#"CREATE TABLE IF NOT EXISTS t_example  (
                    id INTEGER PRIMARY KEY,
                    name TEXT,
                    age INTEGER
               )
            "#).execute(&pool).await.expect("create table");

    let mut tx = pool.begin().await.expect("begin tx");
    for idx in 1..=2 {
        sqlx::query(r#"INSERT INTO t_example(id,name,age) VALUES(?,?,?)
                           ON CONFLICT(id) DO UPDATE
                           SET name=excluded.name,
                                age=excluded.age"#)
            .bind(idx)
            .bind(format!("foo_{}", idx))
            .bind(idx * 100)
            .execute(&mut tx).await.expect(&format!("insert idx:{} in tx", idx));
    }
    tx.commit().await.expect("tx commit");
    let example_stream =
        sqlx::query_as::<_, Example>(r#"SELECT id,name,age FROM t_example WHERE 1=1"#)
            .fetch(&pool);
    example_stream.try_for_each(|it| {
        dbg!(it);
        futures::future::ready(Ok(()))
    }).await.expect("for each query result");
    Ok(())
}
```