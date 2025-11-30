---
title: "Пишем партиционированный кеш на go map (x3 быстрее стандартной map)"
author: "Vadim Inshakov"
lang: "ru"
---


![](https://cdn-images-1.medium.com/max/2048/1*wkaFh4Dy6A_gSnZCIBlr1Q.png)

Представьте, что вы делаете inmemory-кеш, который должен работать под высокой нагрузкой. При этом основная нагрузка приходится на запись. Для этой задачи отлично подходит go map, защищенная мьютексами.

Но что будет, если, скажем, ваш кеш работает на мощном сервере с 64-мя ядрами и ваш go-рантайм создает 64 потока, каждый из которых потенциально может обращаться к вашему кешу? В таком случае мы столкнемся с множественными системными вызовами и значительными задержками из-за блокировок: когда один поток записывает данные, остальные вынуждены ждать.

Решение: разделить go map на отдельные партиции или сегменты, каждый из которых будет блокироваться независимо от других. Это позволит каждому потоку обращаться к своей партиции без необходимости блокировать доступ к остальным, тем самым оптимизируя процесс записи и обеспечивая высокую производительность даже в условиях высокой нагрузки.

```go
type PartitionedMap struct {
   partsnum   uint           // количество партиций
   partitions []*partition   // сами партиции
   finder     partitioner    // абстрактный поисковик партиций
}
```

Итак, мы будем хранить сами партиции, некий объект, позволяющий находить партиции для ключа, и количество партиций (как будет показано ниже, количество партиций нужно хранить, чтобы мэтчить ключи с патрициями).

Партиция — это отдельный мапа с мьютексом:

```go
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
```

Поисковик по партициям (partitioner) должен для каждого отдельного ключа возвращать индекс партиции:

```go
type partitioner interface {
   Find(key string) (uint, error)
}
```

Самый простой способ реализации этого интерфейса — алгоритм деления с остатком:

```go
type hashSumPartitioner struct {
   partitionsNum uint
}
    
func (h *hashSumPartitioner) Find(key string) (uint, error) {
   hashSum := crc32.ChecksumIEEE([]byte(key))
    
   return uint(hashSum) % h.partitionsNum, nil
}
```

Здесь для примера используется чексумма ключа crc32, но можно использовать любую функцию хеширования, главное чтобы она работала быстро. Полученное при хешировании значение делится на количество партиций и в итоге мы получаем индекс партиции, куда можно положить (или откуда можно достать) некоторый ключ.

Допустим, если у вас 3 партиции

```go
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
```

И ваша функция хеширования для ключа `testkey` выдала 125, то мы найдем нужную партицию так `125%3=41` (остаток 2) , следовательно, ключ testkeyпринадлежит партиции с индексом 2 (part2).

Теперь мы для любого ключа можем определить партицию:

```go
func (c *PartitionedMap) Set(key string, value any) error {
   // находим индекс партиции
   partitionIndex, err := c.finder.Find(key)
   if err != nil {
      return err
   }
    
   // достаем партицию из слайса
   partition := c.partitions[partitionIndex]
   // записываем ключ в партицию
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
```

Всё просто! Напишем бенчмарк, чтобы сравнить нашу реализацию со стандартной мапой.

**BenchmarckStd** — бенчмарк для стандартной мапы с мьютексом.

**BenchmarkSyncStd** — бенчмарк для sync.Map.

**BenchmarkPartiotioned** — бенчмарк партиционированной мапы.

```go
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
```

Результаты (на 6-ядерном процессоре):

```go
go test -bench=. -benchtime=3s
    
goos: darwin
goarch: amd64
pkg: github.com/vadimInshakov/partmap
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
    
BenchmarkStd/set_std_concurrently-12                  3289076   1332 ns/op
BenchmarkSyncStd/set_sync_map_std_concurrently-12     2408612   1691 ns/op
BenchmarkPartitioned/set_partitioned_concurrently-12  13536134  408.6 ns/op
```

Партиционированная мапа в **3.25** раз быстрее мапы на мьютексах и в **4.13** раз быстрее sync.Map.

Все исходники доступны здесь: [https://github.com/vadiminshakov/partmap](https://github.com/vadimInshakov/partmap)
