# Personal Knowledge Graphs in AI RAG-powered Applications with libSQL

I spend a long time working on privacy first personal ai assistant at [mykin.ai](mykin.ai)

Our application is localfirst and focused on sovereign data ownership. So, one of the challenges was finding the proper storage of data.

## Device friendly

I partially describe several options for embeddable and device-friendly databases in my previous article
[Embeddable and Portable Storage for AI-Powered Apps and Personal Knowledge Graphs](https://ai.plainenglish.io/embeddable-and-portable-storage-for-ai-powered-apps-and-personal-knowledge-graphs-e46b0564e73c)

AI-powered apps, especially the semantic memory part, set few expectations for database capabilities

-   general queries on structured data (regular application data) like messages, conversations, settings, etc
-   vector search and similarity search capabilities to RAG pipelines and different LLM and ML-powered flows
-   Graph and graph search capabilities (ML and semantic memory )

As far as we work on mobile, we have a few tech capabilities, too

-   embeddable with good support for mobile bindings
-   single file database that simplifies a backup
-   portable
-   battery friendly
-   fast and nonblocking io as much as possible
-   wide community support
-   reliability

Vector and Graph capabilities for embeddable databases were relatively new challenges for modern databases. The close competitor was Postgress, with extensions that add
[Apache AGE](https://age.apache.org/?source=post_page-----50b0e7aa10c4--------------------------------)

and

[pgvector](https://github.com/pgvector/pgvector?source=post_page-----50b0e7aa10c4--------------------------------)

So, we need a similar embeddable setup.

## Graphs

Currently, there are practically no graph-oriented databases that have portable and embeddable capabilities with a mobile or small device friend setup.

I made a few articles to model and show how to use relational databases for graph and hypergraph capabilities. It is a wide topic that has a lot of exciting research topics. You can find more about it in my articles.
[Personal Knowledge Graphs. Semantic Entity Persistence in Relational Model](https://blog.stackademic.com/personal-knowledge-graphs-semantic-entity-persistence-in-relational-model-d5692bb8e8bb)

## Vectors

It is a wide variety of vector databases and libraries on the market. Some of the libraries like [faiss](https://github.com/facebookresearch/faiss?source=post_page-----50b0e7aa10c4--------------------------------) .

It is top-performing and even has some capabilities to persist sectors to a file. So, it was a good start, but...

It is a big challenge to keep heterogeneous data storages in sync, and even the process of sync will consume time, battery, and CPU resources of an application and eat the main thread of the app that will, making it less and less user-friendly.

For me, it was clear — we need something integrated into a database.

After weeks of research and a prototype, we stopped at the SQLite ecosystem. SQLite has been the most popular and reliable database for mobile devices for decades.

But what about vectors?

SQLite has an extendable architecture that allows native modules to extend a database capability. I found this project that brings faiss to a SQLite.
[sqlite-vss](https://github.com/asg017/sqlite-vss?source=post_page-----50b0e7aa10c4--------------------------------)
Unfortunately, it was not reliable and had a few major bugs and issues, so I almost gave up.

We were lucky to find a better answer to our question. [LibSQL](https://turso.tech/libsql?source=post_page-----50b0e7aa10c4--------------------------------)

[Libsql](https://github.com/tursodatabase/libsql?tab=readme-ov-file&source=post_page-----50b0e7aa10c4--------------------------------) is open to contribution. It is an open-source replacement for SQLite that brings many features and performance optimization to the table. Some of these features deserve a separate article,

like :

-   alter table extension that makes a migration easy
-   webassembly defined functions !
-   Virtual WAL interface

and much more

but the most critical one is that it extends SQLite with vector search capabilities.
[AI & Embeddings](https://docs.turso.tech/features/ai-and-embeddings?source=post_page-----50b0e7aa10c4--------------------------------)

It is built smartly with minimal database changes, so it is easy to migrate and still compatible with SQLite.

It is no vector type, or let's say it is aliased on top of BLOB

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

So **F32\_BLOB(512)** specifies a meta information about a vector. It is a value type float 32-bit and a dimension of the array.

This type is more an alliance on BLOB but it gives a posibility to a database to validate a vector shape and it is data types

Now we have the ability to use a vector search for clustering a Graph and use it in a LLM powered pipelines

On edge, we store few meta information about triple

-   display label — describe edge label un normalized one
-   label — normalized label
-   vectorTripple — is the most interesting part. we normalize a node labels and edge label and concat them together. it allows us to make embedding out of it and make edges searchable by vector search

Our vectorTripple column adds a vector search capability to our personal knowledge graph .

To insert data we could use a vector function that gets embedding as :

-   float32 array
-   a blob of serialized float32 array
-   string representation like ‘\[0.5432635, 0.3333 ….\]’

```js
insert into edge (rowid, id, fromId, toId , label, vectorTriple, displayLabel, createdAt, updatedAt)
  values (? , ?, ? , ? , ? , vector(${this._store.toVector(
          await this.embeddingsService.embedDocument(`${fromNode.label} ${normalizedLabel} ${toNode.label}`)
        )}) , ? , ?, ?);
```

With **vector\_distance\_cos** function we could already do a distance calculation and queries

```sql
select  e.id, e.label, vector_distance_cos(e.vectorTriple , ${vector}) distance 
from egde e 
where distance < 0.15
```

It is fantastic but very slow and ineffective and CPU intensive as far as you need to scan the full table and calculate the distances. So, we need a vector index.

We are lucky we can create an index for embedding columns !!!

```
CREATE INDEX idx_edge_vectorTriple ON edge (libsql_vector_idx(vectorTriple));
```

Now, instead of a full scan, we could search in the index

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

So let's step by

```
vector_top_k('idx_edge_vectorTriple', '[0.32323, 0.525, ....]', 20) as i
```

This will give us back **rowIDs** of similarity and vector search from index **idx\_edge\_vectorTriple**, based on the **vectorTriple** column.

You should be careful. By default, it uses rowid. So, to combine the result of the vector search with any data and tables in your DB, you need to use a simple join statement. All is an integral part of the base and queriable

```
inner join edge as e on i.id = e.rowid
```

The index does not return distances, so you still need to calculate them yourself self, but now it happens on the way smaller dataset

```sql
select  e.id, e.label, vector_distance_cos(e.vectorTriple , vector('[0.32323, 0.525, ....]')) distance
```

vector function is a smart move that converts string like ‘\[1,32,2, …. \]’ to a blob type that stored in a database

Now you could do an extra filter on distances if you need to find close related triples

```
where distance <= 0.15
```

Now we have a vector search on top of personal Knowledge


## Conclusion

Personal Knowledge Graphs can be modeled in a relation model and are usually relatively middle-sized, so they never lead to performance issues with SQL queries. With Libsql, we have native and low-level support of vectors. It allows us to build a clustering of the graph and RAG pipelines on the user devices if needed. This feature is still under active development and may require some time, but from my experience, it is stable now.
