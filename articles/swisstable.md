
## SwissTable: Structure and Benchmark

![](https://cdn-images-1.medium.com/max/2050/1*3KEwpIWMPuklEDi2nxSHwA.png)

A new map implementation is being discussed in the Go community — SwissTable.

## Disclaimer

* I am testing the SwissTable implementation from Dolt.

* At the time of writing, the implementation in the Go runtime has not been officially released, so details may differ.

* Consider this an exploration of the principles behind SwissTable.

Let’s examine how Dolt’s solution works, understand its structure, and identify potential weaknesses.

## SwissTable Repository

Dolt’s SwissTable implementation:
[https://github.com/dolthub/swiss](https://github.com/dolthub/swiss)

## SwissTable Structure

SwissTable does not use traditional buckets (at least in Dolt’s implementation). Instead, it is divided into data and metadata.

    // Map is an open-addressing hash map
    // based on Abseil's flat_hash_map.
    type Map[K comparable, V any] struct {
        ctrl     []metadata
        groups   []group[K, V]
        ...
    }

**Data**

Data is stored in a slice called groups:

    // group is a group of 16 key-value pairs
    type group[K comparable, V any] struct {
        keys   [groupSize]K
        values [groupSize]V
    }

**Metadata**

Metadata is represented by bitmasks:

    // metadata is the h2 metadata array for a group.
    // find operations first probe the controls bytes
    // to filter candidates before matching keys
    type metadata [groupSize]int8

## How SwissTable Works

Let’s consider how a key is added to the map.

1. **Hash the key**

   The key is hashed and split into two parts, hi and lo:

   ```go 
   hi, lo := splitHash(m.hash.Hash(key))
   ```


2. **Determine the group**

   Use the hi part to determine the data group (similar to a bucket) by dividing the hash modulo the number of groups:

   ```go
   g := probeStart(hi, len(m.groups))
   ```
   
   Here, g is the index of the group where the key-value pair will be stored. For example, it could be group 3.


3. **Find the position in the group**
   Next, metadata is used to locate the position within the group:

   ```go
   matches := metaMatchH2(&m.ctrl[g], lo)
   ```

   matches is a bitmask. For instance, 0b00100000 indicates that slot 2 is available for writing.


4. **Insert the key-value pair**

   Check the keys and write the data to the corresponding position:

   ```go
   for matches != 0 {     
   s := nextMatch(&matches)     
   if key == m.groups[g].keys[s] {         
   m.groups[g].keys[s] = key         
   m.groups[g].values[s] = value         
   return     
   }
   }
   ```

   In this example, the data is written to m.groups[3].keys[2] and m.groups[3].values[2].

It’s fairly straightforward. The complexity lies in the highly efficient SIMD instructions used by the algorithm, which we won’t delve into here.

## Rehashing is a bottleneck

When the map exceeds its limit, rehashing is triggered. Here’s the code:

    func (m *Map[K, V]) Put(key K, value V) {
        if m.resident >= m.limit {
            m.rehash(m.nextSize())
        }
        ...
    }

Rehashing involves:

* Creating new groups.

* Recomputing hashes.

* Rewriting the data.

Here’s a simplified version of the code:

    func (m *Map[K, V]) rehash(n uint32) {
        ...
        m.groups = make([]group[K, V], n)
        m.ctrl = make([]metadata, n)
        ...
        for g := range ctrl {
            for s := range ctrl[g] {
                c := ctrl[g][s]
                ...
                m.Put(groups[g].keys[s], groups[g].values[s])
            }
        }
    }

This blocking and non-concurrent code can become a bottleneck for write-heavy workloads. Each time the map exceeds its limit, significant time is spent on rehashing.

## Benchmark

### Comparing Standard Map and SwissTable

I tested write performance with the same initial size for both maps:

    func BenchmarkCompareMaps(b *testing.B) {
        m := make(map[string]int, 10)
        b.Run("set std concurrently", func(b *testing.B) {
            var wg sync.WaitGroup
            var mu sync.RWMutex
            for i := 0; i < b.N; i++ {
                wg.Add(1)
                i := i
                go func() {
                    mu.Lock()
                    m[fmt.Sprint(i)] = i
                    mu.Unlock()
                    wg.Done()
                }()
            }
            wg.Wait()
        })
    
       mSwiss := swiss.NewMap[string, int](10)
       b.Run("set swiss concurrently", func(b *testing.B) {
            var wg sync.WaitGroup
            var mu sync.RWMutex
            for i := 0; i < b.N; i++ {
                wg.Add(1)
                i := i
                go func() {
                    mu.Lock()
                    mSwiss.Put(fmt.Sprint(i), i)
                    mu.Unlock()
                    wg.Done()
                }()
            }
            wg.Wait()
        })
    }

Results:

    go test -run=X -bench=BenchmarkCompareMaps -benchtime=5s  
    goos: darwin  
    goarch: arm64  
    cpu: Apple M2 Pro  

    BenchmarkCompareMaps/set_std_concurrently-10      4212386              1354 ns/op  
    BenchmarkCompareMaps/set_swiss_concurrently-10    5483253              1649 ns/op

Not cool.

### Comparing with a Partitioned Map

Now, let’s compare it with a simple partitioned map — partmap:

    mPart := NewPartitionedMapWithDefaultPartitioner(100, 10)
    b.Run("set partitioned concurrently", func(b *testing.B) {
        var wg sync.WaitGroup
        for i := 0; i < b.N; i++ {
            wg.Add(1)
            i := i
            go func() {
                mPart.Set(fmt.Sprint(i), i)
                wg.Done()
            }()
        }
        wg.Wait()
    })

Results:

    BenchmarkCompareMaps/set_partitioned_concurrently-10  13336983   493.9 ns/op

The partitioned map performs significantly better for write-heavy workloads. The SwissTable implementation in the Go runtime might bring better performance.

## Resources

* [Benchmark Code](https://gist.github.com/vadiminshakov/81b91c4ac0d4ca2dfec35c1bb43b29af)

* [Partmap GitHub Repository](https://github.com/vadiminshakov/partmap)
