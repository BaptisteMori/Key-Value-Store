Whitepaper
#TODO
# Goal
A key:value store on efficient on disk, with low memory usage to store TOAST (The Oversized-Attribute Storage Technique) values. It's not meant to be intensively requested due to the low memory usage. But the main preoccupation is still the read even for values of varying sizes on disk. It's also not meant to store big items. For future usage it should be optimize for value of a size $\le$ 64 Ko, but it can be use with bigger items. For the data that are heavier, they should be stored in another storage ( like a data-lake, s3, ... )

Characteristics:
- **Read Performance**: Optimize for high-throughput read operations
- **Disk Efficiency**: Minimize disk accesses and maintain locality
- **Memory Frugality**: Function effectively with limited memory cache
- **Scalability**: Support for billions of keys and terabytes of data
- **Flexibility**: Configurable thresholds for different deployment scenarios

# B+ Tree fundamentals

## Basics structure
[Wikipedia](https://en.wikipedia.org/wiki/B%2B_tree)
A B+ Tree is a balanced tree data structure that maintains sorted data and allows for efficient insertions, deletions, and searches. Key components:

- **Root Node**: Single entry point to the tree
- **Internal Nodes**: Store keys and pointers to child nodes
- **Leaf Nodes**: Store keys and values (or value pointers)
- **Node Linking**: Leaf nodes are linked for sequential access
#TODO a bit more explanation on the B+ Tree

#TODO add an image of a B+Tree

## Properties for Disk Optimization
- High branching factor (typically 100-1000 entries per node)
- Minimal height (3-4 levels even for billions of entries)
- Nodes sized to match disk page size (typically 4KB, 8KB, or 16KB)
- Sequential access capabilities through leaf node linking

## Performance Characteristics
- **Search**: O(log_B N) where B is the branching factor and N is the number of keys
- **Range Queries**: Efficient due to sequential leaf traversal
- **Disk Access Pattern**: Minimizes random I/O operations
- **Cache Utilization**: Internal nodes often stay in cache due to frequent access

# Structure overview

## KVStore

```
+-------------------+ 0
| Header            | 
|                   |
+-------------------+ 4096
| B+ Tree Indes     |
|                   |
+-------------------+
| Value store       |
| Region            |
+-------------------+
| Free space Map    |
|                   |
+-------------------+ 
```

```c
struct KVStore {
    FileHeader header;             // File metadata and configuration
    BPlusTreeIndex index;          // B+ Tree structure for key indexing
    ValueStore values;             // Storage area for values of different sizes
    FreeSpaceMap free_space;       // Tracking of available space
}
```

### Header 

## Value
### Header 
```c
struct ValueHeader { 
	uint64_t size; // Total size of value 
	uint8_t flags; // Storage flags
	uint8_t storage_type; // Direct, extent, or chunked 
}
```
`flags`:
- Compression
- Chunked
- Extended

The values are stored in different type structure depending on their size. 
### Value types

#### Direct value
**Small values** ( $\lt$ 4KB):
- Stored directly in data pages
- Retrieved in a single I/O operation
- Optimized for immediate access
```c
struct DirectValue { 
	ValueHeader header; 
	uint8_t data[]; // Inline data for small values 
}
```
#### Extent value
**Medium Values** (4KB-64KB)
- Stored in dedicated extents
- Sequential layout for efficient reading
- Light pointer structure for location
```c
struct ExtentValue { 
	ValueHeader header; 
	uint32_t extent_id; // ID of the extent containing the value 
	uint32_t offset; // Offset within the extent 
}
```

#### Chunk value
**Large Values** ( $\gt$ 64KB)
- Chunked into fixed-size segments (typically 8KB)
- Chunk table for efficient segment location
- Intelligent chunk grouping to minimize disk access
```c
struct ChunkedValue { 
	ValueHeader header; 
	uint16_t chunk_count; // Number of chunks 
	uint32_t chunk_size; // Size of each chunk (except possibly last) 
	struct { 
		uint32_t extent_id; // Extent containing this chunk 
		uint32_t offset; // Offset within the extent 
	} chunks[]; // Array of chunk locations 
}
```







## Memory caching






# Theoretical limitations
# Annexs

## Variables
```
MAX_TOAST_SIZE: 64Ko  // Set at creation ( immutable )
MAX_CACHE_SIZE: 500Mo

```

## Possibilities

| Solution / Critère      | Performance en lecture | Performance en écriture | Gestion des valeurs volumineuses | Complexité d'implémentation | Efficacité de l'espace    | Durabilité | Adaptabilité |
| ----------------------- | ---------------------- | ----------------------- | -------------------------------- | --------------------------- | ------------------------- | ---------- | ------------ |
| **Hybride Hash+B-Tree** | Bonne                  | Bonne                   | Bonne                            | Moyenne                     | Moyenne                   | Moyenne    | Bonne        |
| **LSM Tree**            | Moyenne                | Excellente              | Excellente                       | Élevée                      | Bonne                     | Bonne      | Excellente   |
| **Append-only**         | Moyenne                | Excellente              | Excellente                       | Faible                      | Faible (avant compaction) | Excellente | Moyenne      |
| **Radix Tree**          | Très bonne             | Bonne                   | Moyenne                          | Élevée                      | Bonne                     | Moyenne    | Bonne        |
| **COLA**                | Bonne                  | Bonne                   | Moyenne                          | Très élevée                 | Excellente                | Moyenne    | Bonne        |


