# Personal Knowledge Graphs in AI RAG-powered Applications with libSQL

I’ve spent a lot of time working on a privacy-first personal AI assistant at [mykin.ai](mykin.ai).

Our app is local-first and focused on the users' sovereign data ownership. A key challenge we faced was finding the right way to store that data securely.

## Device-friendly

I partially covered several options for embeddable and device-friendly databases in my previous article,
[Embeddable and Portable Storage for AI-Powered Apps and Personal Knowledge Graphs](https://ai.plainenglish.io/embeddable-and-portable-storage-for-ai-powered-apps-and-personal-knowledge-graphs-e46b0564e73c)

AI-powered apps, particularly in the semantic memory component, place several expectations on database capabilities:

- General queries on structured data (e.g., messages, conversations, settings, etc.)
- Vector search and similarity search capabilities for RAG pipelines and various LLM and ML-powered flows
- Graph and graph search capabilities (for ML and semantic memory)

Since we work on mobile, we also require a few technical capabilities:

- Embeddable with strong support for mobile bindings
- Single-file database for simplified backups
- Portability
- Battery efficiency
- Fast, non-blocking I/O wherever possible
- Strong community support
- Reliability

Vector and graph capabilities for embeddable databases are relatively new challenges for modern databases. A key competitor was PostgreSQL, with extensions like
[Apache AGE](https://age.apache.org/?source=post_page-----50b0e7aa10c4--------------------------------) and [pgvector](https://github.com/pgvector/pgvector?source=post_page-----50b0e7aa10c4--------------------------------)

So, we need a similar embeddable setup.

## Graphs

Currently, there are practically no graph-oriented databases that offer portable and embeddable capabilities with a mobile or small-device-friendly setup.

I’ve written a few articles that model and demonstrate how to use relational databases for graph and hypergraph capabilities. It’s a broad topic with many exciting research avenues. You can find more about it in my article:
[Personal Knowledge Graphs. Semantic Entity Persistence in Relational Model](https://blog.stackademic.com/personal-knowledge-graphs-semantic-entity-persistence-in-relational-model-d5692bb8e8bb)

## Vectors

There is a wide variety of vector databases and libraries on the market. Some libraries, like [faiss](https://github.com/facebookresearch/faiss?source=post_page-----50b0e7aa10c4--------------------------------), are top-performing and even offer the ability to persist vectors to a file. So, it was a good start, but…

It’s a big challenge to keep heterogeneous data storage in sync. The synchronization process consumes time, battery, and CPU resources, affecting the main thread of the app and making it less user-friendly.

For me, it was clear — we needed something integrated into the database.

After weeks of research and prototyping, we settled on the SQLite ecosystem. SQLite has been the most popular and reliable database for mobile devices for decades.

But what about vectors?

SQLite has an extendable architecture that allows native modules to expand its capabilities. I found a project that brings faiss to SQLite:
[sqlite-vss](https://github.com/asg017/sqlite-vss?source=post_page-----50b0e7aa10c4--------------------------------)
Unfortunately, it wasn’t reliable and had a few major bugs, so I almost gave up.

Luckily, we found a better solution: [LibSQL](https://turso.tech/libsql?source=post_page-----50b0e7aa10c4--------------------------------)

[Libsql](https://github.com/tursodatabase/libsql?tab=readme-ov-file&source=post_page-----50b0e7aa10c4--------------------------------) is open to contribution. It’s an open-source replacement for SQLite that brings many features and performance optimizations to the table. Some of these features deserve a separate article, like:

- Table alterations that make migrations easier
- WebAssembly-defined functions!
- Virtual WAL interface
- And much more.

But the most critical feature is that it extends SQLite with vector search capabilities. 
[AI & Embeddings](https://docs.turso.tech/features/ai-and-embeddings?source=post_page-----50b0e7aa10c4--------------------------------)

It’s built smartly with minimal database changes, making migration easy while maintaining compatibility with SQLite.

There is no dedicated vector type; instead, it's aliased on top of BLOB.

```sql
CREATE TABLE node (
          id varchar(36) primary key not null,
          label varchar not null default '',
          vectorLabel F32_BLOB(512) ,
          displayLabel varchar not null default '',
          createdAt real,
          updatedAt real
         );
CREATE TABLE edge (
          id varchar(36) primary key not null,
          fromId varchar(36) not null default '',
          toId varchar(36) not null default '',
          label varchar not null default '',
          displayLabel varchar not null default '',
          vectorTriple F32_BLOB(512) ,
          createdAt real,
          updatedAt real,
         );
```

**F32\_BLOB(512)** defines metadata about the vector: it specifies the float 32-bit value type and the array’s dimension.

This type is more of an alias for BLOB but allows the database to validate the vector's shape and data types.

Now, we can use vector search for clustering graphs and applying it in LLM-powered pipelines.

On the edge, we store metadata about the triple:

- displayLabel — a non-normalized description of the edge label
- label — a normalized label
- vectorTriple — the most interesting part. We normalize node labels and the edge label, concatenate them, and create an embedding that allows edges to be searchable via vector search.

Our vectorTriple column adds vector search capability to our personal knowledge graph.

To insert data, we use a vector function to get an embedding as:

- A float32 array
- A BLOB of serialized float32 array
- A string representation like ‘\[0.5432635, 0.3333 ….\]’

```js
insert into edge (rowid, id, fromId, toId , label, vectorTriple, displayLabel, createdAt, updatedAt)
  values (? , ?, ? , ? , ? , vector(${this._store.toVector(
          await this.embeddingsService.embedDocument(`${fromNode.label} ${normalizedLabel} ${toNode.label}`)
        )}) , ? , ?, ?);
```

With the **vector\_distance\_cos** function, we can calculate distances and run queries.

```sql
select  e.id, e.label, vector_distance_cos(e.vectorTriple , ${vector}) distance 
from egde e 
where distance < 0.15
```

This is fantastic but very slow and CPU-intensive because it requires scanning the entire table. So, we need a vector index.

Luckily, we can create an index for embedding columns!

```
CREATE INDEX idx_edge_vectorTriple ON edge (libsql_vector_idx(vectorTriple));
```

Now, instead of scanning the full table, we can search in the index:

```sql
select  e.id, e.label, vector_distance_cos(e.vectorTriple , vector('[0.32323, 0.525, ....]')) distance
    from vector_top_k('idx_edge_vectorTriple', vector('[0.32323, 0.525, ....]'), ${_top}) as i
    inner join edge as e on i.id = e.rowid
    inner join node as fn on e.fromId = fn.id
    inner join node as tn on e.toId = tn.id
    where distance <= 0.15`
    order by distance
    limit 20;
```

Let's break it down:

```
vector_top_k('idx_edge_vectorTriple', '[0.32323, 0.525, ....]', 20) as i
```

This retrieves **rowIDs** of similar vectors from the **idx\_edge\_vectorTriple** index, based on the **vectorTriple** column.

Be cautious. By default, it uses rowid. To combine the results of the vector search with other data in your DB, you need a simple join statement:

```
inner join edge as e on i.id = e.rowid
```

The index doesn't return distances, so you still need to calculate them manually on a much smaller dataset:

```sql
select  e.id, e.label, vector_distance_cos(e.vectorTriple , vector('[0.32323, 0.525, ....]')) distance
```

The vector function is a smart move that converts a string like ‘\[1,32,2, …. \]’ into a BLOB type stored in the database.

Now, you can filter distances to find closely related triples:

```
where distance <= 0.15
```

Now, we have vector search capabilities on top of our personal knowledge graph!

## Conclusion

Personal Knowledge Graphs can be modeled in a relational model, and since they are usually middle-sized, they don’t cause significant performance issues with SQL queries. With LibSQL, we now have native, low-level support for vectors. This allows us to build graph clustering and RAG pipelines on user devices if needed. The feature is still under active development and may require further refinement, but in my experience, it is already stable.
Collapse
