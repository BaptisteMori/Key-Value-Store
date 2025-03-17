Whitepaper

#TODO To complete

# Goal

A persistent key:value store on efficient on disk, with low memory usage to store TOAST (The Oversized-Attribute Storage Technique) values. It's not meant to be intensively requested due to the low memory usage. But the main preoccupation is still the read even for values of varying sizes on disk. It's also not meant to store big items. For future usage it should be optimize for value of a size $\le$ 64 Ko, but it can be use with bigger items. 
For the data that are heavier, they should be stored in another storage ( like a data-lake, s3, ... )

Characteristics:

- **Read Performance**: Optimize for high-throughput read operations
- **Disk Efficiency**: Minimize disk accesses and maintain locality
- **Memory Frugality**: Function effectively with limited memory cache
- **Scalability**: Support for billions of keys and terabytes of data
- **Flexibility**: Configurable thresholds for different deployment scenarios

# B+ Tree fundamentals

## Basics structure

[Wikipedia](https://en.wikipedia.org/wiki/B%2B_tree) 
[[notes_extents]]
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

```
+-------------------------+
| Extent 0                |
| +---------------------+ |
| | KVStore             | |
| +---------------------+ | Extent : container for reservation for coniguous storage
| +------+       +------+ | KVStore: main file containing descriptor
| | idx0 | ..... | idxn | | idx    : B+Tree index
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
	uint_t registrar; // the size of INDEX_ID_SIZE
}
```


## Index B+ Tree file

The IDs/Keys are incremental, so theire is a need to save the unused keys.
The advantage is 

Idea of the access to a value:
```
ID/Key : 0001 xxxx xxxx xxxx 
|-> 0001 -> index 1
|-> find the pointeur in index 1 : VSFm, offset xxxx
|-> access to VSFm at offset xxxx
|-> return value
```

The goal of this index is given an id, find the right pointer.
The index file should only store pointer to the value in the right Value Store File and should not store value. The purpose of this index is to quickly find a pointer from an id.

```
+-------------------+     +-------------------+
| Index File 1      | ... | Index File N      |
| (0 - 10M keys)    |     | (90M - 100M keys) |
| +---------------+ |     | +---------------+ |
| | B+Tree Root   | |     | | B+Tree Root   | |
| +---------------+ |     | +---------------+ |
| | Internal Nodes| |     | | Internal Nodes| |
| +---------------+ |     | +---------------+ |
| | Leaf Nodes    | |     | | Leaf Nodes    | |
| +---------------+ |     | +---------------+ |
+-------------------+     +-------------------+
```


```c
struct IndexBplusTree{
	IndexHeader header;
	IndexStore index;
}
```


```
+------------------------------------------+ 0
| Header                                   |
+------------------------------------------+ 4096
| Page #1 (The root)                       |
| +--------------------------------------+ |
| | Header of the page                   | |
| | - Type (interne/leaf)                | |
| | - Number of keys                     | |
| | - Pointers to the parent/neigborg    | |
| +--------------------------------------+ |
| | Entries (InternalEntry or LeafEntry) | |
| | [for interne page]:                  | |
| |   key1 | Ptr_child1 | key2 | ...     | |
| | [for leaf page ]:                    | |
| |   key1 | VSF_ID1 | Offset1 | ...     | |
| +--------------------------------------+ |
| | Free space                           | |
| +--------------------------------------+ |
+------------------------------------------+ 4096 + page_size
| Page #2                                  |
| ...                                      |
+------------------------------------------+
| ...                                      |
+------------------------------------------+
| Page #N                                  |
| ...                                      |
+------------------------------------------+
```
- **IndexFileHeader**: L'en-tête contient des informations spécifiques à ce fichier d'index, notamment son `index_id` qui correspond aux bits de poids fort des clés qu'il contient. Par exemple, l'index avec ID=5 (0101 en binaire) contient toutes les clés dont les 4 bits de poids fort sont 0101.
- **Pages B+Tree**: Chaque page représente un nœud dans l'arbre B+. Les pages sont de taille fixe (généralement 4Ko ou 8Ko) pour s'aligner avec les secteurs du disque. Il existe deux types de pages:
    - **Pages internes**: Contiennent des paires clé-pointeur où chaque pointeur dirige vers une page enfant. Ces pages forment la structure de l'arbre et permettent la navigation rapide.
    - **Pages feuilles**: Contiennent les entrées finales avec des paires clé-valeur. La "valeur" ici est en fait un pointeur vers le stockage réel (VSF_ID + offset). Les pages feuilles sont chaînées entre elles pour permettre des parcours séquentiels efficaces.

### Pointer
```c
struct ValuePointer{
	uint16_t vsf_id;
	uint48_t offset;
}
```

### Header
```c
struct IndexFileHeader { 
	uint32_t magic; // Nombre magique "BIDX" 
	uint16_t version; // Version du format 
	uint8_t index_id; // ID de cet index (basé sur les bits de poids fort) 
	uint8_t flags; // Flags divers 
	uint64_t creation_timestamp; // Date de création 
	uint64_t last_modified; // Dernière modification 
	uint64_t key_count; // Nombre total de clés stockées 
	uint16_t page_size; // Taille d'une page en octets (ex: 4096) 
	uint16_t max_keys_per_leaf; // Nombre max de clés par page feuille 
	uint16_t max_keys_per_internal;// Nombre max de clés par page interne 
	uint16_t root_page_id; // ID de la page racine 
	uint32_t height; // Hauteur actuelle de l'arbre 
	uint32_t page_count; // Nombre total de pages 
	uint32_t free_page_count; // Nombre de pages libres 
	uint64_t first_free_page; // Pointeur vers la première page libre 
	uint8_t reserved[24]; // Espace réservé 
};
```

### Index Store
```c

```

#### B+Tree page
```c
struct BTreePage { 
	uint32_t page_id; // Identifiant unique de la page 
	uint8_t page_type; // Type: 1=feuille, 2=interne 
	uint8_t flags; // Flags divers 
	uint16_t key_count; // Nombre de clés dans cette page 
	uint32_t parent_page_id; // ID de la page parent (0 si racine) 
	
	/* Pour les pages feuilles uniquement */ 
	uint32_t next_leaf_id; // ID de la page feuille suivante (0 si dernière) 
	uint32_t prev_leaf_id; // ID de la page feuille précédente (0 si première) 
	
	/* Pour les pages internes uniquement */ 
	uint32_t leftmost_child_id; // ID de l'enfant le plus à gauche 
	uint16_t free_space_offset; // Début de l'espace libre 
	uint16_t free_space_size; // Quantité d'espace libre disponible 
	uint8_t data[]; // Espace pour les entrées (taille variable) 
};
```

```
+------------------------------------------+ 0
| Page header (BTreePage)                  |
| - page_id = 42                           |
| - page_type = 2 (interne)                |
| - key_count = 3                          |
| - parent_page_id = 7                     |
| - leftmost_child_id = 101                |
| - free_space_offset = 160                |
| - free_space_size = 3936                 |
+------------------------------------------+ 32
| InternalEntry #1                         |
| - key = 0x1000000000000000               |
| - child_page_id = 102                    |
+------------------------------------------+ 44
| InternalEntry #2                         |
| - key = 0x3000000000000000               |
| - child_page_id = 103                    |
+------------------------------------------+ 56
| InternalEntry #3                         |
| - key = 0x7000000000000000               |
| - child_page_id = 104                    |
+------------------------------------------+ 68
|                                          |
|            Free Space Map                |
|                                          |
+------------------------------------------+ 4096
```
Dans cet exemple, la page interne a l'ID 42 et contient 3 clés. Elle a un parent avec l'ID 7 et est un nœud interne de l'arbre.

- Le `leftmost_child_id` (101) pointe vers le sous-arbre contenant toutes les clés inférieures à la première clé (0x1000000000000000).
- La première entrée (`key=0x1000000000000000, child_page_id=102`) indique que toutes les clés supérieures ou égales à 0x1000000000000000 mais inférieures à 0x3000000000000000 se trouvent dans le sous-arbre dont la racine a l'ID 102.
- La deuxième entrée (`key=0x3000000000000000, child_page_id=103`) couvre la plage de 0x3000000000000000 à 0x7000000000000000.
- La troisième entrée (`key=0x7000000000000000, child_page_id=104`) couvre toutes les clés supérieures ou égales à 0x7000000000000000.

Cette organisation permet une recherche rapide (O(log n)) en divisant l'espace des clés à chaque niveau.
#### Internal Entry
```c
struct InternalEntry { 
	uint64_t key; // Separation Key
	uint32_t child_page_id; // ID of the child page at the right of the key 
};
```

#### Leaf structure
```c
struct LeafEntry {
    uint64_t key;              // Key of the value
    uint8_t  flags;            // Flags
    uint8_t  reserved;         // reserved
    uint16_t vsf_id;           // ID of the VSF file
    uint32_t offset;           // Offset in the VSF file
};
```

```
+------------------------------------------+ 0
| En-tête de page (BTreePage)             |
| - page_id = 101                          |
| - page_type = 1 (feuille)                |
| - key_count = 4                          |
| - parent_page_id = 42                    |
| - next_leaf_id = 105                     |
| - prev_leaf_id = 0                       |
| - free_space_offset = 192                |
| - free_space_size = 3904                 |
+------------------------------------------+ 32
| LeafEntry #1                             |
| - key = 0x0010000000000001               |
| - vsf_id = 3                             |
| - flags = 0x00                           |
| - offset = 245760                        |
+------------------------------------------+ 48
| LeafEntry #2                             |
| - key = 0x0020000000000042               |
| - vsf_id = 3                             |
| - flags = 0x00                           |
| - offset = 459264                        |
+------------------------------------------+ 64
| LeafEntry #3                             |
| - key = 0x0080000000000ABC               |
| - vsf_id = 5                             |
| - flags = 0x01 (compressed)              |
| - offset = 128000                        |
+------------------------------------------+ 80
| LeafEntry #4                             |
| - key = 0x00F0000000001234               |
| - vsf_id = 7                             |
| - flags = 0x00                           |
| - offset = 765432                        |
+------------------------------------------+ 96
|                                          |
|            Espace libre                  |
|                                          |
+------------------------------------------+ 4096
```
Cette page feuille a l'ID 101 et contient 4 clés. Elle est la feuille la plus à gauche (car `prev_leaf_id=0`) et a une feuille suivante avec l'ID 105.
- Chaque entrée contient une clé complète et un pointeur vers la valeur correspondante dans un fichier VSF.
- Par exemple, la troisième entrée (`key=0x0080000000000ABC`) indique que la valeur se trouve dans le fichier VSF #5 à l'offset 128000 et qu'elle est compressée (`flags=0x01`).
- Les clés sont triées pour permettre une recherche binaire efficace au sein de la page.
- Les pages feuilles sont chaînées (`next_leaf_id` et `prev_leaf_id`) pour faciliter les parcours séquentiels.

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




