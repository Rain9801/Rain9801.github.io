---
layout: post
title: STL内存管理
date: 2017-06-13 10:00:00.000000000 +09:00

---

STL空间配置器总是隐藏在一切组件的背后（具体地说是指容器，container），默默工作默默付出。为什么不说 allocator 是内存配置器而说它是空间配置器呢？因为，空间不一定是内存，空间也可以是磁盘或其它辅助储存媒体。是的，你可以写一个 allocator，直接向硬盘取空间。
对象建构前的空间配置，和对象解构后的空间释放，由 < stl_alloc.h > 负责，SGI对此的设计哲学如下：

 1. 考虑多线程（multi-threads）状态。
 2. 考虑内存不足时的应变措施。
 3. 考虑过多“小型区块”可能造成的内存破碎（fragment）问题。

为了将问题控制在一定的复杂度内，以下的讨论以及所摘录的源码，皆排除多线程状态的处理。
考虑小型区块所可能造成的内存破碎问题，SGI 设计了双层级配置器，第一级配置器直接使用 malloc() 和 free()，第二级配置器则视情况采用不同的策略：当配置区块超过 128bytes，视之为“足够大”，便调用第一级配置器；当配置区块小于128bytes，视之为“过小”，为了降低额外负担，便采用复杂的 memory pool 整理方式，而不再求助于第一级配置器。
```
# ifdef __USE_MALLOC
...
typedef __malloc_alloc_template<0> malloc_alloc malloc_alloc malloc_alloc malloc_alloc;
typedef malloc_alloc alloc alloc alloc alloc; // 令 alloc 为第一级配置器
#else
...
// 令 alloc 为第二级配置器
typedef __default_alloc_template<__NODE_ALLOCATOR_THREADS, 0> alloc alloc alloc alloc;
#endif /* ! __USE_MALLOC */
```
alloc 并不接受任何 template 型别参数。

第一级配置器以 malloc(), free(), realloc() 等 C 函式执行实际的内存配置、释放、重配置动作，并实作出类似 C++ new-handler 7 的机制。是的，它不能直接运用 C++ new-handler 机制，因为它并非使用 ::operator new 来配置记内存。

所谓 C++ new handler 机制是，你可以要求系统在内存配置需求无法被满足时，唤起一个你所指定的函式。换句话说一旦 ::operator new 无法达成任务，在丢出std::bad_alloc异常状态之前，会先呼叫由客端指定的处理例程。此处理例程通常即被称为 new-handler。

第二级配置器多了一些机制，避免太多小额区块造成内存的破碎。小额区块带来的其实不仅是内存破碎而已，配置时的额外负担（overhead）也是一大问题。额外负担永远无法避免，毕竟系统要靠这多出来的空间来管理内存。但是区块愈小，额外负担所占的比例就愈大、愈显得浪费。

SGI 第二级配置器的作法是，如果区块够大，超过 128 bytes，就移交第一级配置器处理。当区块小于 128 bytes，则以内存池（memory pool）管理，此法又称为次层配置（sub-allocation）：每次配置一大块内存，并维护对应之自由链表（free-list）。下次若再有相同大小的内存需求，就直接从free-lists中拨出。如果客端释还小额区块，就由配置器回收到free-lists中— 是的，别忘了，配置器除了负责配置，也负责回收。为了方便管理，SGI 第二级配置器会主动将任何小额区块的内存需求量上调至8的倍数 （例如客端 要 求 30 bytes ， 就自动 调整 为 32bytes），并维护 16 个free-lists，各自管理大小分别为 8, 16, 24, 32, 40, 48, 56, 64, 72,80, 88, 96, 104, 112, 120, 128 bytes 的小额区块。free-lists的节点结构如下：

```
union obj obj obj obj {
union obj * free_list_link;
char client_data[1]; /* The client sees this. */
};
```

为了维护链表（lists），每个节点需要额外的指标（指向下一个节点），这不又造成另㆒种额外负担吗？你的顾虑是对的，但早已有好的解决办法。注意，上述 obj 所用的是 union，由于 union 之故，从其第一字段观之，obj 可被视为一个指针，指向相同形式的另一个 obj。从其第二字段观之，obj 可被视为一个指针，指向实际区块，一物二用的结果是，不会为了维护链表所必须的指针而造成内存的另一种浪费（我们正在努力节省内存的开销呢）。

身为一个配置器，__default_alloc_template 拥有配置器的标准介面函式allocate()。此函式首先判断区块大小，大于 128 bytes 就呼叫第一级配置器，小于 128 bytes 就检查对应的free list。如果free list之内有可用的区块，就直接拿来用，如果没有可用区块，就将区块大小上调至 8 倍数边界，然后调refill()。当它发现free list中没有可用区块了，就调用refill() 准备为free list重新填充空间。新的空间将取自内存池（ 经由chunk_alloc() 完成）。预设取得 20 个新节点（新区块），但万一内存池空间不足，获得的节点数（区块数）可能小于 20。

chunk_alloc()函式以end_free-start_free来判断内存池的水量。如果水量充足，就直接拨出 20 个区块传回给free list。如果水量不足以提供 20 个区块，但还足够供应一个以上的区块，就拨出这不足 20 个区块的空间出去。这时候其 pass by reference 的 nobjs 参数将被修改为实际能够供应的区块数。如果内存池连一个区块空间都无法供应，对客端显然无法交待，此时便需利用 malloc()从 heap 中配置内存，为内存池注入活水源头以应付需求。新水量的大小为需求量的两倍，再加上一个随着配置次数增加而愈来愈大的附加量。
 >这篇文章主要整理自<< STL源码剖析 >>，有哪里引用有什么不对，欢迎指正


