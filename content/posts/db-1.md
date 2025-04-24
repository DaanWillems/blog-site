---
title: "Building a database: Getting Started"
date: "2025-04-24"
summary: "Small summary"
toc: true
readTime: true
autonumber: true
math: true
katex: true
tags: []
showTags: false
---

In almost every project I ever built, a database was a central component. Often I'd read online about what specific database engines excelled in, and what they lacked. However, I never fully understood what this meant exactly. That always irritated me. I think in cases like this, the best way to learn more about a topic is to try build something myself. So I set out with very little knowledge but a lot of optimism to try built my own database engine. 

In this part I try to make a rough plan for building the database.

## Requirements 

Before starting with anything, it is important determine what requirements there are for the database. Clear requirements can direct the design while also preventing unnecessary work. 

- Data written must be persistent (not lost on system restarts)
- The database must be crash resistant
- There should be some sort of schema to define what the data looks like
- It must support: strings, integers and booleans. 
- It must support: Defining a primary key
- There should be a way of interfacing with the database (inserting, reading, updating and deleting data)
- There should be a primary index to speed up finding data
- The database should be optimized for quick writes

Some things that are unnecessary (for now):

- It is not an a relational database, so no need to represent relations
- No secondary indexes (Indexes on fields other than the primary key)
- Updating existing schema's

### Schema and Query Language

The schema should define what kind of data can be written to the database. It enforces that data is structured a certain way. The query language allows us to insert, read, update and delete data. We'll take heavy ‘inspiration’ from SQL to define our query language, which will also be used to define the schema. 

I want the schema to support tables, so we can have multiple structures. Tables consist of columns, where each column has a key and a type. One of the columns contains the primary key, which is used to build the primary index. Columns can have the values: string, integer and boolean. The query language should allow simple operations on the data, inserting, reading, updating and deleting. Given these requirements, let's define a grammar for the query language we'll call it: SSQL (Simple SQL).

```
<int> ::= [0-9]+

<string> ::= “[a-Z0-9_-]+"

<bool> ::= true

<bool> ::= false

<type> ::= int

<type> ::= string

<type> ::= bool

<expr> ::= <int>

<expr> ::= <string>

<expr> ::= <bool>

<expr_list> ::= <expr_list>

<expr_list> ::= <expr_list>,  <expr_list>

<expr_list> ::= <expr>

<expr_list> ::= <select_expr>

<expr_list> ::= <select_expr>,  <select_expr>

<select_expr> ::= <string>

<key_marker> ::= <null>

<key_marker> ::= primary

<column_def> ::= <string> <type> <key_marker>,

<where_expr> ::= <where_expr> && <where_expr>

<where_expr> ::= <where_expr> || <where_expr>

<where_expr> ::= <string> == <expr>

<where_expr> ::= <string> != <expr>

create table <string> {

<column_def>

};

delete table <string>;

select <select_expr> from <string> where <where_expr>;

update <string> set <expr_list> where <where_expr>;

delete from <string> where <where_expr>;

insert into <string> values <expr_list>;
```

This grammar is fully LL(1) compatible, and should therefore be pretty simple to parse. Now that is out of the way, let's move on to the actual data storing part. 

## Storing data on disk

A core part of a database is being able to store data persistently (on disk) and being able to search it. 

There are many ways of storing data safely on disk and accessing. Modern databases often use one of two data structures to store data: B-Trees or SSTables. While B-Trees are very good atsearching through the data quickly, is it complicated to store a B-Tree on disk. SSTables excel at writing quickly, and are used for popular databases such as Cassandra.

SSTables work by building a Sorted String table on disk. When data is written to the database, it is initially stored in an in memory data structure. When the in-memory data structure reaches a certain size, it is written to disk as an SSTable. The SSTable is immutable,and cannot be updated. This means mutations on existed records must be written to a new SStable. Altered records can be added to a new SStable, deleted records are also added, and tombstoned (marked as deleted).

Imagine a key value table written stored on disk. It includes a delete 

Table 1:

```
Key, Deleted, Value

1, False, "a"
2, False, "b"
3, False, "c"
6, False, "d"
```

At this point the user adds the record (4, 'a2'), removed the record with id 2 and updataethe record 6 to value 'd2'. These changes are written to a new table on disk:

Table 2:

```
Key, Deleted, Value

2, True,  "b"
4, False, "a2"
6, False, "d2"
```

When the database is queried, we first look into table 2, and then table 1. This ensures the database always returns the most up to date information. A downside of this approach is that if a key is queried that is not in the database, all tables will have to be searched. It also introduces data duplication. In order to mitigate this tables are compacted in the background over time. For example, the two tables above could be compacted into the following single table:

Table 3: 

```
Key, Deleted, Value

1, False, "a"
3, False, "c"
4, False, "a2"
6, False, "2"
```

SSTables enable quick writes, as the database only ever has to append to a file, which is muck quicker than writing to a file in random location.

## Designing the database
The database will consist of the following parts:

- A memory table that is used to store recent operations on the database

- A WAL (Write Ahead Log) to guarantee persistence of the memory table in case the computer shuts down

- SSTables that store ordered rows.

When a user inserts a new record, the actions is written to the WAL and inserted into the memory table (memtable). The memory table is ordered by the primary key, but the WAL is ordered by time. As more data is inserted over time, the memtable grows. When it has a reached a certain size (2MB). The table is written to a SSTable on disk. When the SSTable has reached some predefined size, it is closed and a new SSTable can be created. 

Each table in the database is given a separate directory with sstables, but all database commands are stored in the same memtable and WAL. 

### Inserting

If a user inserts a record, we write the insertion to the WAL and insert the row in the memtable in its appropriate ordered location.

### Updating

If a user updates a record, we update the data in the memtable, but if the record is not in there (because it's been flushed to disk) it is simply added to the memtable as if it is an insertion.

### Reading

When a user queries a specific primary key, the database engine searches the memtable. If it isn't there it then starts to search the most recent SSTable and all following SSTables until it finds it. This means that if the key is not in the database the process has to search everything. This part can be sped up by an indexing strategy. 

### Deleting

Like updating we we update the record by setting a deleted flag to true

## A first implementation

I've decided to implement the database in Golang as I'm familiar with it and it's simple to write. 

### Memtables

Let's start by implementing the memtable component. We need the memtable to be able to support different types of tables later on, so let's design it with that in mind.

We'll maintain an ordered list of entries. Each entry has a primary key, a deleted flag and a list of values. Depending on the table structure, the primary key might be a a string, int or bool. The values can also be any type. In order to support this the primary key and values with be stored as byte arrays. The database can then look into the table structure to determine how to interpret the values. 

```go
type Memtable struct {
	entries *list.List
}

type MemtableEntry struct {
	id      []byte
	values  [][]byte
	deleted bool
}
```

The memtable has functions for insert / update / get

```go
func (m *Memtable) Get(id []byte) *MemtableEntry {
	for e := m.entries.Front(); e != nil; e = e.Next() {
		next := e.Next()
		if next != nil && bytes.Equal(id, next.Value.(MemtableEntry).id) {
			entry := next.Value.(MemtableEntry)
			return &entry
		}
	}
	return nil
}

func (m *Memtable) Update(id []byte, values [][]byte) {
	entry := MemtableEntry{
		id:      id,
		values:  values,
		deleted: false,
	}

	for e := m.entries.Front(); e != nil; e = e.Next() {
		if bytes.Equal(e.Value.(MemtableEntry).id, id) {
			e.Value = entry
			return
		}
	}
	return
}

func (m *Memtable) Insert(id []byte, values [][]byte) {

	entry := MemtableEntry{
		id:      id,
		values:  values,
		deleted: false,
	}

	for e := m.entries.Front(); e != nil; e = e.Next() {
		next := e.Next()
		if next != nil && bytes.Compare(id, next.Value.(MemtableEntry).id) == -1 {
			m.entries.InsertBefore(entry, next)
			return
		}
	}

	m.entries.PushBack(entry)
}
```

SSTables

When the memtable is full, we want to store it to disk. This is where the SSTable comes in. 




