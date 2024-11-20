## Files 文件

The implementation of leveldb is similar in spirit to the representation of a
single [Bigtable tablet (section 5.3)](https://research.google/pubs/pub27898/).
However the organization of the files that make up the representation is
somewhat different and is explained below.

LevelDB的实现原理在本质上与单个[Bigtable tablet (section 5.3)](https://research.google/pubs/pub27898/)的表示方式相似。然而，构成这种表示方式的文件组织结构略有不同，下面将对此进行解释。

Each database is represented by a set of files stored in a directory. There are
several different types of files as documented below:

每个数据库都由存储在一个目录中的一组文件来表示。如下文所述，存在集中不同类型的文件：

### Log files 日志文件

A log file (*.log) stores a sequence of recent updates. Each update is appended
to the current log file. When the log file reaches a pre-determined size
(approximately 4MB by default), it is converted to a sorted table (see below)
and a new log file is created for future updates.

日志文件(*.log)存储了一系列近期的更新操作。每次更新都会被追加到当前的日志文件中。当日志文件达到预先设定的大小（默认情况下约为4MB）时，他会被转换为一个已排序的表（见下文），并且会创建一个新的日志文件用于后续的更新操作。

A copy of the current log file is kept in an in-memory structure (the
`memtable`). This copy is consulted on every read so that read operations
reflect all logged updates.

当前日志文件的一个副本会保存在内存结构（即memtable）中，每次读取数据时都会查询这个副本，以便读取操作能反映出所有已记录的更新内容。

## Sorted tables 有序表

A sorted table (*.ldb) stores a sequence of entries sorted by key. Each entry is
either a value for the key, or a deletion marker for the key. (Deletion markers
are kept around to hide obsolete values present in older sorted tables).

一个已排序的表（*.ldb)存储了一系列按照键进行排序的条目。每个条目要么是该键对应的值，要么是该键的删除标记。（保留删除标记是为了隐藏旧的已排序表中存在的过时值。）


The set of sorted tables are organized into a sequence of levels. The sorted
table generated from a log file is placed in a special **young** level (also
called level-0). When the number of young files exceeds a certain threshold
(currently four), all of the young files are merged together with all of the
overlapping level-1 files to produce a sequence of new level-1 files (we create
a new level-1 file for every 2MB of data.)

这组已排序的表被组织成一系列的层级。由日志文件生成的已排序表会被放置在一个特殊的**年轻**层级（也称为第0层）中。当年轻文件的数量超过某个阈值（目前是四个）时，所有年轻文件会与所有有重叠部分的第1层文件合并在一起，从而生成一系列新的第1层文件（我们没2MB的数据就创建一个新的第1层文件）。

Files in the young level may contain overlapping keys. However files in other
levels have distinct non-overlapping key ranges. Consider level number L where
L >= 1. When the combined size of files in level-L exceeds (10^L) MB (i.e., 10MB
for level-1, 100MB for level-2, ...), one file in level-L, and all of the
overlapping files in level-(L+1) are merged to form a set of new files for
level-(L+1). These merges have the effect of gradually migrating new updates
from the young level to the largest level using only bulk reads and writes
(i.e., minimizing expensive seeks).

年轻层级中的文件可能包含由重叠的键。然而其他层级中的文件具有互不重叠的明确键范围。考虑层级编号为L且L >= 1的情况。当层级L中的文件的大小超过(10^L)MB字节（即，层级1为10MB字节，层级2为100MB字节，以此类推）时，层级L中的一个文件以及层级（L+1)中所有与之有重叠部分的文件会被合并，以形成一组层级（L+1）的新文件。这些合并操作的效果是，仅通过批量读写（即，最大限度地减少高成本的寻道操作），将新的更新内容从年轻层级逐渐迁移到最高层级。

### Manifest 清单

A MANIFEST file lists the set of sorted tables that make up each level, the
corresponding key ranges, and other important metadata. A new MANIFEST file
(with a new number embedded in the file name) is created whenever the database
is reopened. The MANIFEST file is formatted as a log, and changes made to the
serving state (as files are added or removed) are appended to this log.

一个清单（MANIFEST）文件列出了构成每个层级的已排序表的集合，相应的键范围以及其他重要元数据。每当数据库重新打开时，就会创建一个新的清单文件（文件名中嵌入了一个新的编号）。清单文件的格式是日志形式，对服务状态所做的更改（比如添加或删除文件时）都会被追加到这个日志中。

### Current 当前文件

CURRENT is a simple text file that contains the name of the latest MANIFEST
file.

CURRENT是一个简单的文本文件，其中包含最新的MANIFEST文件的名称。

### Info logs 信息日志

Informational messages are printed to files named LOG and LOG.old.

信息性的消息会被打印到名为LOG和LOG.old的文件中。

### Others 其他

Other files used for miscellaneous purposes may also be present (LOCK, *.dbtmp).

可能还会存在用于其他各种用途的文件（如LOCK、*.dbtmp)。

## Level 0 第0层

When the log file grows above a certain size (4MB by default):
Create a brand new memtable and log file and direct future updates here.

In the background:

1. Write the contents of the previous memtable to an sstable.
2. Discard the memtable.
3. Delete the old log file and the old memtable.
4. Add the new sstable to the young (level-0) level.

当日志文件增长到超过一定大小（默认是4MB）时：
创建一个全新的内存表和日志文件，并将后续的更新操作导向这里。

在后台进行以下操作：
1. 将之前内存表的内容写入一个SSTable（静态排序表）。
2. 丢弃该内存表。
3. 删除旧的日志文件和旧的内存表。
4. 将新的SSTable添加到年轻（第0层）层级中。

## Compactions 合并操作

When the size of level L exceeds its limit, we compact it in a background
thread. The compaction picks a file from level L and all overlapping files from
the next level L+1. Note that if a level-L file overlaps only part of a
level-(L+1) file, the entire file at level-(L+1) is used as an input to the
compaction and will be discarded after the compaction.  Aside: because level-0
is special (files in it may overlap each other), we treat compactions from
level-0 to level-1 specially: a level-0 compaction may pick more than one
level-0 file in case some of these files overlap each other.

当层级L的大小超过其限制时，我们会在一个后台线程中对其进行合并操作。合并操作会从层级L中选取一个文件以及下一层级L+1中所有与之有重叠部分的文件。需要注意的是，如果层级L的一个文件仅与该层级L+1的一个文件的部分区域有重叠，那么层级L+1的整个文件都会被用作合并操作的输入。并且在合并操作完成后会被丢弃。另外：由于第0层比较特殊（该层内的文件可能相互重叠），我们对从第0层到第1层的合并操作会进行特殊处理：在第0层的合并操作中，如果某些文件相互重叠，可能会选取不止一个地0层的文件。

A compaction merges the contents of the picked files to produce a sequence of
level-(L+1) files. We switch to producing a new level-(L+1) file after the
current output file has reached the target file size (2MB). We also switch to a
new output file when the key range of the current output file has grown enough
to overlap more than ten level-(L+2) files.  This last rule ensures that a later
compaction of a level-(L+1) file will not pick up too much data from
level-(L+2).

一次合并操作会将所选文件的内容进行合并，以生存一系列层级为（L+1）的文件。当当前输出文件达到目标文件大小（2MB）后，我们就会切换去生成一个新的层级为（L+1）的文件。当当前输出文件的键范围增长到足以与超过十个层级为（L+2）的文件产生重叠时，我们也会切换到一个新的输出文件。这最后一条规则确保了之后对层级（L+1）的文件进行合并操作时，不会从层级为（L+2）的文件中获取过多的数据。


The old files are discarded and the new files are added to the serving state.

旧文件会被丢弃，新闻界则会被添加到服务状态中。

Compactions for a particular level rotate through the key space. In more detail,
for each level L, we remember the ending key of the last compaction at level L.
The next compaction for level L will pick the first file that starts after this
key (wrapping around to the beginning of the key space if there is no such
file).

针对特定层级的合并操作会在键空间中循环进行。更详细地说，对于每个层级L，我们会记住在层级L上一次合并操作的结束键。层级L的下一次合并操作将会选取在此键之后开始的第一个文件（如果没有这样的文件，就会循环回到键空间的起始位置）。

Compactions drop overwritten values. They also drop deletion markers if there
are no higher numbered levels that contain a file whose range overlaps the
current key.

合并操作会丢弃被覆盖的值。如果不存在更高层级的文件其范围与当前键由重叠，那么它们也会丢弃删除标记。

### Timing 时间安排

Level-0 compactions will read up to four 1MB files from level-0, and at worst
all the level-1 files (10MB). I.e., we will read 14MB and write 14MB.

第0层的合并操作最多会从第0层读取四个1MB的文件，最坏的情况下还会读取所有第1层的文件（10MB）。也就是说，我们将会读取14MB的数据并写入14MB的数据。

Other than the special level-0 compactions, we will pick one 2MB file from level
L. In the worst case, this will overlap ~ 12 files from level L+1 (10 because
level-(L+1) is ten times the size of level-L, and another two at the boundaries
since the file ranges at level-L will usually not be aligned with the file
ranges at level-L+1). The compaction will therefore read 26MB and write 26MB.
Assuming a disk IO rate of 100MB/s (ballpark range for modern drives), the worst
compaction cost will be approximately 0.5 second.

除了特殊的第0层合并操作之外，我们会从层级L中选取一个2MB的文件。在最坏的情况下，它将会与层级L+1的大约12个文件产生重叠（10个是因为层级（L+1）的大小是层级L的十倍，另外两个是在边界处，因为层级L的文件范围通常与层级L+1的文件范围不一致）。因此，这次合并操作将会读取26MB的数据并写入26MB的数据。假设磁盘IO速率为100MB/s（现代硬盘大致的速率范围），最坏情况下的合并成本大约为0.5秒。

If we throttle the background writing to something small, say 10% of the full
100MB/s speed, a compaction may take up to 5 seconds. If the user is writing at
10MB/s, we might build up lots of level-0 files (~50 to hold the 5*10MB). This
may significantly increase the cost of reads due to the overhead of merging more
files together on every read.

如果我们将后台写入速度限制在一个较小的值，比如全速100MB/s的10%，那么一次合并操作可能最多需要5秒。如果用户的写入速度为10MB/s，我们可能会累计大量的第0层文件（大约50个来容纳5*10MB的数据）。由于每次读取时合并更多文件的开销，这可能会显著增加读取的成本。


Solution 1: To reduce this problem, we might want to increase the log switching
threshold when the number of level-0 files is large. Though the downside is that
the larger this threshold, the more memory we will need to hold the
corresponding memtable.

解决方案1：为了减少这个文件，当第0层文件的数量较多时，我们可能想要提高日志切换阈值。不过不利的一面是，这个阈值越大，我们为了保存相应的内存表就需要越多的内存。

Solution 2: We might want to decrease write rate artificially when the number of
level-0 files goes up.

解决方案2：当第0层文件的数量增加时，我们可能想要人为地降低写入速率。

Solution 3: We work on reducing the cost of very wide merges. Perhaps most of
the level-0 files will have their blocks sitting uncompressed in the cache and
we will only need to worry about the O(N) complexity in the merging iterator.

解决方案3：我们致力于降低超宽合并的成本。也许大多数第0层文件的块会以未压缩的状态放在缓存中，这样我们只需要担心合并迭代器中的O(N)复杂度问题。

### Number of files 文件数量

Instead of always making 2MB files, we could make larger files for larger levels
to reduce the total file count, though at the expense of more bursty
compactions.  Alternatively, we could shard the set of files into multiple
directories.

我们不必总是创建2MB大小的文件，对于层级较高的情况，可以创建更大的文件以减少文件总数，不过这会以更具突发性的合并操作为代价。或者，我们也可以将文件集分割到多个目录中。

An experiment on an ext3 filesystem on Feb 04, 2011 shows the following timings
to do 100K file opens in directories with varying number of files:

2011年2月4日在ext3文件系统上进行的一项实验展示了在包含不同数量文件的目录中进行10万次文件打开操作的耗时情况如下：


| Files in directory 目录中的文件数量 | Microseconds to open a file 打开一个文件所需的微秒数 |
|-------------------:|----------------------------:|
|               1000 |                           9 |
|              10000 |                          10 |
|             100000 |                          16 |

So maybe even the sharding is not necessary on modern filesystems?

所以，也许在现代文件系统中，甚至连分割到多个目录这种做法都没必要了？

## Recovery 恢复

* Read CURRENT to find name of the latest committed MANIFEST
* Read the named MANIFEST file
* Clean up stale files
* We could open all sstables here, but it is probably better to be lazy...
* Convert log chunk to a new level-0 sstable
* Start directing new writes to a new log file with recovered sequence#

* 读取CURRENT文件以找到最新提交的MANIFEST文件的名称。
* 读取所找到的MANIFEST文件
* 清理过期文件
* 我们本可以再这里打开所有的SSTable，但或许惰性加载的方式会更好...
* 将日志块转换为一个新的第0层SSTable。
* 开始将新的写入操作导向一个带有恢复后的序列号的新日志文件。

## Garbage collection of files 文件的垃圾回收

`RemoveObsoleteFiles()` is called at the end of every compaction and at the end
of recovery. It finds the names of all files in the database. It deletes all log
files that are not the current log file. It deletes all table files that are not
referenced from some level and are not the output of an active compaction.

在每次合并操作结束以及恢复操作结束时，都会调用`RemoveObsoleteFiles()`函数。它会找出数据库中所有文件的名称。它会删除所有非当前日志文件的日志文件。它会删除所有未在某个层级中被引用且不是正在进行合并操作输出的表文件。
