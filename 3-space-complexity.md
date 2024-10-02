# The Space Complexity of Vector Indexes in LibSQL

In this post, I'm continuing my exploration into vector search and graph clustering at [Kin](https://mykin.ai). Kin is a personal AI that helps you navigate your work-life, available on both iOS and Android.

In my previous posts, I outlined our challenges and journey with libSQL: 

1. [Personal Knowledge Graphs in AI RAG-powered Applications with libSQL](https://turso.tech/blog/personal-knowledge-graphs-in-ai-rag-powered-applications-with-libsql)
2. [Building Vector Search and Personal Knowledge Graphs on Mobile with libSQL and React Native](https://turso.tech/blog/graph-rag-pipelines-in-libsql)

Let's take a look at the internals of the libSQL vector search. In short:

> LibSQL enhances existing data types by incorporating the LM-DiskANN algorithm for vector indexing directly into your database.

You can find the full paper here: [FreshDiskANN Paper](https://cse.unl.edu/~yu/homepage/publications/paper/2023.LM-DiskANN-Low%20Memory%20Footprint%20in%20Disk-Native%20Dynamic%20Graph-Based%20ANN%20Indexing.pdf).

![Graph Illustration](https://miro.medium.com/v2/resize:fit:1400/1*a3JeuEIDj4djXgql0_ybUw.png)

FreshDiskANN is a graph-based family of Approximate Nearest Neighbor (ANN) algorithms. It's optimized for scenarios where the dataset is too large to fit in memory. While slower than in-memory approaches due to disk access latency, FreshDiskANN minimizes these delays as much as possible.

It also reduces I/O operations by preloading a neighbor graph when needed. This results in a small memory footprint, optimized I/O, and faster performance, all critical factors for mobile apps.

Of course, this optimization comes at a cost!

## Space Requirements

To make the algorithm effective, we store a high-context graph and its neighbor graphs on disk, which requires significant storage space. I was surprised by the increase in database size after migrating to LibSQL.

Initially, we had 3.5 MB of data with vectors. After migrating, the size increased to 50 MB. While this is manageable, I decided to investigate further.

Each index has a special shadow system table that stores graphs. You can query the page size of these system tables with this SQL query:

```sql
SELECT name, SUM("pgsize") FROM "dbstat" WHERE name LIKE 'idx%shadow' GROUP BY name;
```

In our case, the vector index for graph triples takes up 19 MB. 

### What Can We Do About It?

By default, we create a vector index with the following command:

```sql
CREATE INDEX idx_edge_vectorTriple_default ON edge (libsql_vector_idx(vectorTriple));
```

This uses 32-bit float values for neighbor vectors and calculates the maximum neighbor parameter as `3 * sqrt(vector length)`. For us, with a vector length of 512, that’s `sqrt(512) * 3` = around 70 neighbors.

There are a few parameters we can adjust to make the index more space-efficient:

- Vector compression
- Maximum number of neighbors

Vector compression controls how we store neighbor vectors in the graph used for searching. We have several options:

- Float 16
- Float 8
- Int 8
- 1-bit

While 1-bit compression is the most space-efficient, it requires a 1-bit-aware embedding model. We opted for float8 compression, which works well with our existing model.

Max neighbors is a tricky parameter, as it impacts space, speed, and the accuracy of the search. Since ANN doesn't guarantee perfect results, there is a trade-off between speed and accuracy. However, graph-based ANN provides a good compromise.

I was confident about the compression setting but wanted to experiment with the max neighbors parameter. So, I created a few alternative indexes:

```sql
CREATE INDEX idx_edge_vectorTriple_default ON edge (libsql_vector_idx(vectorTriple));
CREATE INDEX idx_edge_vectorTriple_compressed ON edge (libsql_vector_idx(vectorTriple, 'compress_neighbors=float8'));
CREATE INDEX idx_edge_vectorTriple_compressed_50 ON edge (libsql_vector_idx(vectorTriple, 'compress_neighbors=float8', 'max_neighbors=50'));
CREATE INDEX idx_edge_vectorTriple_compressed_20 ON edge (libsql_vector_idx(vectorTriple, 'compress_neighbors=float8', 'max_neighbors=20'));
```

Then, I ran a size estimation query. Just using compression showed good results with a 3x reduction: 7581696 / 2404352. Reducing the neighbors to 20 while using compression gave an 8x improvement: 19705856 / 2404352!

So, the index became 8 times smaller!

My main concern was search quality. How does using 20 neighbors compare to 70 neighbors?

### Testing Search Quality

I designed a small test to compare vector search results. I selected the most connected nodes and used them for vector search queries.

```sql
SELECT n.id, n.displayLabel, vector_extract(n.vectorLabel) vec, 
       (IFNULL(edge_from.edge_count, 0) + IFNULL(edge_to.edge_count, 0)) AS total_edges
FROM node n
LEFT JOIN (
    SELECT fromId, COUNT(*) AS edge_count
    FROM edge
    GROUP BY fromId
) AS edge_from ON n.id = edge_from.fromId
LEFT JOIN (
    SELECT toId, COUNT(*) AS edge_count
    FROM edge
    GROUP BY toId
) AS edge_to ON n.id = edge_to.toId
ORDER BY total_edges DESC
LIMIT ?;
```

I stored each vector search run for every query and index combination in a table.

```typescript
const edgeIndexes = ['idx_edge_vectorTriple_default', 'idx_edge_vectorTriple_compressed', 'idx_edge_vectorTriple_compressed_50', 'idx_edge_vectorTriple_compressed_20'];

for (let tripleQuery of Object.values(nodeQueries)) {
    for (let edgeIndex of edgeIndexes) {
        const query = `SELECT e.id 
                       FROM vector_top_k('${edgeIndex}', vector('[${tripleQuery.vector}]'), ${topN}) AS i
                       INNER JOIN edge e ON e.rowid = i.id`;
        const statement = db.prepare(query);
        statement.raw(true);
        const data = statement.all();
        for (let [eid] of data) {
            const insertStatement = db.prepare('INSERT INTO vsr_edge_run (id, runId, queryId, indexName, entityId) VALUES(?, ?, ?, ?, ?);');
            insertStatement.run([seqId++, runId, tripleQuery.id, edgeIndex, eid]);
        }
    }
}
```

Now, we can compare the search result order with the default index.

```sql
WITH default_ids AS (
    SELECT queryId, GROUP_CONCAT(entityId ORDER BY entityId) AS ids
    FROM vsr_edge_run
    WHERE indexName = 'idx_edge_vectorTriple_default'
    GROUP BY queryId
)

SELECT q.value, r.indexName, r.ids
FROM (
    SELECT queryId, indexName, GROUP_CONCAT(entityId ORDER BY entityId) AS ids
    FROM vsr_edge_run
    WHERE indexName != 'idx_edge_vectorTriple_default'
    GROUP BY queryId, indexName
) r
INNER JOIN default_ids d ON r.queryId = d.queryId
INNER JOIN vsr_triples_queries q ON q.id = r.queryId
WHERE r.ids != d.ids;
```

My test dataset had fewer than 1,000 nodes, and the triple vectors were fairly distributed without forming large clusters. The search results were identical, even with a smaller number of neighbors.

For datasets with more than 10,000 triples, the difference in search results would likely become more noticeable, and at some point, you would see a deviation.

We work primarily in the Personal Knowledge Graph domain, where the focus is on middle-sized datasets that provide value to the user. It's more about quality than quantity. For me, keeping the index small while maintaining search speed is an excellent technical compromise.

We're super excited about the performance of libSQL, and with these optimizations, we can now index users data efficiently, without running out of space.

## Sounds interesting?
If this sounds interesting and you’d like to work with us on these exciting projects, please reach out at jobs@mykin.ai!
