---
title: "Building a database (part 2): Write-ahead logging and compaction"
date: "2025-08-04"
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

All the code from this part can be found [here](https://github.com/DaanWillems/db/tree/main/parts/part2)

## Combining the Memtable and SSTable

[In the previous part]({{< relref "posts/building-a-database-part1-memtable-sstable.md" >}}), we coded a simple Memtable and SSTable to store data. The next step is to combine these two parts behind one interface. This will become the 'storage engine', the component of the database that is in charge of writing, reading and organizing on disk.

For now the behaviour of the storage engine can be described as follows:

If we insert into the database, it should be written to the memtable. If the memtable exceeds a given size, it should be flushed to disk. When querying we should first look into the memtable, and only then into the SSTable on disk.

### Cleaning up

Before moving on, we'll clean up the code a bit. All code files relating to the storage engine are moved into a package. We'll create a storage_engine.go file that will become the entrypoint into the package. All functions not in this file will be made private.

In the storage_engine.go file we'll add logic for starting the storage engine

```go
type Config struct {
	MemtableSize  int //Threshold of entries before flushing to disk
	DataDirectory string
	blockSize     int
}

var memtable Memtable
var config Config

func InitializeStorageEngine(cfg Config) {
	memtable = NewMemtable()
	config = cfg
}
```
The MemtableSize is the maximum number of entries the memtable may contain before flushing to disk. At a later point it will make more sense to look at the total number of bytes in the table, as entries can vary in size.

Let's write the logic for inserting data into the storage engine.

```go
func Insert(id int, values []string) {
	entry := Entry{
		id:      []byte{byte(id)},
		value:   value,
		deleted: false,
	}

	memtable.Insert(entry)

	if memtable.entries.Len() >= config.MemtableSize {
		writer := newSSTableWriterFromPath("data.db")
		err := writer.writeFromMemtable(memtable)
		if err != nil {
			panic(err)
		}

		memtable = newMemtable() // Reset memtable after flushing
	}
}
```
The insert function is also responsible for flushing and refreshing the memtable when it exceeds the maximum number of entries

```go
func Query(id []byte) ([]byte, error) {
	//First check in the memtable
	entry := memtable.Get(id)

	if entry != nil {
		return entry.values[0], nil
	}

	//For now we only have the capability of writing and reading a single sstable from disk
	reader := newSSTableReaderFromPath(fmt.Sprintf("./test.db"))
	entry, err = reader.scan(id)

	if err != nil {
		panic(err)
	}

	return entry.value, nil
}
```
The query function first looks into the memtable, and only after that into the SSTable on disk

## Write-ahead logging
The database has a basic level of persistence now. But if the program crashes or is terminated before writing the memtable to disk, this data is lost.

To avoid this problem we can keep a separate file on disk to which all database operations are logged first, called a WAL (Write-ahead log). Because this file is append only, it'll be very quick. Each time an entry gets inserted, we'll write it to the WAL first, and only then insert it into the memtable.

Should our program crash, we can read and 'replay' the WAL to restore the state of the memtable. Once the memtable gets written to disk we can discard the WAL.

The format for writing to the WAL is very simple, we'll serialize each entry and write the length of content + the content of the entry. This will also allow us to easily read it back. We'll also need a method for clearing the WAL once the memtable gets written.

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

func writeEntryToWal(entry Entry) {
	len, content := entry.Serialize()
	wal_file.Write([]byte{byte(len)})
	wal_file.Write(content)
	wal_file.Sync()
}
```

### Replaying

When the application starts, it should try to open the WAL into read mode and restore the memtable. We'll start by reading the first byte to get the entry size, and then the entry which gets deserialized and inserted into the memtable. We continue until there are no more entries to read. Conveniently the memtable object is accessible to us as it's all in the database package.

```go
func replayWal(_wal_path string) {
	fd, err := os.Open(_wal_path)
	if err != nil {
		if os.IsNotExist(err) {
			return
		}
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

		entry := &Entry{}
		entry.deserialize(bufio.NewReader(bytes.NewBuffer(content)))

		memtable.insert(*entry)
	}
}
```

Now we only need to call the appropriate methods in the storage_engine.go file.
In initialize we'll try to replay the wal to restore the memtable, and then open it for writing:

```go
func InitializeStorageEngine(cfg Config) {
	memtable = newMemtable()
	replayWal(fmt.Sprintf("./%v/wal", cfg.DataDirectory))
	openWAL(fmt.Sprintf("./%v/wal", cfg.DataDirectory))
	config = cfg
}
```

Before inserting into the memtable, we'll insert into the WAL

```go
writeEntryToWal(entry)
memtable.Insert(entry)
```

## File manager
Right now the data is always written to the file 'data.db'. This works well, until we want to flush the memtable a second time. We need a system for maintaining multiple files on disk. The storage engine needs to know which files already exist and can be searched. It is also useful to know which file are currently opened by the database. 

We'll start by making a dedicated directory for storing data, we can also place the WAL in this directory. Instead of writing to 'data.db' when flushing to disk, a unique name is generated for each new file. This way we can store data in more than one file.

First, it makes sense to unify our file opening and closing. Right now we are also opening files all over the place. Lets create a filemanager two functions for opening read and write files and use those instead. These functions will also keep track of which files are currently opened. 

```go
type FileManager struct {
	openReadFiles  map[string]*os.File
	openWriteFiles map[string]*os.File
}

var fileManager FileManager

func initFileManager(path string) {
	fileManager = FileManager{
		loaded:         true,
		openReadFiles:  map[string]*os.File{},
		openWriteFiles: map[string]*os.File{},
	}
}

func (fileManager *FileManager) openWriteFile(path string) (*os.File, error) {
	if val, ok := fileManager.openWriteFiles[path]; ok {
		return val, nil
	}
	fd, err := os.OpenFile(path, os.O_RDWR|os.O_CREATE, 0644)

	if err != nil {
		return nil, err
	}

	fileManager.openWriteFiles[path] = fd

	return fd, nil
}

func (fileManager *FileManager) openReadFile(path string) (*os.File, error) {
	if val, ok := fileManager.openReadFiles[path]; ok {
		return val, nil
	}
	fd, err := os.Open(path)
	if err != nil {
		return nil, err
	}

	fileManager.openReadFiles[path] = fd

	return fd, nil
}

func (fileManager *FileManager) close() {
	if fileManager.loaded {
		fileManager.ledgerFile.Close()
	}

	for _, fd := range fileManager.openReadFiles {
		err := fd.Close()
		if err != nil {
			log.Fatal(err)
		}
	}

	for _, fd := range fileManager.openWriteFiles {
		err := fd.Close()
		if err != nil {
			log.Fatal(err)
		}
	}
}
```
We can now call fileManager.openReadFile(path) or fileManager.openWriteFile(path) and the filemanager will keep track of what files are open. 

In order to keep track of what files contain data, we have to do some additional book keeping. Each time we write a new SSTable to disk, we'll also add the new file name to a ledger file. When we want to look for data in the storage engine, we can first do a lookup to see which files are available, and search them until we find what we are looking for. 

ledger.go:
```go
type FileManager struct {
	loaded         bool
	ledgerFile     *os.File
	ledger         []string
	openReadFiles  map[string]*os.File
	openWriteFiles map[string]*os.File
}

var fileManager FileManager

func initFileManager(path string) {
	fileManager = FileManager{
		loaded:         true,
		openReadFiles:  map[string]*os.File{},
		openWriteFiles: map[string]*os.File{},
	}

	indexFile, err := fileManager.openWriteFile(path)
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

	fileManager.ledgerFile = indexFile
	fileManager.ledger = dataFiles
}

func (fileManager *FileManager) getDataIndex() []string {
	return fileManager.ledger
}

func (fileManager *FileManager) addFileToLedger(fileName string) {
	fileManager.ledger = append(fileManager.ledger, fileName)
	fileManager.ledgerFile.Write([]byte(fileName + "\n"))
	fileManager.ledgerFile.Sync()
}

func (fileManager *FileManager) writeDataFile(memtable *Memtable) {
	if !fileManager.loaded {
		panic("databaseFileStructure is not loaded")
	}

	fileName := fmt.Sprintf("%v", time.Now().UnixNano())
	writer := newSSTableWriterFromPath(fmt.Sprintf("./%v/%v", config.DataDirectory, fileName))
	err := writer.writeFromMemtable(memtable)

	if err != nil {
		panic(err)
	}

	fileManager.addFileToLedger(fileName)
}

```
The file manager is initialized in the storage engine:

```go
func InitializeStorageEngine(cfg Config) {
	memtable = newMemtable()
	initFileManager(fmt.Sprintf("./%v/ledger", cfg.DataDirectory))
	replayWal(fmt.Sprintf("./%v/wal", cfg.DataDirectory))
	openWAL(fmt.Sprintf("./%v/wal", cfg.DataDirectory))
	config = cfg
}
```

Let's also add a close function for neatly closing the database.
```go
func Close() {
	closeWAL()
	fileManager.close()
}
````

Finally let's update the insert and query functions so they use the ledger.

```go
func Insert(id int, value []byte) {
	entry := Entry{
		id:      IntToBytes(id),
		value:   value,
		deleted: false,
	}

	writeEntryToWal(entry)
	memtable.insert(entry)

	if memtable.entries.Len() >= config.MemtableSize {
		fileManager.writeDataFile(&memtable)
		memtable = newMemtable() // Reset memtable after flushing
		resetWAL()               //Discard the WAL
	}
}

func Query(id []byte) ([]byte, error) {
	//First check in the memtable
	entry := memtable.Get(id)

	if entry != nil {
		return entry.value, nil
	}

	for _, path := range fileManager.getDataIndex() {
		reader := newSSTableReaderFromPath(fmt.Sprintf("./data/%v", path))
		entry, err := reader.scan(id)

		if err != nil {
			panic(err)
		}

		if entry == nil {
			continue
		}

		return entry.value, nil
	}

	return nil, nil
}
```

## Compaction
One of the downsides of SSTables being immutable is that they can contain information that is no longer relevant. Perhaps the record has been deleted, or updated. This can cause the table to grow unnecessarily large. To address this issue we can compact tables together using a merge algorithm. During compaction we can discard irrelevant values.

We will implement an iterative merge algorithm using buffered I/O. The tables themselves are already sorted, which makes it a lot easier.
The algorithm will work as follows:
1. Open _n_ table files for reading, and 1 for writing
2. Read the first key from each table
3. Compare the keys, write the smallest key and corresponding entry to the output file. If the keys are the same, take the most recent entry.
4. Advance the reader for the tables, and repeat step 3 until all but one table is exhausted. 
5. Write the remaining entries from the non-exhausted table to the output file.
6. Close all files and delete the old tables.

There are several different strategies we can choose from that decide when tables get compacted. Usually compaction with one of these strategies will run in the background. For now we will just focus on implementing the merge algorithm. The next part will dive into compaction strategies. 

We implement a function 'getNextEntry', this will function accepts _n_ readers and returns the 'first' entry, sorted on ID value and insertion time. The function assumes that the readers are sorted from oldest to most recent. It also advances all readers approporiately. 

It works by iterating the readers, and peeking (not reading) the ID of the next entry. It keeps track of the lowest entry ID it has encountered in the _min_ and _outputReaders_ variables. If it encounters an entry with an identical ID of the entry with the lowest ID so far it, it adds the additional new reader to _outputReaders_. If however a new lowest ID is found, the state is cleared and only this most recent one gets stored.

Next, it advances all the readers available in _outputReaders_ and returns the smallest entry. This step discards older duplicate values.

```go
func getNextEntry(readers []*SSTableReader) (*Entry, []*SSTableReader) {
	var min []byte
	outputReaders := []*SSTableReader{} 
	emptyReaders := []*SSTableReader{}

	//Get reader with smallest key
	//Its assumed that readers are ordered oldest to newest
	for _, reader := range readers {
		id, err := reader.peekNextId()
		if checkEOF(err) {
			emptyReaders = append(emptyReaders, reader)
			continue
		}

		if min == nil {
			min = id
			outputReaders = append(outputReaders, reader)
		} else if bytes.Compare(id, min) == -1 {
			min = id
			outputReaders := []*SSTableReader{}
			outputReaders = append(outputReaders, reader)
		} else if bytes.Equal(id, min) {
			outputReaders = append(outputReaders, reader)
		}
	}

	var entry Entry

	for _, reader := range outputReaders {
		entry, _ = reader.readNextEntry() //use the latest (most recent) newest entry
	}

	return &entry, emptyReaders
}
```

This function gets called by compactNNTablen which writes the entries to the new table, and removes readers from the state if they are exhausted. If all but 1 readers are exhaused, this function writes the remaining entries from the last table to the output table.

```go
func compactNSSTables(inputs []*SSTableReader, output *SSTableWriter) error {
	for {
		entry, emptyReaders := getNextEntry(inputs)
		output.writeSingleEntry(entry)

		for _, emptyReader := range emptyReaders {
			for index, reader := range inputs {
				if reader == emptyReader {
					//Remove from map
					inputs = append(inputs[:index], inputs[index+1:]...)
				}
			}
		}

		if len(inputs) == 0 {
			return nil
		}

		if len(inputs) == 1 {
			for _, remainder := range inputs {
				for {
					entry, err := remainder.readNextEntry()
					if checkEOF(err) {
						return nil
					}
					if err != nil {
						return err
					}
					output.writeSingleEntry(&entry)
				}
			}
		}
	}
}
```
Merging tables is very performant as we can rely on the sorted nature of the tables. It only needs sequential IO to access all the values, which is super quick. 

To make sure this is working correctly, I have written a number of tests. As this is quite a lot of code to include in this post, I have decided to leave them out. However you can find them [here in the repository](https://github.com/DaanWillems/db/blob/main/parts/part2/storage/compact_test.go)


## Conclusion
In this part we implemented a WAL to make sure data in the memtable is crash resistant. We also added the ability to store & search multiple SSTables on disk, and an algorithm for compacting tables together. It's starting to look like an actual database!

In the next part we'll use the compaction algorithm to implement a background process that continually compacts new tables in the background using a compaction strategy.
