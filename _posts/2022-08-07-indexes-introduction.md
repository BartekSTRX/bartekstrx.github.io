---
title: "Database indexes - introduction"
date: 2022-08-07
---


# Introduction
This is the first post in a series about various topics related to SQL databases. I will cover topics such as DB indexes, query analysis and optimization, and less-known SQL features. It is not intended as an in-depth treatment of those topics, but as an introduction and non-formal overview. In the examples, I will focus on Microsoft SQL Server, but most of the things I will cover apply to any relational database system. 

In this post, I will start by explaining what database indexes are. 


# Database indexes
Database indexes are auxiliary data structures in a database. They store data derived from the data stored in database tables. The main reason to use DB indexes is to make queries run faster. 
The downside of using them is that all other operations (insert, update, delete) are slower - each time we modify data in a table, the DB engine has to make corresponding modifications in all indexes for that table. 
So this is a trade-off: if we do not create any indexes, some queries may run slowly, but inserts and updates will run fast; if we create too many indexes on a table, queries will be executed faster, but everything else will slow down. 

As a result, typically we should create indexes only for the most often executed queries. In one of the next posts, I will cover basic techniques for identifying the most common queries run on a database table. 


# Kinds of indexes
There are several kinds of indexes
- row store indexes
    - clustered
    - nonclustered
- spatial indexes - a special type of indexes that can be created on columns of spatial types, that is geometry or geography
- XML indexes - indexes on columns of type XML, useful if we want to run XPath queries on columns of XML type
- full-text search indexes - only columns of certain types, such as char and varchar can be indexed for full-text search

Additional SQL features related to indexes are
- indexing a computed column
- indexes with included columns
- filtered indexes

Typically when people talk about database indexes, they mean nonclustered row store indexes.


# Clustered and nonclustered indexes
MS SQL Server has a somewhat broad definition of what an index is - it uses the term "clustered index" to refer to an index that stores the table data itself. Oracle uses the term "index-organized table" which perhaps better describes what it is - it's a table that has its data stored in an index-like structure, not an auxiliary index containing derived data used to speed up queries execution. 

In MS SQL Server every table by default has a clustered index. Since a clustered index stores the table data, it's clear that there can be only one such index for the table. There is no such thing as a database table with two or more clustered indexes. 

Both clustered and nonclustered indexes are commonly implemented using a data structure called B-tree. While a clustered index stores table data in leaf nodes of the tree, a nonclustered index stores a pointer to a node in the clustered index. I will cover B-trees and more details about indexes implementation in the next post in this series. 


# Heap tables
As mentioned in the previous section, every table can have at most one clustered index. But can a table not have one? The answer is yes, such a table is called a heap table, heap-organized table, or simply a heap. While in a clustered index data is stored ordered by the clustering key, in a heap table data is stored unordered. Lack of order makes some operations, like inserting new rows, faster. 
A heap table can have nonclustered indexes, similar to a table with a clustered index. In that case, a nonclustered index stores a row ID (RID) instead of a pointer to a node in the clustered index. 

Using heap tables is a bit more advanced technique, and I will not focus on them here. 

# Summary
In this post, I explained what database indexes are and the basics of how data is stored in database tables. In the next post, I will explain what are B-trees, data structures commonly used for implementing database indexes. 

# References
Microsoft Docs 
- [indexes](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/indexes?view=sql-server-ver16)
- [heap tables](https://docs.microsoft.com/en-us/sql/relational-databases/indexes/heaps-tables-without-clustered-indexes?view=sql-server-ver16)

