# The space complexity of vector indexes in LibSQL
Hey, so I continue my adventure in vector search and Graph clustering at [kin](mykin.ai)
Last time, I described our challenge and journey to LibSQL
[Personal Knowledge Graphs in AI RAG-powered Applications with libSQL](./graphrag-pipelines.md)

So to summarize

LibSQL builds on existing data types and directly adds effective LM-DiskANN algorithm-based vector indexes to your database.

The full paper can be found here: [paper FreshDisckANN](https://cse.unl.edu/~yu/homepage/publications/paper/2023.LM-DiskANN-Low%20Memory%20Footprint%20in%20Disk-Native%20Dynamic%20Graph-Based%20ANN%20Indexing.pdf)

![](https://miro.medium.com/v2/resize:fit:1400/1*a3JeuEIDj4djXgql0_ybUw.png)

FreshDiskANN is a graph-based family of ANN algorithms.

It is optimized for scenarios where the dataset is too large to fit in memory. While slower than in-memory approaches due to the inherent latency of disk access, FreshDiskANN is designed to minimize these delays as much as possible.

Also, FreshDiskANN tries to minimize io operation and preload a neighbor graph when needed.

As a result, you have a small memory footprint, optimized IO, and speed, all of which are critical for mobile apps.

All this came with a price !!!

## Space

To make the algorithm effective, we keep a high-context graph and the neighbors' graph colocated on a disk, which requires space. I was surprised when I looked at my database size after migrating to LibSQL.

So, from the original 3.5 MB of data with vectors. I got 50 MB. Still, it is pretty manageable, but I have started my investigations.

First, every index has a special shadow system table that stores graphs.

We could query the page size of this system tables in pages.

```sql
SELECT name , SUM("pgsize") FROM "dbstat" WHERE name like 'idx%shadow' group by name ;

```

Our vector index for graph triples takes 19 Mb

What could we do about it?

## Optimization

By default, we create a vector as

```
CREATE INDEX idx_edge_vectorTriple_default  ON edge  (libsql_vector_idx(vectorTriple));

```

it will use Float 32 values for neighbor vectors and calculate the max neighbor parameter as 3 \* sqrt (vector length) . So for us, it will be sqrt(512) \*3 = around 70.

We have a few parameters to tune for a space effective index

-   vector compression
-   max neighbors

Vector compression controls a way how we store a vector of neighbors in a graph that will be used for a search

We have a lot of options

-   float 16
-   float 8
-   int 8
-   1 bit

1-bit compression is the most space-effective but requires a 1-bit-aware embedding model. We decided to go with float8 compression, which works well with our existing model.

Max neighbors are quite tricky parameters. They are affected a lot. They influence not only space but also speed and, more importantly, the accuracy of search.

ANN does not guarantee you will get all values, so close-to-ideal results on a huge data set may not be ideal. Graph ANN is a good compromise on speed and accuracy.

I was pretty sure about compression, but I was keen to discover the max parameter.

So first, I create a few alternative indexes

```sql
CREATE INDEX idx_edge_vectorTriple_default  ON edge  (libsql_vector_idx(vectorTriple));
CREATE INDEX idx_edge_vectorTriple_compressed  ON edge  (libsql_vector_idx(vectorTriple, 'compress_neighbors=float8'));
CREATE INDEX idx_edge_vectorTriple_compressed_50  ON edge  (libsql_vector_idx(vectorTriple, 'compress_neighbors=float8', 'max_neighbors=50'));
CREATE INDEX idx_edge_vectorTriple_compressed_20  ON edge  (libsql_vector_idx(vectorTriple, 'compress_neighbors=float8', 'max_neighbors=20'));

```

After that, I run my size estimation query

![](https://miro.medium.com/v2/resize:fit:1400/1*SQ8cY1WQjc-ruCnNqObwJw.png)

Just a compression one gives good results 7581696 / 2404352 = 3x

As you can see, 20 neighbors with a compression 19705856 / 2404352 give 8x improvement !!

So the index gets 8 times smaller!!!

My main question was about search quality. How good is 20 neighbors in comparison to 70

I decided to design a small test to get close vector search results. I took the most connected nodes and used them for vector search queries.

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
            `)
```

Now, I store every vector search run for every query and index combination in a table.

```typescript

const edgeIndexes = ['idx_edge_vectorTriple_default', 'idx_edge_vectorTriple_compressed', 'idx_edge_vectorTriple_compressed_50','idx_edge_vectorTriple_compressed_20']
    
    for (let tripleQuery of Object.values(nodeQueries)) {
        for (let edgeIndex of edgeIndexes) {

            const query = `select e.id id
                            from vector_top_k('${edgeIndex}', vector('[${tripleQuery.vector}]'), ${topN}) as i
                            inner join edge e on e.rowid = i.id
                            `
            const statement = db.prepare(query)
            statement.raw(true)
            const data = statement.all()
            for (let [eid] of data) {
                //  (id int primary key , runId int , queryId int , indexName text, entityId int , entityOrder int)
                const insertStatement = db.prepare('insert into vsr_edge_run (id, runId, queryId , indexName,entityId) values(?, ?, ?,?,?);')
                insertStatement.run([seqId++, runId, tripleQuery.id, edgeIndex, eid])
            }   
        }
    }
```

Now, when we have all run results, we compare the search results' order with a default index.

```sql
 WITH default_ids AS (
            SELECT 
                queryId, 
                GROUP_CONCAT(entityId ORDER BY entityId) AS ids 
            FROM 
                vsr_edge_run 
            WHERE 
                indexName = 'idx_edge_vectorTriple_default' 
            GROUP BY 
                queryId
        )

        SELECT 
            q.value,
            r.indexName,
            r.ids
        FROM 
            (
                SELECT 
                    queryId, 
                    indexName, 
                    GROUP_CONCAT(entityId ORDER BY entityId) AS ids 
                FROM 
                    vsr_edge_run 
                WHERE 
                    indexName != 'idx_edge_vectorTriple_default' 
                GROUP BY 
                    queryId, 
                    indexName
            ) r
        INNER JOIN 
            default_ids d 
            ON r.queryId = d.queryId
        INNER JOIN 
            vsr_triples_queries q 
            ON q.id = r.queryId
        WHERE 
            r.ids != d.ids;
```

My test dataset was a graph of less than 1000 nodes. In general, my triple vectors are quite distributed and do not form big clusters.

As a result, all this gives identical search results for much smaller neighbors.

For datasets with 10,000+ triples, the difference could be very visible, and at some point, you will see a deviation in search results.

We work more in a Personal Knowledge Graph domain where the data set focuses on middle-size data that have value to a user. It is more about quality than quantity. For me, keeping the index small and searching a bit faster is an excellent technical compromise.

Now we have more space and battery for your ideas !!

