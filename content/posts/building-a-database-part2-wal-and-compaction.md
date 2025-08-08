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

[In the previous part]({{< relref "posts/building-a-database-part1-memtable-sstable.md" >}}), we coded a simple Memtable and SSTable to store data. The next step is two combine these two parts behind one interface. This will become the 'storage engine', the component of the database that is in charge of writing, reading and organizing in disk.

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
func Query(id []byte]) ([]byte, error) {
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
Right now the data is always written to the file 'data.db'. This works well, until we want to flush the memtable a second time. We need a system for maintaining multiple files on disk. The storage engine needs to know which files already exist and can be searched. It is also useful to which file are currently opened by the database. 

We'll start by making a dedicated directory for storing data, we can also place the WAL in this directory. Instead of writing to 'data.db' when flushing to disk, a unique name is generated for each new file. This way we can store data in more than one file.

In order to keep track of what files contain data, we have to do some book keeping. Each time we add a new file, we'll also add the new file name to a ledger file. When we want to look for data in the storage engine, we can first do a lookup to see which files are available, and search them until we find what we are looking for. 

Right now we are also opening files all over the place. Lets create 2 functions for opening read and write files and use those instead. These functions will also keep track of which files are currently opened. 

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

func (fileLedger *FileManager) close() {
	if !fileLedger.loaded {
		panic("File ledger is not loaded")
	}

	fileLedger.ledgerFile.Close()
}

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

func (fileLedger *FileManager) getDataIndex() []string {
	return fileLedger.ledger
}

func (fileLedger *FileManager) openWriteFile(path string) (*os.File, error) {
	if val, ok := fileLedger.openWriteFiles[path]; ok {
		return val, nil
	}
	fd, err := os.OpenFile(path, os.O_RDWR|os.O_CREATE, 0644)

	if err != nil {
		return nil, err
	}

	fileLedger.openWriteFiles[path] = fd

	return fd, nil
}

func (fileLedger *FileManager) openReadFile(path string) (*os.File, error) {
	if val, ok := fileLedger.openReadFiles[path]; ok {
		return val, nil
	}
	fd, err := os.Open(path)
	if err != nil {
		return nil, err
	}

	fileLedger.openReadFiles[path] = fd

	return fd, nil
}

func (fileLedger *FileManager) addFileToLedger(fileName string) {
	fileLedger.ledger = append(fileLedger.ledger, fileName)
	fileLedger.ledgerFile.Write([]byte(fileName + "\n"))
	fileLedger.ledgerFile.Sync()
}

func (fileLedger *FileManager) writeDataFile(memtable *Memtable) {
	if !fileLedger.loaded {
		panic("databaseFileStructure is not loaded")
	}

	fileName := fmt.Sprintf("%v", time.Now().UnixNano())
	writer := newSSTableWriterFromPath(fmt.Sprintf("./%v/%v", config.DataDirectory, fileName))
	err := writer.writeFromMemtable(memtable)

	if err != nil {
		panic(err)
	}

	fileLedger.addFileToLedger(fileName)
}

```
We replace all os.Open and os.OpenFile calls in the code with our new openWriteFile and openReadFile functions.

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
		id:      []byte{byte(id)},
		value:   value,
		deleted: false,
	}

	writeEntryToWal(entry)
	memtable.insert(entry)

	if memtable.entries.Len() >= config.MemtableSize {
		writeDataFile(&memtable)
		memtable = newMemtable() // Reset memtable after flushing
		resetWAL()               //Discard the WAL
	}
}

func Query(id int) ([]byte, error) {
	//First check in the memtable
	entry := memtable.Get([]byte{byte(id)})

	if entry != nil {
		return entry.value, nil
	}

	for _, path := range getDataIndex() {
		fd, err := os.Open(fmt.Sprintf("./data/%v", path))
		panicIfErr(err)

		entry, err := scanSSTable(bufio.NewReader((fd)), []byte{byte(id)})

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

## Conclusion
In this part we implemented a WAL to make sure data in the memtable is crash resistant. We also added the ability to store and search multiple SSTables on disk. It's starting to look like an actual database!

In the next part we'll work on a very important feature: compacting tables. We'll implement a background process that automatically scans and compacts SSTables to reclaim disk space and improve lookup speeds.
