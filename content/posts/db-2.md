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

## Combining the Memtable and SSTable
In the previous part, we coded a simple Memtable and SSTable to store data. The next step is two combine these two parts behind one interface. This will become the 'storage engine', the component of the database that is in charge of writing, reading and organizing in disk. 

For now the behaviour of the storage engine can be described as follows:

If we insert into the database, it should be written to the memtable. If the memtable exceeds a given size, it should be flushed to disk. When querying we should first look into the memtable, and only then into the SSTable on disk. 

Before moving on, we'll clean up the code a bit. All code files relating to the storage engine  are moved into a package. We'll create a storage_engine.go file that will become the entrypoint into the package. All functions not in this file will be made private.

![Image displaying directory layout](/images/directory-layout-part2.png)

In the storage_engine.go file we'll add logic for starting the storage engine 

```go
type Config struct {
	MemtableSize int //Threshold of entries before flushing to disk
}

var memtable Memtable
var config Config

func InitializeStorageEngine(cfg Config) {
	memtable = NewMemtable()
	config = cfg
}
```
The MemtableSize is the maximum number of entries the memtable may contain before flushing to disk. At a later point it will make more sense to look at the total number of bytes in the table, as entries can vary in size.  

We'll also need methods for inserting and retrieving data.

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
		table, err := createSSTableFromMemtable(&memtable, 10)
		if err != nil {
			panic(err)
		}

		table.Flush("./test.db")

		memtable = newMemtable() // Reset memtable after flushing
	}
}
```
The insert function is also responsible for flushing and refreshing the memtable when it exceeds the maximum number of entries

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

	entry, err = searchInSSTable(reader, []byte{byte(id)})

	if err != nil {
		panic(err)
	}

	return entry.values[0], nil
}
```
The query function first looks into the memtable, and only after that into the SSTable on disk

## Write-ahead logging
The database has a basic level of persistence now. But if the program crashes or is terminated before writing the memtable to disk, this data is lost. 

To avoid this problem we can keep a separate file on disk to which all database operations are logged first, called a WAL (Write-ahead log). Because this file is append only, it'll be very quick. Each time an entry gets inserted, we'll write it to the WAL first, and only then insert it into the memtable. 

Should our program crash, we can read and 'replay' the WAL to restore the state of the memtable. Once the memtable gets written to disk we can discard the WAL. 

The format for writing to the WAL is very simple, we'll serialize each entry and write the length of content + the content of the entry. This will also us to very easily read  it back. We'll also need a method for clearing the WAL once the memtable gets written.

```go
var wal_file *os.File
var wal_path string

func openWAL(path string) {
	var err error
	wal_path = path
	wal_file, err = os.OpenFile(path, os.O_APPEND|os.O_CREATE|os.O_WRONLY, 0644)
	if err != nil {
		panic(err)
	}
}

func resetWAL() {
	var err error
	wal_file.Close()
	wal_file, err = os.Create(wal_path)
	if err != nil {
		panic(err)
	}
}

func writeEntryToWal(entry MemtableEntry) {
	len, content := entry.Serialize()
	wal_file.Write([]byte{byte(len)})
	wal_file.Write(content)
	wal_file.Sync()
}
```

### Replaying

When the application starts, it should try to open the WAL into read mode and restore the memtable. We'll start by reading the first byte to get the entry size, and then the entry which gets deserialized and inserted into the memtable. We continue until there are no more entries to read. Conveniently the memtable object is accessible to us as it's all in the database package.

```go
func ReplayWal() {
	fd, err := os.Open(wal_path)
	if err != nil {
		panic(err)
	}
	defer fd.Close()

	reader := bufio.NewReader(fd)

	for {
		entryLen := make([]byte, 1)
		_, err = reader.Read(entryLen)

		if err != nil {
			if err == io.EOF {
				break
			}
			panic(err)
		}

		content := make([]byte, int(entryLen[0]))
		_, err = reader.Read(content)

		if err != nil {
			panic(err)
		}

		entry := &MemtableEntry{}
		entry.Deserialize(content)

		memtable.Insert(*entry)
	}
}
```

Now we only need to call the appropriate methods in the database.go file.
In initialize we'll try to replay the wal to restore the memtable, and then open it for writing:

```go
func InitializeStorageEngine(cfg Config) {
	memtable = newMemtable()
	replayWal("wal")
	openWAL("wal")
	config = cfg
}
```

Before inserting into the memtable, we'll insert into the WAL

```go
writeEntryToWal(entry)
memtable.Insert(entry)
```

## File ledger
Right now the data is always written to the file 'data.db'. This works well, until we want to flush the memtable a second time. We need a system for maintaining multiple files on disk. The storage engine needs to know which files already exist and can be searched.

We'll start by making a dedicated directory for storing data, we can also place the WAL in this directory. Instead of writing to 'data.db' when flushing to disk, a unique name is generated for each new file. This way we can store data in more than one file.

In order to keep track of what files contain data, we have to do some book keeping. Each time we add a new file, we'll also add the new file name to a ledger file. When we want to look for data in the storage engine, we can first do a lookup to see which files are available, and search them until we find what we are looking for. 

ledger.go:
```go
package storage

import (
	"bufio"
	"fmt"
	"os"
	"time"
)

type FileLedger struct {
	loaded     bool
	ledgerFile *os.File
	ledger     []string
}

var fileLedger FileLedger

func closeLedger() {
	if !fileLedger.loaded {
		panic("File ledger is not loaded")
	}

	fileLedger.ledgerFile.Close()
}

func loadFileLedger() {
	indexFile, err := os.OpenFile("./data/ledger", os.O_RDWR|os.O_CREATE, 0644)
	if err != nil {
		panic(err)
	}

	scanner := bufio.NewScanner(indexFile)

	dataFiles := []string{}
	for scanner.Scan() {
		dataFiles = append(dataFiles, scanner.Text())
	}

	if err := scanner.Err(); err != nil {
		fmt.Println(err)
	}

	fileLedger = FileLedger{
		ledgerFile: indexFile,
		ledger:     dataFiles,
		loaded:     true,
	}
}

func getDataIndex() []string {
	return fileLedger.ledger
}

func addFileToLedger(fileName string) {
	fileLedger.ledger = append(fileLedger.ledger, fileName)
	fileLedger.ledgerFile.Write([]byte(fileName + "\n"))
	fileLedger.ledgerFile.Sync()
}

func writeDataFile(table *SSTable) {
	if !fileLedger.loaded {
		panic("databaseFileStructure is not loaded")
	}

	fileName := fmt.Sprintf("%v", time.Now().UnixNano())

	err := os.WriteFile(fmt.Sprintf("data/%v", fileName), *table.Blocks, 0644)
	addFileToLedger(fileName)

	if err != nil {
		panic(err)
	}
}
```


## Conclusion
In this part we implemented a WAL to make sure data in the memtable is crash resistant. We also added the ability to store and search multiple SSTables on disk. It's starting to look like an actual database! 

In the next part we'll work on a very important feature: compacting tables. We'll implement a background process that automatically scans and compacts SSTables to reclaim disk space and improve lookup speeds.

