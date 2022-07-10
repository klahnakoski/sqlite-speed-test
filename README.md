# sqlite-speed-test
Can Sqlite be faster at set intersection than numpy?


## Context

This project is one step in asking the question:  Can Python coordinate a large number of Sqlite databases, over multiple machines, and serve as a transactional data warehouse?

The idea is to let Python perform the high level decisions, while Sqlite does the number crunching. The Python system talks in terms of SQL queries and Sqlite files; accepting input as Sqlite files; providing query results as Sqlite files.

> This is a data warehouse project: Joins will be limited

### Nomenclature

* **Node** - A machine, or independent process, that has CPU, memory and disk resources
* **Cluster** - Many nodes that act as a single warehouse
* **Shard** - Data is split into shards, which are scattered evenly over the nodes.
* **File** - A shard can be split into many sqlite database files on a single node. 
* **Database** - same as "file", since sqlite database is a single file

### Queries

Queries will be go through the map/reduce process. Each shard will be given a query, and the results are merged to a central node. Python does not deal with records; rather, it forms the queries that will let Sqlite merge the results from multiple files.

* network map phase - send query to all nodes 
* local map - each node appears to have one shard, but really it has many shards; the query is further distributed to the local shards
* local union - the query results are merged, then sent back to originating node
* network union - results from all nodes are merged for final result

Since query results are Sqlite files, they can be enormous in size.

### Insert data

Data is submitted as Sqlite files. The process of converting from the external format (JSON documents) to the Sqlite representation is called "ingestion". Ingestion can be parallelized easily. If ingestion is provided as client library code, then we can increase the overall ingestion speed of the cluster. 

There is only one file per node accepting new records at a time; it is called the ingestion file. All other files are "static"

* Create database - make a new database, fill it with data
* submit database to nodes - this should be a simple file transfer
* confirm reciept - required for transactions
* merge databases - every submitted database is a shard: The nodes will merge these shards as necessary

Shards will probably be merged in pairs; each shard being twice the size of the next; minimizing the number of times the data gets scanned.

### Remove data

It is very useful to be able to replace or update data. This will be done by removing data from existing shards, and adding it to the smallest shard. 

* delete - delete command distributed to all shards to ensure it is gone
* insert - added to ingestion shard

There may be an "inverse" database: A database of deleted records. This will allow a shard to be truly static, while still supporting updates. When shards are merged, they can be merged with their inverse for a net effect.  


### Redundancy and replication

Replication provides an opportunity to distribute many queries among the many nodes for faster response times. Replication also provides backup and redundancy to mitigate the loss of a node.

A simple file transfer of a shard provides replication, but this can be a long time for large databases. A synchronisation process is required

* lock merges - a node will prevent its shards from merging; except for the ingestion shard, which can continue to accept inserts
* copy databases - copy all the locked shards while the ingestion shard grows, repeat lock-and-copy until there is only one (small) shard left to copy
* lock inserts - stop ingestion for the last shard
* copy last database - and register new node as able to accept yet-more-data   
* unlock inserts - resume ingestion
* resume merges - both nodes can resume merges

### Locality

Due to the very large size of the warehouse, being able to leverage S3 for columns that are not used often will be a great benefit.

* memory - databases in memory are best for fastest query times
* local drive - unopened databases can remain on the node's drive for reasonable response time
* S3 - large shards, or shards not used often, can be left on S3 

### Merges

A single node may have its data split among many shards.

All shards on a node are static, except for the ingestion shard. This means merges consist of reading two or more shards, and writing out a new combined shard as a result. When the merge is complete, the Python controller can redirect queries to this new
shard, and delete the others.


### Format

The Sqlite database may have the following format

* Each column has their own database? - this allows use to move not-used columns to S3, and tightly pack sparse columns. Queries must be split into two phases: First find the matching document ids, then accumulate the result.
* Each datebase is two tables - Using two tables, rather than table-and-index, might be best for merges: The sorted nature should allow merge join, which is a fast single-pass process. Sqlite may not be efficient rebuilding the new, merged, index. Tests are required.    
    * primary - map from document sorted id to field value
    * index - map from sorted field value to document id

This format may not work well with query results.  Query results should be as flat as possible; much like a CSV file.  Having a columnar format for internal data and a row format for query results, may add complication. 



