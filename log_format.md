leveldb Log format LevelDB日志格式
==================
The log file contents are a sequence of 32KB blocks.  The only exception is that
the tail of the file may contain a partial block.

Each block consists of a sequence of records:

日志文件的内容是由一系列32KB的块组成。唯一的例外是最后一个文件可能只包含了部分块。
每个块包含了一系列的记录：

    block := record* trailer?
    record :=
      checksum: uint32     // crc32c of type and data[] ; little-endian
      length: uint16       // little-endian
      type: uint8          // One of FULL, FIRST, MIDDLE, LAST
      data: uint8[length]

A record never starts within the last six bytes of a block (since it won't fit).
Any leftover bytes here form the trailer, which must consist entirely of zero
bytes and must be skipped by readers.

一条记录绝不会在一个数据块的最后六个字节内开始（因为放不下）。这里剩余的任何字节构成尾部，尾部必须全部由零字节组成，并且读取程序必须跳过这些字节。

Aside: if exactly seven bytes are left in the current block, and a new non-zero
length record is added, the writer must emit a FIRST record (which contains zero
bytes of user data) to fill up the trailing seven bytes of the block and then
emit all of the user data in subsequent blocks.

题外话：如果当前数据块恰好还剩下七个字节，并且要添加一条新的非零长度记录，那么写入程序必须先发出一条FIRST记录（其中包含零字节的用户数据）来填满该数据块剩余的七个字节，然后再在后续的数据块中发出所有的用户数据。

More types may be added in the future.  Some Readers may skip record types they
do not understand, others may report that some data was skipped.

将来可能会添加更多的类型，有些读取程序可能会跳过他们无法理解的记录类型，而其他读取程序可能会报告某些数据已被跳过。

    FULL == 1
    FIRST == 2
    MIDDLE == 3
    LAST == 4

The FULL record contains the contents of an entire user record.

FULL记录包含整条用户记录的内容。

FIRST, MIDDLE, LAST are types used for user records that have been split into
multiple fragments (typically because of block boundaries).  FIRST is the type
of the first fragment of a user record, LAST is the type of the last fragment of
a user record, and MIDDLE is the type of all interior fragments of a user
record.

FIRST，MIDDLE，LAST是用于哪些已被分割成多个片段的用户记录的类型（通常是由于块边界的原因）。FIRST是用户记录第一个片段的类型，尾是用户记录最后一个片段的类型，而MIDDLE则是用户记录所有中间片段的类型。

Example: consider a sequence of user records:

示例：考虑一些列用户记录：

    A: length 1000
    B: length 97270
    C: length 8000

**A** will be stored as a FULL record in the first block.

**B** will be split into three fragments: first fragment occupies the rest of
the first block, second fragment occupies the entirety of the second block, and
the third fragment occupies a prefix of the third block.  This will leave six
bytes free in the third block, which will be left empty as the trailer.

**C** will be stored as a FULL record in the fourth block.

**A** 将作为一条完整FULL记录存储在第一个数据块中。

**B** 将被分割为三个片段：第一个片段占据第一个数据块的剩余部分，第二个片段占据整个第二个数据块，第三个片段占据第三个数据块的开头部分。这样在第三个数据块中会剩余六个字节，这六个字节将作为尾部留空。

**C** 将作为一条FULL记录存储在第四个数据块中。

----

## Some benefits over the recordio format: 相较于recordio格式的一些优势

1. We do not need any heuristics for resyncing - just go to next block boundary
   and scan.  If there is a corruption, skip to the next block.  As a
   side-benefit, we do not get confused when part of the contents of one log
   file are embedded as a record inside another log file.

2. Splitting at approximate boundaries (e.g., for mapreduce) is simple: find the
   next block boundary and skip records until we hit a FULL or FIRST record.

3. We do not need extra buffering for large records.


1. 我们不需要任何用于重新同步的启发式方法——只需转到下一个数据块边界并进行扫描即可。如果存在损坏情况，就跳到下一个数据块。附带的好处是，当一个日志文件的部分内容作为一条记录嵌入到另一个日志文件中时，我们不会产生混淆。

2. 在大致边界外进行分割（例如，对于MapReduce而言）很简单：找到下一个数据块边界，然后跳过记录，知道遇到FULL记录或FIRST记录。

3. 对于大型记录，我们不需要额外的缓冲。

## Some downsides compared to recordio format: 相较于recordio格式的一些劣势

1. No packing of tiny records.  This could be fixed by adding a new record type,
   so it is a shortcoming of the current implementation, not necessarily the
   format.

2. No compression.  Again, this could be fixed by adding new record types.


1. 无法对微小记录进行打包。这可以通过添加一种新的记录类型来解决，所以这是当前实现方式的一个不足之处，而不一定是这种格式本身的问题。

2. 没有压缩功能。同样，这也可以通过添加新的记录类型来解决。