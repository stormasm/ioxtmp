
rg "scan\("

* iox_query/src/provider.rs line 747

```rust
fn build_deduplicate_plan_for_overlapped_chunks(
    ctx: IOxSessionContext,
    table_name: Arc<str>,
    output_schema: Arc<Schema>,
    chunks: Vec<Arc<dyn QueryChunk>>, // These chunks are identified overlapped
    predicate: Predicate,
    sort_key: &SortKey,
) -> Result<Arc<dyn ExecutionPlan>> {
    // Note that we may need to sort/deduplicate based on tag
    // columns which do not appear in the output

    // We need to sort chunks before creating the execution plan. For that, the chunk order is used. Since the order
    // only sorts overlapping chunks, we also use the chunk ID for deterministic outputs.
    let chunks = {
        let mut chunks = chunks;
        chunks.sort_unstable_by_key(|c| (c.order(), c.id()));
        chunks
    };
```

* compactor/src/query.rs line 259

```rust
// Order of the chunk so they can be deduplicate correctly
fn order(&self) -> ChunkOrder {
    let seq_num = self.min_sequence_number.get();
    let seq_num = u32::try_from(seq_num)
        .expect("Sequence number should have been converted to chunk order successfully");
    ChunkOrder::new(seq_num)
        .expect("Sequence number should have been converted to chunk order successfully")
}
```




```rust
RUST_BACKTRACE=1 ./target/debug/influxdb_iox
```

```rust
2022-06-01T03:54:17.599052Z ERROR panic_logging: Thread panic panic_info=panicked at 'Sequence number should have been converted to chunk order successfully', compactor/src/query.rs:259:14
thread 'tokio-runtime-worker' panicked at 'Sequence number should have been converted to chunk order successfully', compactor/src/query.rs:259:14
note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
2022-06-01T03:54:17.616258Z ERROR panic_logging: Thread panic panic_info=panicked at 'Sequence number should have been converted to chunk order successfully', compactor/src/query.rs:259:14
thread 'tokio-runtime-worker' panicked at 'Sequence number should have been converted to chunk order successfully', compactor/src/query.rs:259:14
```
