# Go语言源码

## Go编译时的命令

go build -gcflags "-N -l" -o test test.go

## 进入main函数之前的工作

```GO
1:调用了runtime.args(SB)
func args(c int32,v **byte){
    argc=c
    argv=v
    sysargs(c,v)
}
2:调用了runtime.osinit(SB)(
    用于确定CPU Core的数量
func osinit(){
    ncpu=getproccount()
}

3:调用了runtime.schedinit(SB)
func schedinit(){
    sched.maxmcount=10000

    stackinit()
    mallocinit()
    mcommoninit(_g_,m)

    // 处理命令行参数和环境变量
    goargs()
    goenvs()

    //处理GODEBUG,GOTRACEBACK 调试相关的环境变量设置
    paredebugvars()

    //垃圾回收器初始化
    gcinit()

    //通过CPU Core 和GOMAXPROCS 环境变量确定 P 数量
    procs := int(ncpu)
        if n := atoi(gogetenv("GOMAXPROCS"));n>0{
                if n> _MaxGomaxprocs
                    n=_MaxGomaxprocs

                 }
            procs=n
}
```

4：调用了runtime.main

## 内存分配

```Go
// class  bytes/obj  bytes/span  objects  tail waste  max waste
//     1          8        8192     1024           0     87.50%
//     2         16        8192      512           0     43.75%
//     3         32        8192      256           0     46.88%
//     4         48        8192      170          32     31.52%
//     5         64        8192      128           0     23.44%
//     6         80        8192      102          32     19.07%
//     7         96        8192       85          32     15.95%
//     8        112        8192       73          16     13.56%
//     9        128        8192       64           0     11.72%
//    10        144        8192       56         128     11.82%
//    11        160        8192       51          32      9.73%
//    12        176        8192       46          96      9.59%
//    13        192        8192       42         128      9.25%
//    14        208        8192       39          80      8.12%
//    15        224        8192       36         128      8.15%
//    16        240        8192       34          32      6.62%
//    17        256        8192       32           0      5.86%
//    18        288        8192       28         128     12.16%
//    19        320        8192       25         192     11.80%
//    20        352        8192       23          96      9.88%
//    21        384        8192       21         128      9.51%
//    22        416        8192       19         288     10.71%
//    23        448        8192       18         128      8.37%
//    24        480        8192       17          32      6.82%
//    25        512        8192       16           0      6.05%
//    26        576        8192       14         128     12.33%
//    27        640        8192       12         512     15.48%
//    28        704        8192       11         448     13.93%
//    29        768        8192       10         512     13.94%
//    30        896        8192        9         128     15.52%
//    31       1024        8192        8           0     12.40%
//    32       1152        8192        7         128     12.41%
//    33       1280        8192        6         512     15.55%
//    34       1408       16384       11         896     14.00%
//    35       1536        8192        5         512     14.00%
//    36       1792       16384        9         256     15.57%
//    37       2048        8192        4           0     12.45%
//    38       2304       16384        7         256     12.46%
//    39       2688        8192        3         128     15.59%
//    40       3072       24576        8           0     12.47%
//    41       3200       16384        5         384      6.22%
//    42       3456       24576        7         384      8.83%
//    43       4096        8192        2           0     15.60%
//    44       4864       24576        5         256     16.65%
//    45       5376       16384        3         256     10.92%
//    46       6144       24576        4           0     12.48%
//    47       6528       32768        5         128      6.23%
//    48       6784       40960        6         256      4.36%
//    49       6912       49152        7         768      3.37%
//    50       8192        8192        1           0     15.61%
//    51       9472       57344        6         512     14.28%
//    52       9728       49152        5         512      3.64%
//    53      10240       40960        4           0      4.99%
//    54      10880       32768        3         128      6.24%
//    55      12288       24576        2           0     11.45%
//    56      13568       40960        3         256      9.99%
//    57      14336       57344        4           0      5.35%
//    58      16384       16384        1           0     12.49%
//    59      18432       73728        4           0     11.11%
//    60      19072       57344        3         128      3.57%
//    61      20480       40960        2           0      6.87%
//    62      21760       65536        3         256      6.25%
//    63      24576       24576        1           0     11.45%
//    64      27264       81920        3         128     10.00%
//    65      28672       57344        2           0      4.91%
//    66      32768       32768        1           0     12.50%

```

### 1:内存块

分为span和object（span面向内部管理，object面像对象分配）
span是由多个地址连续的页组成的大块内存
object是将span按特定大小切分成多个小块，每个小块可以去存储一个对象。

malloc.go
_PageShift =13
_PageSize =1 << _PageShift
_NumSizeClasses=67

```GO
type mspan struct {
next *mspan//链表前向指针，用于将span链接起来
prev *mspan//链表前向指针，用于将span链接起来
startAddr uintptr // 起始地址，也即所管理页的地址
npages    uintptr // 管理的页数
nelems uintptr // 块个数，也即有多少个块可供分配
allocBits  *gcBits //分配位图，每一位代表一个块是否已分配
allocCount  uint16     // 已分配块的个数
spanclass   spanClass  // class表中的class ID
 elemsize    uintptr    // class表中的对象大小，也即块大小
}
```

用于存储对象的object，按8字节倍数分为n种。
比如：大小为24的object可用来存储范围在17-24字节的对象。

有了管理内存的span的基本单位，还要有个数据结构来管理span。这个数据结构叫mcentral

```GO
type mcentral struct {
lock      mutex
spanclass spanClass
nonempty  mSpanList // list of spans with a free object, ie a nonempty free list
empty     mSpanList // list of spans with no free objects (or cached in an mcache)
// nmalloc is the cumulative count of objects allocated from
// this mcentral, assuming all spans in mcaches are
// fully-allocated. Written atomically, read under STW.
nmalloc uint64
}
```

### 内存分配的组件
  
  包括 mcache, mcentral, mheap

#### cache缓存

为了避免多线程申请内存时不断的加锁，Golang为每个线程分配了span的缓存，这个缓存即是cache。

```Go
type mcache struct {
alloc [67*2]*mspan // 按class分组的mspan列表
}
```

解释：
alloc为mspan的指针数组，数组大小为class总数的2倍。数组中每个元素代表了一种class类型的span列表，每种class类型都有两组span列表，第一组列表中所表示的对象中包含了指针，第二组列表中所表示的对象不含有指针，这么做是为了提高GC扫描性能，对于不包含指针的span列表，没必要去扫描。

根据对象是否包含指针，将对象分为noscan和scan两类，其中noscan代表没有指针，而scan则代表有指针，需要GC进行扫描。

mchache在初始化时是没有任何span的，在使用过程中会动态的从central中获取并缓存下来，跟据使用情况，每种class的span个数也不相同。上图所示，class 0的span数比class1的要多，说明本线程中分配的小对象要多一些。

#### mcentral

为所有mcache提供切分好的mspan资源。每个central保存一种特定大小的全局mspan列表，包括已分配出去的和未分配出去的。 每个mcentral对应一种mspan，而mspan的种类导致它分割的object大小不同。当工作线程的mcache中没有合适（也就是特定大小的）的mspan时就会从mcentral获取。

mcentral被所有的工作线程共同享有，存在多个Goroutine竞争的情况，因此会消耗锁资源。

```GO
type mcentral struct {
    // 互斥锁
    lock mutex 

    // 规格
    sizeclass int32 

    // 尚有空闲object的mspan链表
    nonempty mSpanList 

    // 没有空闲object的mspan链表，或者是已被mcache取走的msapn链表
    empty mSpanList 

    // 已累计分配的对象个数
    nmalloc uint64
}
```

#### mheap

当mcentral没有空闲的mspan时，会向mheap申请。而mheap没有资源时，会向操作系统申请新内存。mheap主要用于大对象的内存分配，以及管理未切割的mspan，用于给mcentral切割成小对象。

```GO
type mheap struct {
    lock mutex
    // spans: 指向mspans区域，用于映射mspan和page的关系
    spans []*mspan 
    // 指向bitmap首地址，bitmap是从高地址向低地址增长的
    bitmap uintptr
    // 指示arena区首地址
    arena_start uintptr 
    // 指示arena区已使用地址位置
    arena_used  uintptr
    // 指示arena区末地址
    arena_end   uintptr
    central [67*2]struct {
        mcentral mcentral
        pad [sys.CacheLineSize - unsafe.Sizeof(mcentral{})%sys.CacheLineSize]byte
    }
}
```

Go的内存分配器在分配对象时，根据对象的大小，分成三类：小对象（小于等于16B）、一般对象（大于16B，小于等于32KB）、大对象（大于32KB）。
大体上的分配流程：
32KB 的对象，直接从mheap上分配；
<=16B 的对象使用mcache的tiny分配器分配；
(16B,32KB] 的对象，首先计算对象的规格大小，然后使用mcache中相应规格大小的mspan分配；
如果mcache没有相应规格大小的mspan，则向mcentral申请
如果mcentral没有相应规格大小的mspan，则向mheap申请
如果mheap中也没有合适大小的mspan，则向操作系统申请

参看文献：
1：https://www.iminho.me/wiki/blog-27.html
2：<<Go语言学习笔记>>

### 细节补充

1：Go编译器支持逃逸分析，它会在编译期通过构建调用图来分析局部变量是否会被外部引用，从而决定是否可直接分配在栈上。
2：gcflags -m 可以输出编译优化的信息，其中包括内联和逃逸分析。
