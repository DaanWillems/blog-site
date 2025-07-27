---
title: "Building a database (part 2): Write-ahead logging and SSTable organization"
date: "2025-07-26"
summary: "Small summary"
toc: true
readTime: true
autonumber: true
math: true
katex: true
tags: []
showTags: false
---

In this part we'll improve the persistence and crash resistance of the database. We will also start to work on organizing multiple SSTables on disk. 

## Cleaning up
First we'll clean up the code a bit. All code files relating to the database internals are moved into a database package. We'll create a database.go file that will become the entrypoint into the package. 

![Image displaying directory layout](/images/directory-layout-part2.png)

In the database.go file we'll add logic for starting the database 

```go
package database

import (
	"bufio"
	"os"
)

type Config struct {
	MemtableSize int //Threshold of entries before flushing to disk
}

var memtable Memtable
var config Config

func InitializeDatabase(cfg Config) {
	memtable = NewMemtable()
	config = cfg
}
```
The MemtableSize is the maximum number of entries the memtable may contain before flushing to disk. At a later point it will make more sense to look at the total number of bytes in the table, as entries can vary in size.  

We'll also need methods for inserting and retrieving data

```go
func Insert(id int, values []string) {
	byteValues := [][]byte{}
	for _, v := range values {
		byteValues = append(byteValues, []byte(v))
	}

	entry := MemtableEntry{
		id:      []byte{byte(id)},
		values:  byteValues,
		deleted: false,
	}

	memtable.Insert(entry)

	if memtable.entries.Len() >= config.MemtableSize {
		table, err := CreateSSTableFromMemtable(&memtable, 10)
		if err != nil {
			panic(err)
		}

		table.Flush("./test.db")

		memtable = NewMemtable() // Reset memtable after flushing
	}
}
```
The insert function is also responsible for flushing and refreshing the memtable when it exceeds the maximum number of entires

```go
func Query(id int) ([]byte, error) {
	//First check in the memtable
	entry := memtable.Get([]byte{byte(id)})

	if entry != nil {
		return entry.values[0], nil
	}

	//For now we only have the capability of writing and reading a single sstable from disk
	fd, err := os.Open("./test.db")

	if err != nil {
		panic(err)
	}

	reader := bufio.NewReader(fd) // creates a new reader

	entry, err = SearchInSSTable(reader, []byte{byte(id)})

	if err != nil {
		panic(err)
	}

	return entry.values[0], nil
}
```
The query function first looks into the memtable, and only after that into the SSTable on disk

## Write-ahead logging
The database has a basic level of persistence now. But if the program crashes or is terminated before writing the memtable to disk, this data is lost. 

To avoid this problem we can keep a separate file on disk to which all database operations are logged first called a WAL (Write-ahead log). Because this file is append only, it'll be very quick. Each time an entry gets inserted, we'll write it to the WAL first, and only then insert it into the memtable. 

Should our program crash, we can read and 'replay' the WAL to restore the state of the memtable. Once the memtable gets written to disk we can discard the WAL. 


## Conclusion
In this part we implemented a WAL to make sure data in the memtable is crash resistant. We also added the ability to store and search multiple SSTables on disk. It's starting to look like an actual database! 

In the next part we'll work on a very important feature: compacting tables. We'll implement a background process that automatically scans and compacts SSTables to reclaim disk space and improve lookup speeds.

