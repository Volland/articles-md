# Building Vector Search and Personal Knowledge Graphs on Mobile with libSQL and React Native

Graphs and vector search form a powerful combination for AI-powered applications, which are rapidly growing in popularity. Personal knowledge graphs serve as the foundation of semantic memory for many agent-based AI applications.

At [Kin](https://mykin.ai/), we develop AI agent-based architecture with a sophisticated memory model that is stored directly on the user's device. Kin is a personal AI that helps you navigate your work-life, available on both iOS and Android.

## Our Guiding Principles

The technical guiding principles, or North Stars, of Kin are:

-   [Privacy by Design](https://volodymyrpavlyshyn.medium.com/privacy-by-desing-and-laws-of-ide-991c80170d35) — A guide on maintaining secure and private architecture.
-   [Local-first Architecture](https://volodymyrpavlyshyn.medium.com/business-benefits-of-local-first-for-founders-and-products-f212367b3537) — Keeps user data on device, enables edge ML, faster development.
-   [SSI Principles](https://volodymyrpavlyshyn.medium.com/self-sovereign-identity-in-7-toots-c29fff0f7961) — Focused on user data ownership and sovereignty.

### Data Ownership

All of these principles have one key aspect in common—data ownership. Users have full control and ownership of their data. This shifts away from the traditional, fully cloud-based, centralized model to a local-first architecture, where data is stored and processed on a network of user devices. Cloud services and capabilities may still play a role, but the primary processing happens locally.

This means that complex RAG (retrieval-augmented generation), vector search, and graph clustering must primarily occur on the user's device.

## Expectations for Database Capabilities

Our database capabilities need to support the following:

-   General queries on structured data, such as messages, conversations, and settings.
-   Vector search and similarity search for RAG pipelines and various LLM (Large Language Model) and machine learning-powered workflows.
-   Graph and graph search capabilities for machine learning and semantic memory.

Since we focus on mobile, there are additional technical requirements:

-   Embeddable with strong support for mobile platforms.
-   A single-file database that simplifies backups.
-   Portability across devices.
-   Battery-efficient.
-   Fast and non-blocking I/O wherever possible.
-   Wide community support.
-   Reliability.

## libSQL

If you've been following my articles, you likely already know the answer — [libSQL](https://frontend-git-blog-ai-rag-mobile-chiselstrike.vercel.app/libsql).

I’ve documented the entire journey of implementing vector search and graphs on top of relational models [in my articles](https://ai.plainenglish.io/personal-knowledge-graphs-in-ai-rag-powered-applications-with-libsql-50b0e7aa10c4).

Now, the main question is: how do we run libSQL on a user’s device?

Since we're using React Native, the library must have React Native bindings.

## libSQL on React Native

There are plenty of libraries for React Native that support SQLite, but none for libSQL. Let’s explore some of the most popular options:

#### react-native-sqlite-storage

-   Widely used with support for transactions and raw SQL queries.
-   Compatible with both Android and iOS.
-   Offers a promise-based API.

#### react-native-sqlite-2

-   A lightweight alternative.
-   Based on the WebSQL API.
-   Suitable for simple databases, but with fewer features compared to `react-native-sqlite-storage`.

#### react-native-sqlite

-   Similar to `react-native-sqlite-storage`, but with more minimal features.
-   May require manual linking.

#### watermelondb

-   Built on top of SQLite, offering a more modern approach.
-   Designed for highly scalable databases in React Native apps.
-   Provides an ORM-like interface and efficiently handles large datasets.

#### expo-sqlite (if using Expo)

-   Built-in SQLite support for Expo apps.
-   Lightweight and easy to use but lacks the advanced features found in other libraries.

`expo-sqlite` has become the default library for SQLite in the Expo ecosystem. Initially, I considered advocating for libSQL integration within the community, or forking it to meet our internal needs.

However, this proved far more challenging than anticipated. Large open-source projects can sometimes be resistant to new ideas and improvements, making this a difficult path to navigate.

### OP-SQLite

While searching GitHub, I came across [OP-SQLite](https://github.com/OP-Engineering/op-sqlite), which was described as the fastest SQLite library for React Native, developed by Ospfranco.

It offers several interesting features for React Native apps:

#### Async Operations

By default, queries run synchronously on the JavaScript thread. However, OP-SQLite provides asynchronous versions for some operations, which offload SQLite processing to a separate thread, preventing UI blocking. It also supports true multi-concurrency, so it won't overload the event loop.

#### Raw Execution

If you don't need to handle keys, you can use a simplified execution method that returns an array of results.

#### Hooks

You can subscribe to database changes using an update hook, which provides full row updates.

```ts
// Bear in mind: rowId is not your table primary key but the internal rowId sqlite uses to keep track of the table rows
db.updateHook(({ rowId, table, operation, row = {} }) => { console.warn(`Hook has been called, rowId: ${rowId}, ${table}, ${operation}`); 
// Will contain the entire row that changed only on UPDATE and INSERT operations
console.warn(JSON.stringify(row, null, 2)); });
db.execute('INSERT INTO "User" (id, name, age, networth) VALUES(?, ?, ?, ?)', [ id, name, age, networth, ]);
```

#### Extension Load

This was the first library that allowed me to load an extension by myself—and more! Oskar added the CR-SQL extension as an option within the library, making it work right out of the box.

#### Open to Cooperation

One of LibSQL’s core principles is openness to contributions. Oskar embraced this philosophy, recognized the benefits of libSQL, and added it as an option to OP-SQL.

## Let's Learn How to Use OP-SQLite

So, how can you build a vector search-enabled personal knowledge graph directly on a user’s device?

I assume you already have a React Native or Expo project set up. You will need to add `op-sql` (version 7.3.0 or higher):

```bash
npm install @op-engineering/op-sqlite
```

Next, configure libSQL by adding the following section to your `package.json`:

```json
"op-sqlite": { "libsql": true }
```

Since we're working with a polymorphic library that can run both on devices and Node.js, I created an abstraction that allows for swapping between libSQL implementations as needed.

```ts
import {
  open as openLibsql,
  OPSQLiteConnection,
  QueryResult,
  Transaction,
} from "@op-engineering/op-sqlite";
import {
  BatchQueryOptions,
  DataQuery,
  DataQueryResult,
  IDataStore,
  UpdateCallbackParams,
  StoreOptions,
} from "@mykin-ai/kin-core";
import { documentDirectory } from "expo-file-system";
export class DataStoreService implements IDataStore {
  private _db: OPSQLiteConnection | undefined;
  private _isOpen = false;
  public _name: string;
  private _location: string;
  public useCrSql = true;
  private _options: StoreOptions;
  constructor(
    name = ":memory:",
    location = documentDirectory,
    options: StoreOptions = {
      vectorDimension: 512,
      vectorType: "F32",
      vectorNeighborsCompression: "float8",
      vectorMaxNeighbors: 20,
      dataAutoSync: false,
      failOnErrors: false,
      reportErrors: true,
    }
  ) {
    this._name = name;
    this._options = options;
    if (location?.startsWith("file://")) {
      this._location = location.split("file://")[1];
    } else {
      this._location = location;
    }
    if (this._location.endsWith("/")) {
      this._location = this._location.slice(0, -1);
    }
  }
  getVectorOption() {
    return {
      dimension: this._options.vectorDimension,
      type: this._options.vectorType,
      compression: this._options.vectorNeighborsCompression,
      maxNeighbors: this._options.vectorMaxNeighbors,
    };
  }
  async query(
    query: string,
    params?: any[] | undefined
  ): Promise<DataQueryResult> {
    try {
      await this.open(this._name);
      const paramsWithCorrectTypes = params?.map((param) => {
        if (param === undefined || param === null) {
          return null;
        }
        if (param === true) {
          return 1;
        }
        if (param === false) {
          return 0;
        }
        return param;
      });
      const data = await this._db.executeRawAsync(
        query,
        paramsWithCorrectTypes
      );
      return { isOk: true, data };
    } catch (e) {
      console.error(e.code, e.message);
      return {
        isOk: false,
        data: [],
        errorCode: e.code || "N/A",
        error: e.message,
      };
    }
  }
  async execute(
    query: string,
    params?: any[] | undefined
  ): Promise<DataQueryResult> {
    try {
      await this.open(this._name);
      const paramsWithCorrectTypes = params?.map((param) => {
        if (param === undefined || param === null) {
          return null;
        }
        if (param === true) {
          return 1;
        }
        if (param === false) {
          return 0;
        }
        return param;
      });
      const data = await this._db.executeAsync(query, paramsWithCorrectTypes);
      return { isOk: true, data: data.rows?._array ?? [] };
    } catch (e) {
      console.error(e);
      return {
        isOk: false,
        data: [],
        errorCode: e.code || "N/A",
        error: e.message,
      };
    }
  }
  async open(name: string): Promise<boolean> {
    try {
      if (this._isOpen && name === this._name) {
        return true;
      }
      if (this._isOpen && name !== this._name) {
        await this.close();
        this._isOpen = false;
      }
      this._name = name;
      this._db = openLibsql({ name: this._name, location: this._location });
      console.log("Opened db");
      this._isOpen = true;
      return true;
    } catch (e) {
      console.error("couldn't open db", e);
      return false;
    }
  }
  async isOpen(): Promise<boolean> {
    return Promise.resolve(this._isOpen);
  }
  async close(): Promise<boolean> {
    if (this.useCrSql) {
      this._db.execute(`select crsql_finalize();`);
    }
    this._db.close();
    this._isOpen = false;
    return Promise.resolve(true);
  }
}

```

Now we are ready to create graph tables and indexes. I’ll skip the entire class as it is too long, and focus on the essential parts:

```ts
const vectorOptions = this._store.getVectorOption()
```

This provides vector configurations, such as the type of vector value and the dimension of embeddings, as well as vector index parameters:

```ts
const createR = await this._store.execute(`create table if not exists edge ( id varchar(36) primary key not null, fromId varchar(36) not null default '', toId varchar(36) not null default '', label varchar not null default '', displayLabel varchar not null default '', vectorTriple ${vectorOptions.type}_BLOB(${vectorOptions.dimension}), createdAt real, updatedAt real, source varchar(36) default 'N/A', type varchar default 'edge', meta text default '{}' ); `)
```

Now we have a triple store that references nodes:

```ts
const createR = await this._store.execute(`create table if not exists node ( id varchar(36) primary key not null, label varchar not null default '', vectorLabel ${vectorOptions.type}_BLOB(${vectorOptions.dimension}), displayLabel varchar not null default '', createdAt real, updatedAt real, source varchar(36) default 'N/A', type varchar default 'node', entity text default '{}', meta text default '{}' ); `)
```

If you want to learn more about how to model graphs in relational databases, [read this](https://blog.stackademic.com/personal-knowledge-graphs-semantic-entity-persistence-in-relational-model-d5692bb8e8bb).

### Now it's time to create an index:

```ts
const createIndex = await this._store.execute(` CREATE INDEX IF NOT EXISTS idx_edge_vectorTriple ON edge (libsql_vector_idx(vectorTriple${vectorOptions.compression !== 'none' ? `, 'compress_neighbors=${vectorOptions.compression}'` : ''} ${vectorOptions.maxNeighbors ? `, 'max_neighbors=${vectorOptions.maxNeighbors}'` : ''})); `)
```

We configure `compress_neighbors` and `max_neighbors` to optimize storage space. If you want to learn more about space complexity, [read this](https://ai.plainenglish.io/the-space-complexity-of-vector-search-indexes-in-libsql-3fadb0cdee96).

Now, we can create a triple:

```ts
const createOp = await this._store.execute( ` insert into edge (id, fromId, toId , label, vectorTriple, displayLabel, createdAt, updatedAt) values (?, ? , ? , ? , vector(${this._store.toVector( await this.embeddingsService.embedDocument( `${fromNode.label} ${normalizedLabel} ${toNode.label}`, ), )}) , ? , ?, ?); `, [ this._getUuid(), fromNode.id, toNode.id, normalizedLabel, label, Date.now(), Date.now(), ], );
```

Unfortunately, `op-sql` does not support `float32array` as a parameter like libSQL does. To work around this, we use a bit of dynamic SQL and serialize the vector as part of the query. My `toVector` method serializes the `float32array` and handles quotes. Note that we pass the serialized array to the vector function in SQL. Hopefully, the next version of op-SQL will support `float32arrays`.

### Time to query:

```ts
const _top = top ?? 10 const vector = this._store.toVector(await this.embeddingsService.embedQuery(query)) const querySql = ` select e.id, e.label, e.displayLabel, e.createdAt, e.updatedAt, e.source, e.type , e.meta , fn.label, fn.displayLabel, tn.label, tn.displayLabel, vector_distance_cos(e.vectorTriple , ${vector}) distance from vector_top_k('idx_edge_vectorTriple', ${vector} , ${_top}) as i inner join edge as e on i.id = e.rowid inner join node as fn on e.fromId = fn.id inner join node as tn on e.toId = tn.id where 1=1 ${maxDistance ? `and distance <= ${maxDistance}` : ''} order by distance limit ${_top}; ` const edgeData = await this._store.query(querySql)
```

A few notes:

- The vector index returns `rowid` by default, so be mindful of the joins.
- The index does not return distance automatically, but you can calculate it if needed.
- `vector_top_k` expects a `top` parameter and will return the top N items. If you have complex filtering or external top limits, remember to set a higher top N to ensure a more comprehensive search. In our case, this isn’t an issue.

## Issues and Challenges

I encountered a few challenges in React Native, particularly on iOS. These issues are mainly related to how native modules are compiled and linked on iOS.

One frustrating issue is that if you have another library using a different version of SQLite, it can unpredictably override linking and break libSQL entirely.

### Compilation Clashes

If you have other packages dependent on SQLite (especially those that compile it from source), you will encounter problems.

Some known offenders include:

-   `expo-updates`
-   `expo-sqlite`
-   `cozodb`

You may face duplicate symbols and/or header definitions since each package will try to compile SQLite from its own source. Even if they compile successfully, they may use different compilation flags, which could lead to threading errors.

Unfortunately, there’s no easy solution. You’ll need to manually resolve the double compilation, either by patching the compilation of each package or removing the conflicting dependency.

On Android, you might get away with using a pickFirst strategy ([here’s an article](https://ospfranco.com/how-to-resolve-duplicated-libraries-on-android/) on how to do that). On iOS, depending on the build system, you may be able to patch it with a post-build hook, like this:

```
pre_install do |installer|
 installer.pod_targets.each do |pod|
  if pod.name.eql?('expo-updates')
   # Modify the configuration of the pod so it doesn't depend on the sqlite pod
  end
 end
end
```

Follow [op-sql docs](https://ospfranco.notion.site/Gotchas-bedf4f3e9dc1444480fc687d8917751a) for an updated list of libraries.

### RNRestart Crash

Another iOS issue:

```ts
import RNRestart from 'react-native-restart';
```

If you need to restart the app using `react-native-restart`, make sure you close all connections beforehand:

```ts
import { closeAllConnections } from '@storage/data-store-factory'; 
import RNRestart from 'react-native-restart'; 
export const restartApplication = async (): Promise<void> => { 
  await closeAllConnections(); RNRestart.restart();
};
```

Now, you can create a personal knowledge graph with vector search on a user device!

## Sounds interesting?

If this sounds interesting and you’d like to work with us on these exciting projects, please reach out at jobs@mykin.ai!

I want to extend my thanks to Oskar and the Turso team for their incredible work.
