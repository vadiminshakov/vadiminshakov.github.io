
## Writing a Partitioned Cache Using Go Map (x3 Faster than the Standard Map)

![](https://cdn-images-1.medium.com/max/2048/1*y3HSIiJRa5UxtBFrczEv3Q.png)

Imagine you’re making an in-memory cache that needs to operate under high load, with the primary load focused on writing. For this task, a go map protected by mutex is ideal.

But what if, say, your cache operates on a powerful server with 64 cores, and your go-runtime creates 64 threads, each of which can potentially access your cache? In such a case, we would face multiple system calls and significant delays due to locks: when one thread writes data, the others are forced to wait.

The solution: split the go map into separate partitions or segments, each of which will lock independently of the others. This allows each thread to access its partition without the need to lock access to the others, thus optimizing the writing process and ensuring high performance even under high load.

    type PartitionedMap struct {
       partsnum   uint           // partitions number
       partitions []*partition   // partitions slice
       finder     partitioner    // abstract partitions finder
    }

So, we will store the partitions themselves, an object that allows finding partitions for a key, and the number of partitions (as will be shown below, we need to store the number of partitions to match keys with partitions).

A partition is a separate map with a mutex:

    type partition struct {
       stor map[string]any
       sync.RWMutex
    }
    
    func (p *partition) set(key string, value any) {
       p.Lock()
       p.stor[key] = value
       p.Unlock()
    }
    
    func (p *partition) get(key string) (any, bool) {
       p.RLock()
       v, ok := p.stor[key]
       if !ok {
          p.RUnlock()
          return nil, false
       }
       p.RUnlock()
       return v, true
    }

The partition finder (partitioner) should return the index of the partition for each individual key:

    type partitioner interface {
       Find(key string) (uint, error)
    }

The simplest way to implement this interface is the remainder division algorithm:

    type hashSumPartitioner struct {
       partitionsNum uint
    }
    
    func (h *hashSumPartitioner) Find(key string) (uint, error) {
       hashSum := crc32.ChecksumIEEE([]byte(key))
    
       return uint(hashSum) % h.partitionsNum, nil
    }

Here, for example, the crc32 checksum of the key is used, but any hash function can be used, as long as it works quickly. The value obtained from hashing is divided by the number of partitions, and as a result, we get the index of the partition where a certain key can be placed (or from where it can be retrieved).

Suppose you have 3 partitions:

    func NewPartitionedMap(partitioner partitioner, partsnum uint) *PartitionedMap {
       partitions := make([]*partition, 0, partsnum)
       for i := 0; i < int(partsnum); i++ {
          m := make(map[string]any)
          partitions = append(partitions, &partition{stor: m})
       }
       return &PartitionedMap{partsnum: partsnum, partitions: partitions, finder: partitioner}
    }
    
    pm := NewPartitionedMap(partitionerImpl, 3)
    // pm internals:
    // {
    //   partsnum = 3         
    //   partitions = []*partition{part1, part2, part3}
    //   finder     partitioner = ...
    // }

And if your hashing function for the key testkeyyielded 125, then we would find the needed partition like this: 125%3=41 (remainder 2). Therefore, the key testkeybelongs to the partition with index 2 (part2).

Now, for any key, we can determine the partition:

    func (c *PartitionedMap) Set(key string, value any) error {
       // find partition index
       partitionIndex, err := c.finder.Find(key)
       if err != nil {
          return err
       }
    
       // get partition from slice
       partition := c.partitions[partitionIndex]
       // write key to the partition
       partition.set(key, value)
    
       return nil
    }
    func (c *PartitionedMap) Get(key string) (any, error) {
       partitionIndex, err := c.finder.Find(key)
       if err != nil {
          return nil, err
       }
    
       partition := c.partitions[partitionIndex]
       value, ok := partition.get(key)
       if !ok {
          return nil, errors.New("no such data")
       }
    
       return value, nil
    }

It’s simple! Let’s write a benchmark to compare our implementation with the standard map.

**BenchmarkStd** — benchmark for the standard map with a mutex.

**BenchmarkSyncStd** — benchmark for sync.Map.

**BenchmarkPartitioned** — benchmark for the partitioned map.

    func BenchmarkStd(b *testing.B) {
       m := make(map[string]int)
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
    }
    
    func BenchmarkSyncStd(b *testing.B) {
       b.Run("set sync map std concurrently", func(b *testing.B) {
          var m sync.Map
          var wg sync.WaitGroup
          for i := 0; i < b.N; i++ {
             wg.Add(1)
             i := i
             go func() {
                m.Store(fmt.Sprint(i), i)
                wg.Done()
             }()
          }
          wg.Wait()
       })
    }
    
    func BenchmarkPartitioned(b *testing.B) {
       m := NewPartitionedMap(&hashSumPartitioner{1000}, 1000)
       b.Run("set partitioned concurrently", func(b *testing.B) {
          var wg sync.WaitGroup
          for i := 0; i < b.N; i++ {
             wg.Add(1)
             i := i
             go func() {
                m.Set(fmt.Sprint(i), i)
                wg.Done()
             }()
          }
          wg.Wait()
       })
    }

Results (on a 6-core processor):

    go test -bench=. -benchtime=3s
    
    goos: darwin
    goarch: amd64
    pkg: github.com/vadimInshakov/partmap
    cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
    
    BenchmarkStd/set_std_concurrently-12                  3289076   1332 ns/op
    BenchmarkSyncStd/set_sync_map_std_concurrently-12     2408612   1691 ns/op
    BenchmarkPartitioned/set_partitioned_concurrently-12  13536134  408.6 ns/op

The partitioned map is **3.25** times faster than the mutex-based map and **4.13** times faster than sync.Map.

All source code is available here:

[https://github.com/vadiminshakov/partmap](https://github.com/vadimInshakov/partmap)
