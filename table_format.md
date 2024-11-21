leveldb File format LevelDB文件格式
===================

    <beginning_of_file>
    [data block 1]
    [data block 2]
    ...
    [data block N]
    [meta block 1]
    ...
    [meta block K]
    [metaindex block]
    [index block]
    [Footer]        (fixed size; starts at file_size - sizeof(Footer))
    <end_of_file>

The file contains internal pointers.  Each such pointer is called
a BlockHandle and contains the following information:

文件包含内部指针。每个这样的指针都被称作“块句柄（BlockHandle）”，它包含以下信息：

    offset:   varint64
    size:     varint64

See [varints](https://developers.google.com/protocol-buffers/docs/encoding#varints)
for an explanation of varint64 format.

varint64格式的解释可参见： [varints](https://developers.google.com/protocol-buffers/docs/encoding#varints)

1.  The sequence of key/value pairs in the file are stored in sorted
order and partitioned into a sequence of data blocks.  These blocks
come one after another at the beginning of the file.  Each data block
is formatted according to the code in `block_builder.cc`, and then
optionally compressed.

2. After the data blocks we store a bunch of meta blocks.  The
supported meta block types are described below.  More meta block types
may be added in the future.  Each meta block is again formatted using
`block_builder.cc` and then optionally compressed.

3. A "metaindex" block.  It contains one entry for every other meta
block where the key is the name of the meta block and the value is a
BlockHandle pointing to that meta block.

4. An "index" block.  This block contains one entry per data block,
where the key is a string >= last key in that data block and before
the first key in the successive data block.  The value is the
BlockHandle for the data block.

5. At the very end of the file is a fixed length footer that contains
the BlockHandle of the metaindex and index blocks as well as a magic number.


1. 文件中的键值对序列按照排序顺序存储，并被划分成一系列是数据块。这些数据块在文件开头一次排列。每个数据块依据`block_builder.cc`中的代码进行格式化，然后可选择进行压缩。

2. 在数据块之后，我们存储了一系列元数据块。下面将描述所支持的元数据块类型。将来可能会添加更多的元数据块类型。每个元数据块同样适用`block_builder.cc`进行格式化，然后可选择进行压缩。

3. 一个“元索引”块。它为其他每个元数据块都包含了一个条目，其中键是元数据块的名称，值是指向该元数据块的块句柄。

4. 一个“索引”块。该块为每个数据块都包含了一个条目，其中键是一个字符串，其值大于等于该数据块中的最后一个键，且小于下一个连续数据块中的第一个键。值是该数据块的块句柄。

5. 在文件的末尾是一个固定长度的页脚，它包含元索引和索引块的块句柄以及一个幻数。

        metaindex_handle: char[p];     // Block handle for metaindex
        index_handle:     char[q];     // Block handle for index
        padding:          char[40-p-q];// zeroed bytes to make fixed length
                                       // (40==2*BlockHandle::kMaxEncodedLength)
        magic:            fixed64;     // == 0xdb4775248b80fb57 (little-endian)


## "filter" Meta Block “过滤器”元数据块

If a `FilterPolicy` was specified when the database was opened, a
filter block is stored in each table.  The "metaindex" block contains
an entry that maps from `filter.<N>` to the BlockHandle for the filter
block where `<N>` is the string returned by the filter policy's
`Name()` method.

如果在打开数据库时指定了“过滤策略”，那么每个表中都会存储一个过滤器块。“元索引”块包含一个条目，它将`filter.<N>`映射到过滤器块的块句柄，其中`<N>`是过滤策略的`Name()`方法返回的字符串。

The filter block stores a sequence of filters, where filter i contains
the output of `FilterPolicy::CreateFilter()` on all keys that are stored
in a block whose file offset falls within the range

过滤器块存储了一系列过滤器，其中第i个过滤器包含了对所有存储在文件偏移量处于以下范围的块中的键执行`FilterPolicy::CreateFilter()`的输出：

    [ i*base ... (i+1)*base-1 ]

Currently, "base" is 2KB.  So for example, if blocks X and Y start in
the range `[ 0KB .. 2KB-1 ]`, all of the keys in X and Y will be
converted to a filter by calling `FilterPolicy::CreateFilter()`, and the
resulting filter will be stored as the first filter in the filter
block.

目前“基值”是2KB。例如，如果块X和块Y起始于`[ 0KB .. 2KB-1 ]`这个范围，那么通过调用`FilterPolicy::CreateFilter()`，X和Y中所有键都将被转换为一个过滤器，并且生成的过滤器将作为过滤器块中的第一个过滤器进行存储。

The filter block is formatted as follows:

过滤器块的格式如下：

    [filter 0]
    [filter 1]
    [filter 2]
    ...
    [filter N-1]

    [offset of filter 0]                  : 4 bytes
    [offset of filter 1]                  : 4 bytes
    [offset of filter 2]                  : 4 bytes
    ...
    [offset of filter N-1]                : 4 bytes

    [offset of beginning of offset array] : 4 bytes
    lg(base)                              : 1 byte

The offset array at the end of the filter block allows efficient
mapping from a data block offset to the corresponding filter.

过滤器块末尾的偏移量数组能够实现从数据块偏移量到相应过滤器的高效映射。

## "stats" Meta Block “统计信息”元数据块

This meta block contains a bunch of stats.  The key is the name
of the statistic.  The value contains the statistic.

这个元数据块包含了一系列统计数据。键是统计量的名称，值则包含了具体的统计量内容。

TODO(postrelease): record following stats.

TODO(发布后)：记录以下统计数据。

    data size
    index size
    key size (uncompressed)
    value size (uncompressed)
    number of entries
    number of data blocks
