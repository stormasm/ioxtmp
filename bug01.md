
```rust
rg "scan\("
```

* iox_query/src/provider.rs

```rust
#[async_trait]
impl TableProvider for ChunkTableProvider {
    fn as_any(&self) -> &dyn std::any::Any {
        self
    }

    /// Schema with all available columns across all chunks
    fn schema(&self) -> ArrowSchemaRef {
        self.arrow_schema()
    }

    async fn scan(
        &self,
        projection: &Option<Vec<usize>>,
        filters: &[Expr],
        _limit: Option<usize>,
    ) -> std::result::Result<Arc<dyn ExecutionPlan>, DataFusionError> {
        trace!("Create a scan node for ChunkTableProvider");

        // Note that `filters` don't actually need to be evaluated in
        // the scan for the plans to be correct, they are an extra
        // optimization for providers which can offer them
        let predicate = PredicateBuilder::default()
            .add_pushdown_exprs(filters)
            .build();

        // Now we have a second attempt to prune out chunks based on
        // metadata using the pushed down predicate (e.g. in SQL).
        let chunks: Vec<Arc<dyn QueryChunk>> = self.chunks.to_vec();
        let num_initial_chunks = chunks.len();
        let chunks = self.chunk_pruner.prune_chunks(
            self.table_name(),
            self.iox_schema(),
            chunks,
            &predicate,
        );
        debug!(%predicate, num_initial_chunks, num_final_chunks=chunks.len(), "pruned with pushed down predicates");

        // Figure out the schema of the requested output
        let scan_schema = match projection {
            Some(indices) => Arc::new(self.iox_schema.select_by_indices(indices)),
            None => Arc::clone(&self.iox_schema),
        };

        // This debug shows the self.arrow_schema() includes all columns in all chunks
        // which means the schema of all chunks are merged before invoking this scan
        debug!(schema=?self.arrow_schema(), "All chunks schema");
        // However, the schema of each chunk is still in its original form which does not
        // include the merged columns of other chunks. The code below (put in comments on purpose) proves it
        // for chunk in chunks.clone() {
        //     trace!("Schema of chunk {}: {:#?}", chunk.id(), chunk.schema());
        // }

        let mut deduplicate =
            Deduplicater::new().with_execution_context(self.ctx.child_ctx("deduplicator"));
        let plan = deduplicate.build_scan_plan(
            Arc::clone(&self.table_name),
            scan_schema,
            chunks,
            predicate,
            self.sort_key.clone(),
        )?;

        Ok(plan)
    }
```

```rust
fn build_scan_plan(
```

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
