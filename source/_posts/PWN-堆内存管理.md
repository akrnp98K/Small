---
title: PWN & 堆内存管理
top_img: https://www.apple.com.cn/newsroom/images/values/health/Apple-Health-study-July-2022-hero.jpg.landing-regular_2x.jpg
cover: https://www.apple.com.cn/newsroom/images/values/health/Apple-Health-study-July-2022-hero.jpg.landing-regular_2x.jpg
date: 2022-07-28 13:52:51
tags:
    - 数据结构
    - 底层
    - 算法
categories: 
            - [底层]
            - [数据结构]
            - [算法]
---

# 分析堆的相关工具

在 Phrack 的一篇文章中 《Advanced Doug Lea's malloc exploits》, 有一小节讲到 Heap layout analysis 作者利用了 main_arena 这个静态全局变量, 进行 heap dump 工作, 这里需要注意的是, 需要安装 libc6-dbg 以获取 debugging symbols, 此细节部分请查看 参考资料/glibc的调试相关.

介绍几个工具, 用于堆空间分配的分析.

几个工具大同小异, 简单介绍下原理, 都是采用 python 的 gdb 的 API。

之后通过 cat /proc/PID/maps 获取 heap base, 通过 gdb 的 x/ 查看内存, 通过 debugging symbols 获取 main_arena 地址

``` url
https://github.com/cloudburst/libheap
https://github.com/Mipu94/peda-heap
https://github.com/hugsy/gef
https://github.com/pwndbg/pwndbg
```
+ ltrace:
通过 ltrace 函数跟踪库函数调用。大致原理是, 起一个进程执行命令后, 根据 PID 拿到可执行文件, 之后按照 ELF 解析可执行文件, 拿到符号列表, 之后使用 ptrace attach 到 PID 上, 并在所有函数符号上插入断点。

+ 通过 LD_PRELOAD 的 hook 方式跟踪内存分配函数, 这也是 Phrack 中 《Advanced Doug Lea's malloc exploits》 利用的方法, 缺点就是需要重新执行程序

# 堆内存分配(ptmalloc设计)的思考
>下面是个人想法

## 为什么需要 ptmalloc
首先内存的分配和回收很频繁的, 这也就是其他语言希望实现高效的 GC, 针对频繁的操作, 第一个想到的解决方法就是缓存, 这也就是为什么 ptmalloc 存在各种各样的缓冲区. 假如不存在缓冲区, 每次分配都需要触发系统调用贼慢. 接下来就要引出 ptmalloc 涉及到的几种缓存, 这里只是概念性的解释几种缓存, 具体会在下文详细介绍.

## Bins
为了避免每次触发系统调用, 首先想到的解决方法就是释放的内存暂时不归还给系统, 标记为空闲, 等下一次再需要相同大小时, 直接使用这块空闲内存即可. （存储结构是双向环链表, 类似 hash 表, hash 算法就是 chunk 的长度, 用双向环链表解决 hash 冲突)

这涉及到, 刚刚释放的内存什么时候加到 Bins ? 相邻的两个空闲 chunk 什么时候合并? 怎么合并?

## Top
另一个应该想到的就是, 可以先利用系统调用 brk() 分配一块比较大的内存作为缓存, 之后即使没有在 Bins 中也找不到, 也不需要每次触发系统调用, 直接切割这块大的内存即可.

这就涉及到 '这块大内存' 什么时候重新补充大小(不断切割会导致 top 变小)? 什么时候需要缩小(归还给系统)?

## Fastbins
Bins 和 Top 缓存是最基本的, 如果想要做进一步的优化, 其实就是更细分的缓存, 也就是更准确的命中缓存, 这里 Fastbins 存在的更具体的原因是 避免 chunk 重复切割合并.

如果了解过 Python 源码可能会更理解, 这里的 Fastbins 类似于 Python 中整数对象 PyIntObject 的小整数 small_ints, 这里也只是理念类似, small_ints 准确的说是预先初始化, 可以一直重复使用而不被释放.

再回到 Fastbins 的讨论, 对于长度很小的 chunk 在释放后不会放到 Bins, 也不会标记为空闲, 这就避免了合并, 下次分配内存时首先查找 Fastbins, 这就避免了切割.

## Unsorted bin
Unsorted 是更细粒度的缓存, 属于 '刚刚释放的内存'与 Bins 之间的缓存.

## last_remainder
这其实也是一个缓存, 是针对于切割时使用的, 大致就是希望一直切割同一个 chunk. 在遍历 Unsorted 时使用, 但是它的使用是有条件的.

以上是在ptmalloc 的缓存设计上的一些想法. 下面会具体介绍 ptmalloc 在进行堆内存用到的各种具体的数据结构.

# chunk 结构

贴出一段 glibc-2.19/malloc/malloc.c 中关于 chunk 的解释. 

`boundary tag` 边界标记, 关于它下文会进行介绍
`INTERNAL_SIZE_T` 头部损耗, 参考 `eglibc-2.19/malloc/malloc.c:299`, 其实就是 `size_t`.
``` c
eglibc-2.19/malloc/malloc.c:1094
/*
  -----------------------  Chunk representations -----------------------
*/


/*
  This struct declaration is misleading (but accurate and necessary).
  It declares a "view" into memory allowing access to necessary
  fields at known offsets from a given base. See explanation below.
*/

// 一个 chunk 的完整结构体
struct malloc_chunk {

  INTERNAL_SIZE_T      prev_size;  /* Size of previous chunk (if free).  */
  INTERNAL_SIZE_T      size;       /* Size in bytes, including overhead. */

  struct malloc_chunk* fd;         /* double links -- used only if free. */
  struct malloc_chunk* bk;

  /* Only used for large blocks: pointer to next larger size.  */
  struct malloc_chunk* fd_nextsize; /* double links -- used only if free. */
  struct malloc_chunk* bk_nextsize;
};


/*
   malloc_chunk details:

    (The following includes lightly edited explanations by Colin Plumb.)

    // chunk 的内存管理采用边界标识的方法, 空闲 chunk 的 size 在该 chunk 的 size 字段和下一个 chunk 的 pre_size 字段都有记录
    Chunks of memory are maintained using a `boundary tag' method as
    described in e.g., Knuth or Standish.  (See the paper by Paul
    Wilson ftp://ftp.cs.utexas.edu/pub/garbage/allocsrv.ps for a
    survey of such techniques.)  Sizes of free chunks are stored both
    in the front of each chunk and at the end.  This makes
    consolidating fragmented chunks into bigger chunks very fast.  The
    size fields also hold bits representing whether chunks are free or
    in use.

    An allocated chunk looks like this:

    // 正在使用的 chunk 布局
    chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Size of previous chunk, if allocated            | |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Size of chunk, in bytes                       |M|P|
      mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             User data starts here...                          .
      .                                                               .
      .             (malloc_usable_size() bytes)                      .
      .                                                               |
nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Size of chunk                                     |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

    // 几个术语规定, 'chunk' 就是整个 chunk 开头, 'mem' 就是用户数据的开始, 'Nextchunk' 就是下一个 chunk 的开头
    Where "chunk" is the front of the chunk for the purpose of most of
    the malloc code, but "mem" is the pointer that is returned to the
    user.  "Nextchunk" is the beginning of the next contiguous chunk.

    // chunk 是双字长对齐
    Chunks always begin on even word boundaries, so the mem portion
    (which is returned to the user) is also on an even word boundary, and
    thus at least double-word aligned.

    // 空闲 chunk 被存放在双向环链表
    Free chunks are stored in circular doubly-linked lists, and look like this:

    chunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Size of previous chunk                            |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    `head:' |             Size of chunk, in bytes                         |P|
      mem-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Forward pointer to next chunk in list             |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Back pointer to previous chunk in list            |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
      |             Unused space (may be 0 bytes long)                .
      .                                                               .
      .                                                               |
nextchunk-> +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
    `foot:' |             Size of chunk, in bytes                           |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

    // P 标志位不能放在 size 字段的低位字节, 用于表示前一个 chunk 是否在被使用, 如果为 0, 表示前一个 chunk 空闲, 同时 pre_size 也表示前一个空闲 chunk 的大小, 可以用于找到前一个 chunk 的地址, 方便合并空闲 chunk, 但 chunk 刚一开始分配时默认 P 为 1. 如果 P 标志位被设置, 也就无法获取到前一个 chunk 的 size, 也就拿不到前一个 chunk 地址, 也就无法修改正在使用的 chunk, 但是这是无法修改前一个 chunk, 但是可以通过本 chunk 的 size 获得下一个 chunk 的地址. 
    The P (PREV_INUSE) bit, stored in the unused low-order bit of the
    chunk size (which is always a multiple of two words), is an in-use
    bit for the *previous* chunk.  If that bit is *clear*, then the
    word before the current chunk size contains the previous chunk
    size, and can be used to find the front of the previous chunk.
    The very first chunk allocated always has this bit set,
    preventing access to non-existent (or non-owned) memory. If
    prev_inuse is set for any given chunk, then you CANNOT determine
    the size of the previous chunk, and might even get a memory
    addressing fault when trying to do so.

    Note that the `foot' of the current chunk is actually represented
    as the prev_size of the NEXT chunk. This makes it easier to
    deal with alignments etc but can be very confusing when trying
    to extend or adapt this code.

    The two exceptions to all this are

    // 这里的 the trailing size 是指下一个 chunk 的 pre_size, 因为 top 位于最高地址, 不存在相邻的下一个 chunk, 同时这里也解答了上面关于 top 什么时候重新填满
     1. The special chunk `top' doesn't bother using the
  trailing size field since there is no next contiguous chunk
  that would have to index off it. After initialization, `top'
  is forced to always exist.  If it would become less than
  MINSIZE bytes long, it is replenished.

     2. Chunks allocated via mmap, which have the second-lowest-order
  bit M (IS_MMAPPED) set in their size fields.  Because they are
  allocated one-by-one, each must contain its own trailing size field.

*/
```
`P (PREV_INUSE)` 标志位表示前一个 chunk 是否在使用, 0 为没有在使用.

`prev_size` 表示前一个 chunk 的大小, 仅在` P (PREV_INUSE) `为 0 时有效, 也就是前一个 chunk 为空闲状态.

`size` 表示该整个 chunk 大小, 并非 malloc 返回值.

`fd`, `bk`, `fd_nextsize`,` fd_nextsize `是对于空闲 chunk 而言, 对于正在使用的 chunk, 从当前位置开始就是 malloc 返回给用户可用的空间.

`fd`, `bk` 组成了` Bins `的双向环链表

对于空闲的 chunk 空间布局, 见上, 是环形双向链表. 存放在空闲 chunk 容器中.

关于 chunk 有一些操作, 判断前一个是否在使用, 判断下一个 chunk 是否正在使用, 是不是 `mmap` 分配的, 以及对标志位 P 等的操作.

# 边界标示
对于 chunk 的空间布局组织采用边界标示的方法, chunk 的存储是一段连续的内存, 其实就是 chunk 头部保存长度信息, 可以在适当的时候获取到前一个和后一个 chunk.

这里涉及到 chunk 到用户请求 mem 的想换转化操作, 以及对齐操作等. 

# 空间复用

对于正在使用 chunk, 它的下一个 chunk 的` prev_size `是无效的, 所以这块内存被当前 chunk 给借用了, 因此对于请求分配 chunk 大小分配公式是` chunk_size = (用户请求大小 + (2 - 1) * sizeof(INTERNAL_SIZE_T)) align to 2 * sizeof(size_t)`

最后参考 eglibc-2.19/malloc/malloc.c:44, 会指出一些默认参数值, 以及关于 chunk 的最小 size 和 对齐的相关说明. 这里列出一小部分.
``` c
Supported pointer representation:       4 or 8 bytes
  Supported size_t  representation:       4 or 8 bytes
       Note that size_t is allowed to be 4 bytes even if pointers are 8.
       You can adjust this by defining INTERNAL_SIZE_T

  Alignment:                              2 * sizeof(size_t) (default)
       (i.e., 8 byte alignment with 4byte size_t). This suffices for
       nearly all current machines and C compilers. However, you can
       define MALLOC_ALIGNMENT to be wider than this if necessary.

  Minimum overhead per allocated chunk:   4 or 8 bytes
       Each malloced chunk has a hidden word of overhead holding size
       and status information.

  Minimum allocated size: 4-byte ptrs:  16 bytes    (including 4 overhead)
              8-byte ptrs:  24/32 bytes (including, 4/8 overhead)

       When a chunk is freed, 12 (for 4byte ptrs) or 20 (for 8 byte
       ptrs but 4 byte size) or 24 (for 8/8) additional bytes are
       needed; 4 (8) for a trailing size field and 8 (16) bytes for
       free list pointers. Thus, the minimum allocatable size is
       16/24/32 bytes.

       Even a request for zero bytes (i.e., malloc(0)) returns a
       pointer to something of the minimum allocatable size.

       The maximum overhead wastage (i.e., number of extra bytes
       allocated than were requested in malloc) is less than or equal
       to the minimum size, except for requests >= mmap_threshold that
       are serviced via mmap(), where the worst case wastage is 2 *
       sizeof(size_t) bytes plus the remainder from a system page (the
       minimal mmap unit); typically 4096 or 8192 bytes.
```
翻译几个关键的点, chunk 的大小需要按照 Alignment 进行对齐, 每一个被分配的 chunk 都有一个字的头部消耗, 包含该 chunk 的大小以及状态信息, 具体会在 chunk 结构和边界标示说明.

# 空闲容器(缓存)

下面会介绍 ptmalloc 中存在的各种空闲容器


Bins
``` c
eglibc-2.19/malloc/malloc.c:1341
/*
   -------------------- Internal data structures --------------------

   All internal state is held in an instance of malloc_state defined
   below. There are no other static variables, except in two optional
   cases:
 * If USE_MALLOC_LOCK is defined, the mALLOC_MUTEx declared above.
 * If mmap doesn't support MAP_ANONYMOUS, a dummy file descriptor
     for mmap.

   Beware of lots of tricks that minimize the total bookkeeping space
   requirements. The result is a little over 1K bytes (for 4byte
   pointers and size_t.)
 */

/*
   Bins
    // Bins 就是由空闲 chunk - bin 组成数组, 每一个 bin 都是双向链表. Bin 存放是整理过的 chunks, 并且 bin 中合并过的空闲 chunk 是不存在相邻的, 所以 bin 中的每一个 chunk 都是可以被使用, 并且都是紧挨着正在使用的 chunk 或者 heap  内存末尾.
    An array of bin headers for free chunks. Each bin is doubly
    linked.  The bins are approximately proportionally (log) spaced.
    There are a lot of these bins (128). This may look excessive, but
    works very well in practice.  Most bins hold sizes that are
    unusual as malloc request sizes, but are more usual for fragments
    and consolidated sets of chunks, which is what these bins hold, so
    they can be found quickly.  All procedures maintain the invariant
    that no consolidated chunk physically borders another one, so each
    chunk in a list is known to be preceeded and followed by either
    inuse chunks or the ends of memory.

    // bins 中的 chunk 是按照大小排序的. FIFO, small bins 是不存在按大小排序的, 因为每一个 small bin 都是相同 size 的. 但是对于 large bin 是需要按照顺序插入的. 这样可以在内存分配时很快查找到合适内存.
    Chunks in bins are kept in size order, with ties going to the
    approximately least recently used chunk. Ordering isn't needed
    for the small bins, which all contain the same-sized chunks, but
    facilitates best-fit allocation for larger chunks. These lists
    are just sequential. Keeping them in order almost never requires
    enough traversal to warrant using fancier ordered data
    structures.

    // FIFO, 从头部插入节点, 尾部取节点. 这样有个特定就是更容易内存的合并.
    Chunks of the same size are linked with the most
    recently freed at the front, and allocations are taken from the
    back.  This results in LRU (FIFO) allocation order, which tends
    to give each chunk an equal opportunity to be consolidated with
    adjacent freed chunks, resulting in larger free chunks and less
    fragmentation.

    To simplify use in double-linked lists, each bin header acts
    as a malloc_chunk. This avoids special-casing for headers.
    But to conserve space and improve locality, we allocate
    only the fd/bk pointers of bins, and then use repositioning tricks
    to treat these as the fields of a malloc_chunk*.
 */
```
ptmalloc 采用分箱式管理空闲 chunk, 也就是 Bins. Bins 本身就是一个数组, 每一个存放的是一个对应长度的 chunk 双向环链表的头结点和尾节点. 相同 Size 的 chunk 才能组成一个环,Bins 是按大小依次进行存放.

关于 Bins 为什么定义为 `mchunkptr bins[NBINS * 2 - 2] `而不是 `mchunkptr bins[NBINS * 4 - 2]`, 是如何少一倍的空间实现的双向链表,大致说一下, 对于双向环的的标志头节点, 它的 `prev_size` 和 `size` 是无用的, 所以直接省略, 但是还要把它当成正确的 chunk 结构. 这里的` trick `就在于 `bin_at` 宏, 返回了伪造的 fake chunk 的地址, 这里和` Double Free `以及 `unlink`绕过的利用手法类似.
``` c
/* addressing -- note that bin_at(0) does not exist */
#define bin_at(m, i) \
  (mbinptr) (((char *) &((m)->bins[((i) - 1) * 2]))               \
             - offsetof (struct malloc_chunk, fd))
```

举一个例子, 只摘取一部分, 完整的例子, 在下方的 ptmalloc 利用部分.
``` c
# 查看 unsorted bin 的地址, 其实也就是 bin[1] 的地址
(gdb) heap -b
===================================Heap Dump===================================

unsorted bin @ 0x7ffff7dd1b88
        free chunk @ 0x602160 - size 0x90

        free chunk @ 0x6020b0 - size 0x90

        free chunk @ 0x602000 - size 0x90
# 这里的 0x7ffff7dd1B78 也就是 bin_at 返回的地址, 返回了一个伪造的 chunk 的地址
# 其实这里的 fd 和 bk 才真正属于 bin[1] 的内容.
(gdb) p *(mfastbinptr)0x7ffff7dd1B78
$17 = {prev_size = 6300176, size = 0, fd = 0x602160, bk = 0x602000, fd_nextsize = 0x7ffff7dd1b88 <main_arena+104>, bk_nextsize = 0x7ffff7dd1b88 <main_arena+104>}
```
# small bins, large bins

对于 chunk `size < 512`, 是存放在 small bins, 有 64 个, 每个 bin 是以 8 bytes 作为分割边界, 也就相当于等差序列, 举个例子: small bins 中存放的第一个` chunk 双向环链表 `全部都是由 size 为 16 bytes 大小的 chunk 组成的, 第二个` chunk 双向环链表 `都是由 size 为 16+8 bytes 大小的 chunk 组成的. 但是对于 large bins, 分割边界是递增的, 举个简单例子: 前 32 个 large bins 的分割边界都是 64 bytes, 之后 16 个 large bins 的分割边界是 512 bytes. 以上仅为字长为 32 位的情况下, 具体请参考如下.

``` c
eglibc-2.19/malloc/malloc.c:1436
/*
   Indexing

    Bins for sizes < 512 bytes contain chunks of all the same size, spaced
    8 bytes apart. Larger bins are approximately logarithmically spaced:

    64 bins of size       8
    32 bins of size      64
    16 bins of size     512
     8 bins of size    4096
     4 bins of size   32768
     2 bins of size  262144
     1 bin  of size what's left

    There is actually a little bit of slop in the numbers in bin_index
    for the sake of speed. This makes no difference elsewhere.

    The bins top out around 1MB because we expect to service large
    requests via mmap.

    Bin 0 does not exist.  Bin 1 is the unordered list; if that would be
    a valid chunk size the small bins are bumped up one.
 */
```
large bin 有些特殊, 空闲 chunk 的存放需要排序, large_bin->bk 为最小 size 的 chunk, large_bin->fd 为最大 size 的 chunk.

# Fastbins

关于 Fastbins 的介绍, 可以参考:
``` c
/*
   Fastbins
    // 单向链表, LIFO 规则
    An array of lists holding recently freed small chunks.  Fastbins
    are not doubly linked.  It is faster to single-link them, and
    since chunks are never removed from the middles of these lists,
    double linking is not necessary. Also, unlike regular bins, they
    are not even processed in FIFO order (they use faster LIFO) since
    ordering doesn't much matter in the transient contexts in which
    fastbins are normally used.

    Chunks in fastbins keep their inuse bit set, so they cannot
    be consolidated with other free chunks. malloc_consolidate
    releases all chunks in fastbins and consolidates them with
    other free chunks.
 */
```

当进行内存分配时先从 Fastbins 中进行查找, 之后才在 Bins 进行查找; 释放内存时, 当`chunk size < max_fast `会先存放到 Fastbins.

另一个需要注意的点就是 Fastbins 的合并(清空), 也就是 `malloc_consolidate `这个函数的工作.

+ 何时会触发 malloc_consolidate(仅对 _int_malloc 函数而言) ?

+ small bins 尚未初始化

+ 需要 size 大于 small bins

+ `malloc_consolidate` 如何进行合并 ?

遍历 Fastbins 中的 chunk, 设置每个 chunk 的空闲标志位为 0, 并合并相邻的空闲 chunk, 之后把该 chunk 存放到 unsorted bin 中.

Fastbins 是单向链表, 可以通过 `fastbin->fd `遍历 Fastbins.

# unsorted bin

只有一个 `unsorted bin`, 进行内存分配查找时先在 Fastbins, small bins 中查找, 之后会在 unsorted bin 中进行查找, 并整理 unsorted bin 中所有的 chunk 到 Bins 中对应的 Bin. unsorted bin 位于` bin[1]`.

`unsorted_bin->fd` 指向双向环链表的头结点, `unsorted_bin->bk `指向双向环链表的尾节点, 在头部插入新的节点.

# top chunk

以下引用来自 《glibc内存管理ptmalloc源代码分析》。
>对于非主分配区会预先从 mmap 区域分配一块较大的空闲内存模拟 sub-heap,通过管 理 sub-heap 来响应用户的需求,因为内存是按地址从低向高进行分配的,在空闲内存的最 高处,必然存在着一块空闲 chunk,叫做 top chunk.当 bins 和 fast bins 都不能满足分配需 要的时候,ptmalloc 会设法在 top chunk 中分出一块内存给用户,如果 top chunk 本身不够大, 分配程序会重新分配一个 sub-heap,并将 top chunk 迁移到新的 sub-heap 上, 新的 sub-heap 与已有的 sub-heap 用单向链表连接起来,然后在新的 top chunk 上分配所需的内存以满足分配的需要,实际上,top chunk 在分配时总是在 fast bins 和 bins 之后被考虑,所以,不论 top chunk 有多大,它都不会被放到 fast bins 或者是 bins 中. top chunk 的大小是随着分配和回 收不停变换的,如果从 top chunk 分配内存会导致 top chunk 减小,如果回收的 chunk 恰好 与 top chunk 相邻,那么这两个 chunk 就会合并成新的 top chunk,从而使 top chunk 变大. 如果在 free 时回收的内存大于某个阈值,并且 top chunk 的大小也超过了收缩阈值,ptmalloc 会收缩 sub-heap,如果 top-chunk 包含了整个 sub-heap,ptmalloc 会调用 munmap 把整个 sub-heap 的内存返回给操作系统.

>由于主分配区是唯一能够映射进程 heap 区域的分配区,它可以通过 sbrk()来增大或是 收缩进程 heap 的大小,ptmalloc 在开始时会预先分配一块较大的空闲内存 (也就是所谓的 heap), 主分配区的 top chunk 在第一次调用 mallocd 时会分配一块(chunk_size + 128KB) align 4KB 大小的空间作为初始的 heap,用户从 top chunk 分配内存时,可以直接取出一块内 存给用户.在回收内存时,回收的内存恰好与 top chunk 相邻则合并成新的 top chunk,当该次回收的空闲内存大小达到某个阈值,并且 top chunk 的大小也超过了收缩阈值,会执行内 存收缩,减小 top chunk 的大小,但至少要保留一个页大小的空闲内存,从而把内存归还给 操作系统.如果向主分配区的 top chunk 申请内存,而 top chunk 中没有空闲内存, ptmalloc 会调用 sbrk()将的进程 heap 的边界 brk 上移, 然后修改 top chunk 的大小.

top chunk 位于最高地址.

# mmaped chunk

>当需要分配的 chunk 足够大,而且 fast bins 和 bins 都不能满足要求,甚至 top chunk 本 身也不能满足分配需求时,ptmalloc 会使用 mmap 来直接使用内存映射来将页映射到进程空 间.这样分配的 chunk 在被 free 时将直接解除映射,于是就将内存归还给了操作系统,再 次对这样的内存区的引用将导致 segmentation fault 错误.这样的 chunk 也不会包含在任何 bin 中.

# Last remainder

>Last remainder 是另外一种特殊的 chunk,就像 top chunk 和 mmaped chunk 一样,不会 在任何 bins 中找到这种 chunk.当需要分配一个 small chunk, 但在 small bins 中找不到合适 的 chunk, 如果 last remainder chunk 的大小大于所需的 small chunk 大小,last remainder chunk 被分裂成两个 chunk, 其中一个 chunk 返回给用户, 另一个 chunk 变成新的 last remainder chuk.

# malloc_state

只存在一个主分区, 但是允许多个非主分区, 主分配区域可以访问 heap 区域 和 mmap 区域, 非主分区只能访问 mmap 区域, 每次用 mmap 分配一块大小的内存当做 sub-heap, 用于模拟 heap. 每次进行内存分配必须加锁请求一个分配区.

``` c
eglibc-2.19/malloc/malloc.c:1663
/*
   ----------- Internal state representation and initialization -----------
 */

struct malloc_state
{
  /* Serialize access.  */
  mutex_t mutex;

  /* Flags (formerly in max_fast).  */
  int flags;

#if THREAD_STATS
  /* Statistics for locking.  Only used if THREAD_STATS is defined.  */
  long stat_lock_direct, stat_lock_loop, stat_lock_wait;
#endif

  /* Fastbins */
  mfastbinptr fastbinsY[NFASTBINS];

  /* Base of the topmost chunk -- not otherwise kept in a bin */
  mchunkptr top;

  /* The remainder from the most recent split of a small request */
  mchunkptr last_remainder;

  /* Normal bins packed as described above */
  mchunkptr bins[NBINS * 2 - 2];

  /* Bitmap of bins */
  unsigned int binmap[BINMAPSIZE];

  /* Linked list */
  struct malloc_state *next;

  /* Linked list for free arenas.  */
  struct malloc_state *next_free;

  /* Memory allocated from the system in this arena.  */
  INTERNAL_SIZE_T system_mem;
  INTERNAL_SIZE_T max_system_mem;
};
```

关于 `malloc_init_state `的定义在:
``` c
eglibc-2.19/malloc/malloc.c:1768
/*
   Initialize a malloc_state struct.

   This is called only from within malloc_consolidate, which needs
   be called in the same contexts anyway.  It is never called directly
   outside of malloc_consolidate because some optimizing compilers try
   to inline it at all call points, which turns out not to be an
   optimization at all. (Inlining it in malloc_consolidate is fine though.)
 */

static void
malloc_init_state (mstate av)
{
```
在 `eglibc-2.19/malloc/malloc.c:1741 `有一个已经初始化的主分配区 main_arena, 根据 ELF 的结构解析, 已初始化的全局变量存放在 .data 段, 下图作为实践.
``` c
# 33 是 Section 的 Index
λ : readelf -s /usr/lib/debug//lib/x86_64-linux-gnu/libc-2.23.so | grep main_arena 
   915: 00000000003c3b20  2192 OBJECT  LOCAL  DEFAULT   33 main_arena
# 对应 33 的 Section 恰好为 .data
λ : readelf -S /usr/lib/debug//lib/x86_64-linux-gnu/libc-2.23.so | grep .data
  [16] .rodata           NOBITS           0000000000174720  000002b4
  [23] .tdata            NOBITS           00000000003bf7c0  001bf7c0
  [29] .data.rel.ro      NOBITS           00000000003bf900  001bf7c0
  [33] .data             NOBITS           00000000003c3080  001bf7c0
```
 
# _int_malloc() 分析

先获取分配区指针, 这个过程设计到分配区初始化和分配区加锁, 之后使用 `_int_malloc` 进行核心的内存分配.

``` c
eglibc-2.19/malloc/malloc.c:3295
/*
   ------------------------------ malloc ------------------------------
 */

static void *
_int_malloc (mstate av, size_t bytes)
{
```

这一段分析来自 《glibc内存管理ptmalloc源代码分析》 但是对其中几个步骤做了补充和添加, 可以对比看一下 (以下针对 32 位字长)

ptmalloc 的响应用户内存分配要求的具体步骤为:

1. 获取分配区的锁, 为了防止多个线程同时访问同一个分配区, 在进行分配之前需要取得分配区域的锁. 线程先查看线程私有实例中是否已经存在一个分配区, 如果存 在尝试对该分配区加锁, 如果加锁成功, 使用该分配区分配内存, 否则, 该线程搜 索分配区循环链表试图获得一个空闲（没有加锁）的分配区. 如果所有的分配区都 已经加锁, 那么 ptmalloc 会开辟一个新的分配区, 把该分配区加入到全局分配区循 环链表和线程的私有实例中并加锁, 然后使用该分配区进行分配操作. 开辟出来的 新分配区一定为非主分配区, 因为主分配区是从父进程那里继承来的. 开辟非主分配区时会调用 mmap()创建一个 sub-heap, 并设置好 top chunk.

2. 将用户的请求大小转换为实际需要分配的 chunk 空间大小. 具体查看 request2size 宏 (malloc.c:3332)

3. 判断所需分配 chunk 的大小是否满足 chunk_size <= max_fast (max_fast 默认为 64B), 如果是的话, 则转下一步, 否则跳到第 5 步. (malloc.c:3340)

4. 首先尝试在 Fastbins 中查找所需大小的 chunk 分配给用户. 如果可以找到, 则分配结束. 否则转到下一步. (malloc.c:3340)

5. 判断所需大小是否处在 small bins 中, 即判断 chunk_size < 512B 是否成立. 如果 chunk 大小处在 small bins 中, 则转下一步, 否则转到第 7 步. (malloc.c:3377)

6. 根据所需分配的 chunk 的大小, 找到具体所在的某个 small bin, 从该 Bin 的尾部摘取一个恰好满足大小的 chunk. 若成功, 则分配结束, 否则, 转到 8. (malloc.c:3377)

7. 到了这一步, 说明需要分配的是一块大的内存, 于是, ptmalloc 首先会遍历 Fastbins 中的 chunk, 将相邻的空闲 chunk 进行合并, 并链接到 unsorted bin 中. 对于 Fastbins 的合并是由 malloc_consolidate 做处理. (malloc.c:3421)

8. 遍历 unsorted bin 中的 chunk, 如果请求的 chunk 是一个 small chunk, 且 unsorted bin 只有一个 chunk, 并且这个 chunk 在上次分配时被使用过(也就是 last_remainder), 并且 chunk 的大小大于 (分配的大小 + MINSIZE), 这种情况下就直接将该 chunk 进行切割, 分配结束, 否则继续遍历, 如果发现一个 unsorted bin 的 size 恰好等于需要分配的 size, 命中缓存, 分配结束, 否则将根据 chunk 的空间大小将其放入对应的 small bins 或是 large bins 中, 遍历完成后, 转入下一步. (malloc.c:3442)

9. 到了这一步说明需要分配的是一块大的内存, 并且 Fastbins 和 unsorted bin 中所有的 chunk 都清除干净 了. 从 large bins 中按照 “smallest-first, best-fit”(最小&合适, 也就是说大于或等于所需 size 的最小 chunk) 原则, 找一个合适的 chunk, 从中划分一块合适大小的 chunk 进行切割, 并将剩下的部分放到 unsorted bin, 若操作成功, 则分配结束, 否则转到下一步. (malloc.c:3576)

10. 到了这一步说明在对应的 bin 上没有找到合适的大小, 无论是 small bin 还是 large bin, 对于 small bin, 如果没有对应大小的 small bin, 只能 idx+1. 对于 large bin,在上一步的 large bin 并不一定能找到合适的 chunk 进行切割, 因为 large bins 间隔是很大的, 假如当前的 idx 的 large bin 只有一个 chunk, 但是所需 size 大于该 chunk, 这就导致找不到合适的, 只能继续 idx+1, 最后都需要根据 bitmap 找到之后第一个非空闲的 bin. 在这两种情况下找到的 bin 中的 chunk 一定可以进行切割或者全部分配(剩余的 size < MINSIZE) (malloc.c:3649)

11. 如果仍然都没有找到合适的 chunk, 那么就需要操作 top chunk 来进行分配了. 判断 top chunk 大小是否满足所需 chunk 的大小, 如果是, 则从 top chunk 中分出一块来. 否则转到下一步. (malloc.c:3749)

12. 到了这一步, 说明 top chunk 也不能满足分配要求, 所以, 于是就有了两个选择: 如果是主分配区, 调用 sbrk(), 增加 top chunk 大小；如果是非主分配区, 调用 mmap 来分配一个新的 sub-heap, 增加 top chunk 大小；或者使用 mmap()来直接分配. 在这里, 需要依靠 chunk 的大小来决定到底使用哪种方法. 判断所需分配的 chunk 大小是否大于等于 mmap 分配阈值, 如果是的话, 则转下一步, 调用 mmap 分配, 否则跳到第 13 步, 增加 top chunk 的大小. (malloc.c:3800)

13. 使用 mmap 系统调用为程序的内存空间映射一块 chunk_size align 4kB 大小的空间. 然后将内存指针返回给用户.

14. 判断是否为第一次调用 malloc, 若是主分配区, 则需要进行一次初始化工作, 分配一块大小为(chunk_size + 128KB) align 4KB 大小的空间作为初始的 heap. 若已经初始化过了, 主分配区则调用 sbrk()增加 heap 空间, 分主分配区则在 top chunk 中切割出一个 chunk, 使之满足分配需求, 并将内存指针返回给用户.

# _int_free() 分析

``` c
eglibc-2.19/malloc/malloc.c:3808
/*
   ------------------------------ free ------------------------------
 */

static void
_int_free (mstate av, mchunkptr p, int have_lock)
{
```
下面分析具体过程(针对 32 位字长)

1. free()函数同样首先需要获取分配区的锁, 来保证线程安全.

2. 判断传入的指针是否为 0, 如果为 0, 则什么都不做, 直接 return.否则转下一步.

3. 判断 chunk 的大小和所处的位置, 若 chunk_size <= max_fast, 并且 chunk 并不位于 heap 的顶部, 也就是说并不与 Top chunk 相邻, 则转到下一步, 否则跳到第 5 步.（因为与 top chunk 相邻的 chunk(fastbin) ,会与 top chunk 进行合并, 所以这里不仅需要判断大小, 还需要判断相邻情况）

4. 将 chunk 放到 Fastbins 中, chunk 放入到 Fastbins 中时, 并不修改该 chunk 使用状 态位 P.也不与相邻的 chunk 进行合并.只是放进去, 如此而已.这一步做完之后 释放便结束了, 程序从 free()函数中返回.

5. 判断所需释放的 chunk 是否为 mmaped chunk, 如果是, 则调用 munmap()释放 mmaped chunk, 解除内存空间映射, 该该空间不再有效.如果开启了 mmap 分配 阈值的动态调整机制, 并且当前回收的 chunk 大小大于 mmap 分配阈值, 将 mmap 分配阈值设置为该 chunk 的大小, 将 mmap 收缩阈值设定为 mmap 分配阈值的 2 倍, 释放完成, 否则跳到下一步.

6. 判断前一个 chunk 是否处在使用中, 如果前一个块也是空闲块, 则合并.并转下一步.

7. 判断当前释放 chunk 的下一个块是否为 top chunk, 如果是, 则转第 9 步, 否则转 下一步.

8. 判断下一个 chunk 是否处在使用中, 如果下一个 chunk 也是空闲的, 则合并, 并将合并后的 chunk 放到 unsorted bin 中.注意, 这里在合并的过程中, 要更新 chunk 的大小, 以反映合并后的 chunk 的大小.并转到第 10 步.

9. 如果执行到这一步, 说明释放了一个与 top chunk 相邻的 chunk.则无论它有多大, 都将它与 top chunk 合并, 并更新 top chunk 的大小等信息.转下一步.

10. 判断合并后的 chunk 的大小是否大于 FASTBIN_CONSOLIDATION_THRESHOLD（默认 64KB）, 如果是的话, 则会触发进行 Fastbins 的合并操作(malloc_consolidate), Fastbins 中的 chunk 将被遍历, 并与相邻的空闲 chunk 进行合并, 合并后的 chunk 会被放到 unsorted bin 中. Fastbins 将变为空, 操作完成之后转下一步.

11. 判断 top chunk 的大小是否大于 mmap 收缩阈值（默认为 128KB）, 如果是的话, 对于主分配区, 则会试图归还 top chunk 中的一部分给操作系统.但是最先分配的 128KB 空间是不会归还的, ptmalloc 会一直管理这部分内存, 用于响应用户的分配 请求；如果为非主分配区, 会进行 sub-heap 收缩, 将 top chunk 的一部分返回给操 作系统, 如果 top chunk 为整个 sub-heap, 会把整个 sub-heap 还回给操作系统.做 完这一步之后, 释放结束, 从 free() 函数退出.可以看出, 收缩堆的条件是当前 free 的 chunk 大小加上前后能合并 chunk 的大小大于 64k, 并且要 top chunk 的大 小要达到 mmap 收缩阈值, 才有可能收缩堆.

# 特殊的分配情况举例说明

下面几个的演示例子中没有使用到一些 heap 分析插件, 会在 ptmalloc 的利用那一步使用到 heap 分析的插件.

## 下面一段表明小于 Fastbins的size 在释放后不会进行合并, 如果使用 gdb 查看 chunk 信息可以看到 P 标志位为 1, 这里需要注意的是看下一个 chunk 的 P 标志位, 而不是当前 chunk 的标志位, 这里就不进行演示了.

``` c
#include<stdio.h>
#include<stdlib.h>
void main()
{
  void *m1 = malloc(24);
  int t = 0;
  void * ms[200];

  for(t = 0; t < 200; t++)
    ms[t] = malloc(120); // default fastbin size

  malloc(24);

  for(t = 0; t < 200; t++)
    free(ms[t]);
  void *m2 = malloc(24);
  printf("%p\n",m1);
  printf("%p\n",m2);
}

// result:
λ : gcc -g -o test2 test2.c && ./test2
0x17c2010
0x17c8450
```

## 下面例子表明, 当 fast bin 的相邻为空闲 chunk, 以及相邻 top chunk 的情况, 都不会进行合并, 但是对于 top chunk 的情况有些特殊.
``` c
/*
  TRIM_FASTBINS controls whether free() of a very small chunk can
  immediately lead to trimming. Setting to true (1) can reduce memory
  footprint, but will almost always slow down programs that use a lot
  of small chunks.

  Define this only if you are willing to give up some speed to more
  aggressively reduce system-level memory footprint when releasing
  memory in programs that use many small chunks.  You can get
  essentially the same effect by setting MXFAST to 0, but this can
  lead to even greater slowdowns in programs using many small chunks.
  TRIM_FASTBINS is an in-between compile-time option, that disables
  only those chunks bordering topmost memory from being placed in
  fastbins.
*/
```
当设置 TRIM_FASTBINS=1 fast bin 会与相邻的 top chunk 进行合并

``` c
λ : cat test5.c
#include<stdio.h>
#include<stdlib.h>
void main()
{
    void *m1 = malloc(500);
    void *m2 = malloc(40);
    malloc(1);
    void *m3 = malloc(80);
    free(m1);
    free(m2);
    void *m4 = malloc(40);

    free(m3);
    void *m5 = malloc(80);
    printf("m1, %p\n",m1);
    printf("m2, %p\n",m2);
    printf("m3, %p\n",m3);
    printf("m4, %p\n",m4);
    printf("m5, %p\n",m5);

}
// result:
λ : gcc -g -o test5 test5.c && ./test5
m1, 0x8b1010
m2, 0x8b1210
m3, 0x8b1260
m4, 0x8b1210
m5, 0x8b1260
```

## 下面的例子表明 small bin 在释放后会相邻合并的例子.

``` c
#include<stdio.h>
#include<stdlib.h>
void main()
{
  void *m1 = malloc(24);
  int t = 0;
  void * ms[200];

  for(t = 0; t < 200; t++)
    ms[t] = malloc(121); // small bin size

  malloc(24);

  for(t = 0; t < 200; t++)
    free(ms[t]);
  void *m2 = malloc(24);
  printf("%p\n",m1);
  printf("%p\n",m2);
}

// result:
λ : gcc -g -o test2 test2.c && ./test2
0xeab010
0xeab030
```
## 举例说明 malloc_consolidate 的作用, 以及如何触发 malloc_consolidate.
``` c
#include<stdio.h>
#include<stdlib.h>
void main()
{
        void *m0 = malloc(24);
        void *m1 = malloc(24);
        void *m2 = malloc(0x200);
        void *m3 = malloc(0x100);
        void *m4 = malloc(24);
        void *m5 = malloc(24);
        malloc(121);
        free(m0);
        free(m1);
        free(m2);
        free(m3);
        free(m4);
        free(m5);


        malloc(0x350);
        void *m6 = malloc(0x360);
        malloc(1210); // 触发 Fastbins 合并
        void *m7 = malloc(0x360);
        void *m8 = malloc(24);

        printf("m0,%p\n", m0);
        printf("m1,%p\n", m1);
        printf("m2,%p\n", m2);
        printf("m3,%p\n", m3);
        printf("m4,%p\n", m4);
        printf("m5,%p\n", m5);
        printf("m6,%p\n", m6);
        printf("m7,%p\n", m7);
        printf("m8,%p\n", m8);
}

result:
λ : gcc -g -o test3 test3.c && ./test3
m0,0x1bf7010
m1,0x1bf7030
m2,0x1bf7050
m3,0x1bf7260
m4,0x1bf7370
m5,0x1bf7390
m6,0x1bf77a0
m7,0x1bf7010
m8,0x1bf7380
```
## 下面举例说明, 当 small bins 和 large bins 没有找到对应合适 size 的 Bin, 需要切割的情况.
``` c
#include <stdio.h>
#include <stdlib.h>

void main()
{
    void * m1 = malloc(0x200);
    malloc(121);
    void * m2 = malloc(0x401);
    malloc(121);
    free(m2);
    void * m3 = malloc(24);
    free(m1);
    void * m4 = malloc(24);
    printf("m1, %p\n", m1);
    printf("m2, %p\n", m2);
    printf("m3, %p\n", m3);
    printf("m4, %p\n", m4);
    printf("sizeof(size_t) = %ld\n", sizeof(size_t));
}

result:
λ : gcc -g -o test1 test1.c && ./test1
m1, 0x1a66010
m2, 0x1a662b0
m3, 0x1a662b0 //切割 small bins
m4, 0x1a66010 //切割 large bins
sizeof(size_t) = 8
```
# exploit 在 ptmalloc 中

首先明确大部分的关注点, 是在 leak infomation 和 aa4bmo.

对于 leak infomation, 需要所 dump 的地址内存放关键信息, 比如: 释放后的 chunk 的 fd 和 bk.

对于 aa4bmo, 这一块在另一篇《PWN之堆触发》有完善的介绍和总结.

下面的一些分析实例会用到 heap 的分析插件, 并且会提到一些具体的实践以对应之前的理论.

# Leak Information (泄露关键信息)
Q: 什么是关键信息?

A: libc 地址, heap 地址

通过 ptmalloc 获得的内存 chunk 在释放后会变成上面提到的几种缓存类型, 这里主要提一下 Fastbins, Bins 能够泄漏什么关键信息.

分配区 `main_arena` 是已经初始化静态全局变量存放在 `libc.so.6` 的 `.data` 位置, 可以通过 `main_arena` 泄露 libc 的基址.

## 下面是一个关于 Fastbins 的例子, Fastbins 是单向链表, 通过 fd 指针进行遍历, 每次插入链表头位置, 可以通过已经释放的 Fastbin chunk 的 fd 指针 dump 到 heap 地址.

``` c
#include<stdio.h>                                                                                                                                                                                                  
#include<stdlib.h>                                                                                                                                                                                                 
#include<string.h>                                                                                                                                                                                                 
void main()                                                                                                                                                                                                        
{                                                                                                                                                                                                                  
    void * m1 = malloc(0x80-8);                                                                                                                                                                                    
    void * m2 = malloc(0x80-8);                                                                                                                                                                                    
    memset(m1, 65, 0x80-8);                                                                                                                                                                                        
    memset(m2, 65, 0x80-8);                                                                                                                                                                                        
    malloc(1);                                                                                                                                                                                                     
    free(m1);                                                                                                                                                                                                      
    free(m2);                                                                                                                                                                                                      
    printf("m1: %p\n", m1);                                                                                                                                                                                        
    printf("m2: %p\n", m2);                                                                                                                                                                                        
    printf("sizeof(size_t): %ld\n", sizeof(size_t));                                                                                                                                                               
}

# 主分配区
(gdb) P &main_arena 
$3 = (struct malloc_state *) 0x7ffff7dd1b20 <main_arena>
(gdb) p main_arena 
$2 = {mutex = 0, flags = 0, fastbinsY = {0x0, 0x0, 0x0, 0x0, 0x0, 0x0, 0x602080, 0x0, 0x0, 0x0}, top = 0x602120, last_remainder = 0x0, bins = {...more... }, binmap = {0, 0, 0, 0}, next = 0x7ffff7dd1b20 <main_arena>, next_free = 0x0, attached_threads = 1, system_mem = 135168, max_system_mem = 135168}
# 同上
(gdb) heap
===================================Heap Dump===================================

Arena(s) found:
         arena @ 0x7ffff7dd1b20
# Fastbins 在释放后, P 标志位不会被清空
(gdb) heap -l
===================================Heap Dump===================================

          ADDR             SIZE         STATUS
sbrk_base 0x602000
chunk     0x602000         0x80         (inuse)
chunk     0x602080         0x80         (inuse)
chunk     0x602100         0x20         (inuse)
chunk     0x602120         0x20ee0      (top)
sbrk_end  0x602001
# 查看 bins
(gdb) heap -b
===================================Heap Dump===================================
fast bin 6 @ 0x602080
        free chunk @ 0x602080 - size 0x80 
        free chunk @ 0x602000 - size 0x80 
# 通过观察源码和这里 Fastbins 的顺序应该可以发现 Fastbins 是头插入
(gdb) heap -f
====================================Fastbins====================================

[ fb 0 ] 0x7ffff7dd1b28  -> [ 0x0 ] 
[ fb 1 ] 0x7ffff7dd1b30  -> [ 0x0 ] 
[ fb 2 ] 0x7ffff7dd1b38  -> [ 0x0 ] 
[ fb 3 ] 0x7ffff7dd1b40  -> [ 0x0 ] 
[ fb 4 ] 0x7ffff7dd1b48  -> [ 0x0 ] 
[ fb 5 ] 0x7ffff7dd1b50  -> [ 0x0 ] 
[ fb 6 ] 0x7ffff7dd1b58  -> [ 0x602080 ] (128)
                              [ 0x602000 ] (128)
[ fb 7 ] 0x7ffff7dd1b60  -> [ 0x0 ] 
[ fb 8 ] 0x7ffff7dd1b68  -> [ 0x0 ] 
[ fb 9 ] 0x7ffff7dd1b70  -> [ 0x0 ]
# Fastbins 是根据 fd 指针进行遍历
(gdb) p *(mchunkptr)0x602080
$4 = {prev_size = 4702111234474983745, size = 129, fd = 0x602000, bk = 0x4141414141414141, fd_nextsize = 0x4141414141414141, bk_nextsize = 0x4141414141414141}
# 这里 dump 之前 chunk 的内容可以拿到 heap 的地址
(gdb) x/wx 0x602090
0x602090:       0x00602000
```
下面是一个关于 Bins 的例子, Bins 是双向环链表, 头插入, 可以通过已经释放的 Bin chunk 泄漏 libc 和 heap 地址.

这里需要理解一下由` malloc(0xB0-8)`; 的作用, 以及 Unstored bin 转为 small bins 的过程. 这里如果不清楚可以对应 libc 源码查看上面提到的 `_int_malloc()`的过程.

``` c
#include<stdio.h>                                                                                                                                                                                                  
#include<stdlib.h>                                                                                                                                                                                                 
#include<string.h>                                                                                                                                                                                                 
void main()                                                                                                                                                                                                        
{                                                                                                                                                                                                                  
    void * m1 = malloc(0x90-8);                                                                                                                                                                                    
    malloc(1);                                                                                                                                                                                                     
    void * m2 = malloc(0x90-8);                                                                                                                                                                                    
    malloc(1);                                                                                                                                                                                                     
    void * m3 = malloc(0xA0-8);                                                                                                                                                                                    
    malloc(1);                                                                                                                                                                                                     
    memset(m1, 65, 0x90-8);                                                                                                                                                                                        
    memset(m2, 65, 0x90-8);                                                                                                                                                                                        
    memset(m3, 65, 0xA0-8);                                                                                                                                                                                        

    free(m1);                                                                                                                                                                                                      
    free(m2);                                                                                                                                                                                                      
    free(m3);                                                                                                                                                                                                      
    malloc(0xB0-8);                                                                                                                                                                                                
    printf("m1: %p\n", m1);                                                                                                                                                                                        
    printf("m2: %p\n", m2);                                                                                                                                                                                        
    printf("m3: %p\n", m3);                                                                                                                                                                                        
    printf("sizeof(size_t): %ld\n", sizeof(size_t));                                                                                                                                                               
} 

λ : gdb -q test2
Reading symbols from test2...done.
(gdb) b 19
Breakpoint 1 at 0x4006ac: file test2.c, line 19.
(gdb) r
Starting program: /home/spiderzz/Desktop/pwn/malloc/test2 

Breakpoint 1, main () at test2.c:19
19          malloc(0xB0-8);
(gdb) heap
===================================Heap Dump===================================

Arena(s) found:
         arena @ 0x7ffff7dd1b20

# Unsorted bin 是双向环链表, 这里需要观察, 双向环链表的两个端点 chunk 的 FD 和 BK 的地址不同之处, 因为一个在 libc 的空间, 一个在 heap 的空间.
(gdb) heap -l
===================================Heap Dump===================================

          ADDR             SIZE         STATUS
sbrk_base 0x602000
chunk     0x602000         0x90         (F) FD 0x7ffff7dd1b78 BK 0x6020b0 
chunk     0x602090         0x20         (inuse)
chunk     0x6020b0         0x90         (F) FD 0x602000 BK 0x602160 
chunk     0x602140         0x20         (inuse)
chunk     0x602160         0xa0         (F) FD 0x6020b0 BK 0x7ffff7dd1b78 
chunk     0x602200         0x20         (inuse)
chunk     0x602220         0x20de0      (top)
sbrk_end  0x602001
(gdb) heap -b
===================================Heap Dump===================================

unsorted bin @ 0x7ffff7dd1b88
        free chunk @ 0x602160 - size 0xa0

        free chunk @ 0x6020b0 - size 0x90

        free chunk @ 0x602000 - size 0x90
# 这个也就是返回的 fake chunk 的地址, 这地址其实就是 bin_at 的返回值
(gdb) p *(mfastbinptr)0x7ffff7dd1B78
$1 = {prev_size = 6300192, size = 0, fd = 0x602160, bk = 0x602000, fd_nextsize = 0x7ffff7dd1b88 <main_arena+104>, bk_nextsize = 0x7ffff7dd1b88 <main_arena+104>}         
(gdb) n
20          printf("m1: %p\n", m1);
# 这里需要理解 Bins 的 FD 和 BK.
(gdb) heap -l
===================================Heap Dump===================================

          ADDR             SIZE         STATUS
sbrk_base 0x602000
chunk     0x602000         0x90         (F) FD 0x7ffff7dd1bf8 BK 0x6020b0 
chunk     0x602090         0x20         (inuse)
chunk     0x6020b0         0x90         (F) FD 0x602000 BK 0x7ffff7dd1bf8 
chunk     0x602140         0x20         (inuse)
chunk     0x602160         0xa0         (F) FD 0x7ffff7dd1c08 BK 0x7ffff7dd1c08 (LC)
chunk     0x602200         0x20         (inuse)
chunk     0x602220         0xb0         (inuse)
chunk     0x6022d0         0x20d30      (top)
sbrk_end  0x602001
# 这里需要理解 Unsorted bin 是如何变为 small bin
(gdb) heap -b
===================================Heap Dump===================================

small bin 9 @ 0x7ffff7dd1c08
        free chunk @ 0x6020b0 - size 0x90

        free chunk @ 0x602000 - size 0x90
small bin 10 @ 0x7ffff7dd1c18
        free chunk @ 0x602160 - size 0xa0
# bin_at 的返回, 需要联合上面的两条命令返回的结果一起理解
(gdb) p *(mfastbinptr)0x7ffff7dd1BF8 
$3 = {prev_size = 140737351850984, size = 140737351850984, fd = 0x6020b0, bk = 0x602000, fd_nextsize = 0x602160, bk_nextsize = 0x602160}
# bin_at 的返回, 需要联合上面的两条命令返回的结果一起理解
(gdb) p *(mfastbinptr)0x7ffff7dd1C08
$2 = {prev_size = 6299824, size = 6299648, fd = 0x602160, bk = 0x602160, fd_nextsize = 0x7ffff7dd1c18 <main_arena+248>, bk_nextsize = 0x7ffff7dd1c18 <main_arena+248>}
```
上面提到如何使用 Bins 泄露 libc 和 heap 的地址, 这一部分其实在 Phrack 的` <Advanced Doug Lea's malloc exploits>` 的` 4.5 Abusing the leaked information `一小部分有提到. 可以通过 "find a lonely chunk in the heap" 去泄露, 相当于上面例子中的 m3, 位于 small bin 10, 释放后会修改 FD, BK 为该 Bin 的地址, 进而泄露 libc 的地址. 还有一种方法就是 "find the first or last chunk of a bin", 相当于上面例子中的 m1, m2, 释放后, 会造成 FD 和 BK 一个在 `ptr_2_libc's_memory`, 一个在 `ptr_2_process'_heap`.

下面说明如何使用一个` lonely chunk`, 拿到关键函数的地址, 在 `<Advanced Doug Lea's malloc exploits> `中使用的是 `__morecore `这个函数指针, 它指向 `__default_morecore`, 也就是系统用于增加内存的函数, 默认为 `brk()`, 这里简单提一下.

这里直接使用上面的 m3 作为例子举例,` m3 `在释放后变为 `lonely chunk`, 位于 `small bin 10`
``` c
#0.这里已知该 chunk 所在 bin 的地址 (t0 = 0x7ffff7dd1c08+0x10)(对于为什么需要加 0x10, 是因为 fake chunk, 具体参考上面)
#1.根据 chunk 的 size, 取得对应 bin index, 这里其实也就是 10, 可以查看 bin_index 宏, 查看对应具体实现
#2.根据 bin index, 获取到该 bin 与 main_arena 的地址差, 从而获得 main_arena 的地址.
t0 = 0x7ffff7dd1c08 + 0x10
t1 = (long)&main_arena.bins - (long)&main_arena
t2 = (long)&__morecore - (long)&(main_arena)
t3 = (10-1)*2*8 //至于为什么这么算, 请参考源码 bin_at 宏
&main_arena = t0 - (t3+t1) = 0x7ffff7dd1b20
#3.根据 _morecore 与 main_arena 的地址差, 得到 _morecore 的地址
&__morecore = &main_arena + t2
```