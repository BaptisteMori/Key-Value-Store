Whitepaper

In this file all the size are not coherent, for now. In fact they should follow the variables in the [Annexes](#Variables)
# Goal

A persistent key:value store on efficient on disk, with low memory usage to store TOAST (The Oversized-Attribute Storage Technique) values. It's not meant to be intensively requested due to the low memory usage. But the main preoccupation is still the read even for values of varying sizes on disk. It's also not meant to store big items. For future usage it should be optimize for value of a size $\le$ 64 Ko, but it can be use with bigger items. 
For the data that are heavier, they should be stored in another storage ( like a data-lake, s3, ... )

Characteristics:

- **Read Performance**: Optimize for high-throughput read operations
- **Disk Efficiency**: Minimize disk accesses and maintain locality
- **Memory Frugality**: Function effectively with limited memory cache
- **Scalability**: Support for billions of keys and terabytes of data
- **Flexibility**: Configurable thresholds for different deployment scenarios


# Structure overview

```
+-------------------------+
| Extent 0                |
| +---------------------+ |
| | KVStore             | |
| +---------------------+ | Extent : container for reservation for coniguous storage
| +------+       +------+ | KVStore: main file containing descriptor
| | idx0 | ..... | idxn | | idx    : Bitmap index
| +------+       +------+ | VSF    : Value store file
| +------+       +------+ | eVSF   : extended size of value store file
| | VSF0 | ..... | VSFm | |
| +------+       +------+ |
| +-------+     +-------+ |
| | eVSF0 | ... | eVSFp | |
| +-------+     +-------+ |
+-------------------------+
```
#TODO Is the extent 0 the only one to contain a KVStore file ? or the KVStore is not in the extent ?

## KVStore file

```
master_kvstore
indexs-n.idx
vsf-m.db
```

The master is the descriptor that contain all of the informations to navigate between the files.
```
+-------------------+ 0
| Header            | 
|                   |
+-------------------+ 4096
| Index Registrar   |
|                   |
+-------------------+ ?
| Value store       |
| File Ptr          |
+-------------------+ ?
| Free space Map    |
|                   |
+-------------------+ ?
```
#TODO how to store efficiently the indexes ?

```c
struct KVStore {
    FileHeader header;              // File metadata and configuration
    IndexRegistrar index_registrar; // B+ Tree structure for key indexing
    ValueStoreFilePtr values_ptr;   // Storage area for values of different sizes
    FreeSpaceMap free_space;        // Tracking of available space
}
```

### Header
```c
struct FileHeader {
    uint32_t magic;                  // Magic number "TKVS"
    uint16_t version;                // Version
    uint64_t creation_timestamp;     // Creation time
    uint64_t last_modified;          // Last modification time
    uint64_t max_inline_toast_size;  // Max size for inline storage (typically 64KB)
    uint64_t root_page_pointer;      // Pointer to B+ Tree root
    uint64_t free_space_map_pointer; // Pointer to free space map
    uint64_t first_extent_pointer;   // Pointer to first extent
    uint32_t flags;                  // Various flags
    uint8_t  reserved[64];           // Reserved for future use
};
```

### Index registrar
```c
struct IndexRegistrar{
	uint_t registrar; // the size of INDEX_ID_SIZE, max of the index_id
}
```

## Index file

For a simple implementation, a bitmap index which would store a for a given key a pointer to the value in a VSF file.
It's a hierarchic bitmap index, compose of 3 types of files.

```
+-------------------+
| Master index (L1) |
|                   |
+-------------------+
| Bitmap (L2)       |
|                   |
+-------------------+
| Pointer File (L3) |
|                   |
+-------------------+
```

Advantages:
- **Memory Efficient**: the L1 can be stored in memory
- **Quick Read**: the hierarchical structur help to make read with a complexity of O(1) or O(log n) depending on the implementation
- **Space Efficeint**: the bitmaps allow to represent efficiently presence or absence of keys
- **Scalability**: because the file can be seperated this can expend to billions of keys on multiple index


### Master Index
```
+-----------------------+
| Header                |
|                       |
+-----------------------+
|                       |
|+-------+-------+-----+|
|| L1[0] | L1[1] | ... ||
|+-------+-------+-----+|
+-----------------------+
``` 
#### Header
```c
struct L1Header{
	uint32_t magic;                  // BMIX
	uint16_t version;
	uint64_t creation_timestamp;     // Creation time
    uint64_t last_modified;          // Last modification time
    uint64_t total_keys;             // total number of keys inserted
    uint64_t max_key_value;          // max value for a key
    
    
}
```

#### L1 Entry
```c
struct L1Entry {
    uint32_t bitmap;       // 1 bit for each L2 bloc (32 blocs)
    uint32_t l2_file_id;   // ID of the file containing the L2 blocs
}
```
The bitmap in the L1Entry is used as a map of presence for the L2 bloc
### Bitmap L2
```
+-----------------------+
| Header                |
|                       |
+-----------------------+
|                       |
|+-------+-------+-----+|
|| L2[0] | L2[1] | ... ||
|+-------+-------+-----+|
+-----------------------+
```

#### Header
```c
struct L2Header{

}
```
#### L2 Entry
```c
struct L2Entry{
	uint32_t ptr_file_id; // ID of the pointer file
	uint32_t offset;      // offset in the pointer file
}
```
### Pointer file (L3)

```
+-----------------------+
| Header                |
|                       |
+-----------------------+
|                       |
|+-------+-------+-----+|
|| L3[0] | L3[1] | ... ||
|+-------+-------+-----+|
+-----------------------+
```

#### Header
```c
struct L3Header{

}
```

#### L3 Entry
```c
struct L3Entry{
	uint_t vsf_id;
	uint_t offset;
}
```



## Value store file ( VSF )

```
+-------------------+ 0
| Header            | 
|                   |
+-------------------+ 43
| Value store       |
| Region            |
+-------------------+ ( depending on filling )
| Free space Map    |
|                   |
+-------------------+ VSF_MAX_SIZE
```

```c
struct ValueStoreFile {
	VSFHeader header;
	ValueStore values[];
	FreeSpaceMap free_space;
}
```

#### Header
```c
struct VSFHeader {
	uint32_t magic;                  // Magic number "TKVS"
    uint16_t version;                // Version
    uint64_t creation_timestamp;     // Creation time
    uint64_t last_modified;          // Last modification time
    uint64_t free_space_map_pointer; // Pointer to free space map
    uint64_t first_extent_pointer;   // Pointer to first extent
    uint32_t flags;                  // Various flags
    uint8_t  reserved[64];           // Reserved for future use
}
```

#### Value store region
this is meant to save extent and chunk values
#TODO explain with more details
## eVSF
A VSF but larger.

## Value

```
Value Store
+--------+-------+---------+----+------ - -+
| Size   | Flags | Storage | Id | Data     |
|        |       | Type    |    |          |
+--------+-------+---------+----+------ - -+
|           Header              |  Content |
+-------------------------------+------ - -+
0B                             14B       max: MAX_TOAST_SIZE 
```

A Value Store contain either a DirectValue or a ChunkValue.
```c
struct ValueStore{
	ValueHeader header;
	uint8_t data[];
}
```
### Header

```c
struct ValueHeader {
    uint64_t size;        // Total size of value 
    uint8_t flags;        // Storage flags
    uint8_t storage_type; // Direct, chunked
    uint32_t value_id;    // Id of the value stored ( int 32 or 64, to determine) 
}
```
`flags`:
- Compression
- Chunked, Direct

The values are stored in different type structure depending on their size.

### Value types

#### Direct value

**Small values** ( $\le$ MAX_TOAST_SIZE ):
- Stored in VSF
- Sequential layout for efficient reading
- Light pointer structure for location

```c
struct DirectValue { 
    uint8_t data[];     // Inline data for small values 
}
```

#### Chunk value

**Large Values** ( $\gt$ MAX_TOAST_SIZE)
- Chunked into fixed-size segments (MAX_TOAST_SIZE, typically 64KB)
- Chunk table for efficient segment location
- Intelligent chunk grouping to minimize disk access

```c
struct ChunkedValue { 
    uint1_t is_last_chunk;  // flag to tell if it's the last chunk
    uint8_t data[];         // chunk of data 
    struct { 
        uint32_t vsf_id;    // VSF containing this chunk 
        uint32_t offset;    // Offset within the extent 
    } ptr_next_chunk;             // Array of chunk locations 
}
```
To be more efficient, the chunks should be contiguous

## Free Space Map
```c

```

#TODO Explain this

## Extent
contiguous space reservation to optimize search.

## Memory caching


# Implementation

## Read
```c
function read(key)
    vsf_pointer = read_index(key)
    value = read_vsf(vsf_pointer)
    return value
```


### Read index
```c
function read_index(key):
    // Extract the segments of the key
    l1_segment = (key >> (B - L1_BITS)) & L1_MASK
    l2_segment = (key >> (B - L1_BITS - L2_BITS)) & L2_MASK
    l3_segment = (key >> (B - L1_BITS - L2_BITS - L3_BITS)) & L3_MASK
    
    // Verify the L1 entry
    l1_entry = master_index.l1_entries[l1_segment]
    if ((l1_entry.bitmap & (1 << l2_segment)) == 0):
        return NULL  // No correspondace on this level
    
    // Load L2 file
    l2_file = load_L2(l1_entry.l2_file_id)
    l2_entry = l2_file.l2_entries[l2_segment]
    
    if ((l2_entry.bitmap & (1 << l3_segment)) == 0):
        return NULL  // No correspondace on this level
    
    // Load file pointer
    ptr_file = load_pointer(l2_entry.ptr_file_id)
    
    // Calcul the offset in the file pointer
    ptr_offset = l2_entry.offset + (l3_segment * sizeof(L3Entry))
    
    // Read the pointer entry
    l3_entry = ptr_file.read_offset(ptr_offset)
    
    return l3_entry
```


### Read VSF
```c 
function read_vsf(pointer):
    
```


## Write

When a new value is added, have a parameter ( like SHOW_INDEX_PATH ) that return the pointer for each level of the index and in the VSF file. In that way, if another storage want to point on it it can cut of some steps.

Steps:
- Find a place in a VSF to store the value
- write it and get the offset and the VSF Id
- Find the place of the pointer in the index
- Write the pointer and return the place in each part of the index

```c

```

#TODO implementation + description

# Theoretical limitations

# Annexes

## Variables

```
KEY_SIZE      : 64b   // size of the key in bit
INDEX_ID_SIZE : 8     // size of the index id
MAX_TOAST_SIZE: 64Ko  // Set at creation ( immutable )
MAX_CACHE_SIZE: 500Mo

VSF_MAX_SIZE  : 200Mo // size of a value store file
EVSF_MAX_SIZE : 1Go   // size of an extended values store file
ENABLE_EVSF   : 1     // 1=True, 0=False, accept TOAST of a size greater than
                      // MAX_TOAST_SIZE



```

## Index complexity
### Variables
- **B** : Size of IDs of the key (bits) - typically 64 bits
- **I** : Size of index ID (bits) - example 4 bits
- **P** : Taille d'une page (en octets) - typiquement 4096 ou 8192 octets
- **E** : Taille d'une entrée dans une page feuille (en octets)
- **N** : Facteur de branchement (nombre d'entrées par nœud interne)

### Max number of distinct index
$N_{index}​=2^I$

### Max number of key per index
$N_{key/index} = 2^{(B-I)}$

### Branch factor
Number of entries in internal nodes. Depending on the size of the size of the page:


## Possibilities

|Solution / Critère|Performance en lecture|Performance en écriture|Gestion des valeurs volumineuses|Complexité d'implémentation|Efficacité de l'espace|Durabilité|Adaptabilité|
|---|---|---|---|---|---|---|---|
|**Hybride Hash+B-Tree**|Bonne|Bonne|Bonne|Moyenne|Moyenne|Moyenne|Bonne|
|**LSM Tree**|Moyenne|Excellente|Excellente|Élevée|Bonne|Bonne|Excellente|
|**Append-only**|Moyenne|Excellente|Excellente|Faible|Faible (avant compaction)|Excellente|Moyenne|
|**Radix Tree**|Très bonne|Bonne|Moyenne|Élevée|Bonne|Moyenne|Bonne|
|**COLA**|Bonne|Bonne|Moyenne|Très élevée|Excellente|Moyenne|Bonne|




