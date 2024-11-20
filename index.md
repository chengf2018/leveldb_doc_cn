leveldb
=======

_Jeff Dean, Sanjay Ghemawat_

The leveldb library provides a persistent key value store. Keys and values are
arbitrary byte arrays.  The keys are ordered within the key value store
according to a user-specified comparator function.

leveldb库提供了一个持久化的键值存储。建和值都是任意的字节数组。在键值存储中，键是根据用户指定的比较器函数进行排序的。

## Opening A Database 打开一个数据库

A leveldb database has a name which corresponds to a file system directory. All
of the contents of database are stored in this directory. The following example
shows how to open a database, creating it if necessary:

一个leveldb数据库有一个名称，该名称对应于一个文件系统目录。数据库所有的内容都存储在这个目录中。以下示例展示了如何打开一个数据库，如果有必要的话还会创建它：

```c++
#include <cassert>
#include "leveldb/db.h"

leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
assert(status.ok());
...
```

If you want to raise an error if the database already exists, add the following
line before the `leveldb::DB::Open` call:

如果你想在数据库已经存在时抛出错误，在leveldb::DB::Open调用前添加以下行：

```c++
options.error_if_exists = true;
```

## Status 状态

You may have noticed the `leveldb::Status` type above. Values of this type are
returned by most functions in leveldb that may encounter an error. You can check
if such a result is ok, and also print an associated error message:

你可能注意到了上面的leveldb::Status类型。这个类型的值是大多数可能遇到错误的leveldb函数返回。你可以像这样检查一个结果是否正常，并且还能打印出相关的错误信息：

```c++
leveldb::Status s = ...;
if (!s.ok()) cerr << s.ToString() << endl;
```

## Closing A Database 关闭一个数据库

When you are done with a database, just delete the database object. Example:

完成数据库操作后，只需删除数据库对象。示例：

```c++
... open the db as described above ...
... do something with db ...
delete db;
```

## Reads And Writes 读和写

The database provides Put, Delete, and Get methods to modify/query the database.
For example, the following code moves the value stored under key1 to key2.

数据库提供了Put，Delete和Get方法去修改/查询数据库。例如，下面的代码将储存在key1下面的值移动到了key2.

```c++
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) s = db->Put(leveldb::WriteOptions(), key2, value);
if (s.ok()) s = db->Delete(leveldb::WriteOptions(), key1);
```

## Atomic Updates 原子更新

Note that if the process dies after the Put of key2 but before the delete of
key1, the same value may be left stored under multiple keys. Such problems can
be avoided by using the `WriteBatch` class to atomically apply a set of updates:

注意，如果进程在对key2执行Put操作后，单在对key1执行Delete操作之前崩溃了，那么相同的值可能会被存储在多个建下。通过使用WriteBatch类以原子方式应用一组更新操作，就可以避免此类问题：

```c++
#include "leveldb/write_batch.h"
...
std::string value;
leveldb::Status s = db->Get(leveldb::ReadOptions(), key1, &value);
if (s.ok()) {
  leveldb::WriteBatch batch;
  batch.Delete(key1);
  batch.Put(key2, value);
  s = db->Write(leveldb::WriteOptions(), &batch);
}
```

The `WriteBatch` holds a sequence of edits to be made to the database, and these
edits within the batch are applied in order. Note that we called Delete before
Put so that if key1 is identical to key2, we do not end up erroneously dropping
the value entirely.

WriteBatch包含了一系列要对数据库进行的编辑操作，并且该批次内的这些编辑操作会按顺序执行。注意，我们再执行Put操作之前先调用了Delete操作，这样的话，如果key1与key2相同，我们就不会错误地将该值完全删掉。

Apart from its atomicity benefits, `WriteBatch` may also be used to speed up
bulk updates by placing lots of individual mutations into the same batch.

除了具有原子性方面的优势外，WriteBatch还可通过将大量单独的变更操作放入同一个批次来加快批量更新的速度。

## Synchronous Writes 同步写

By default, each write to leveldb is asynchronous: it returns after pushing the
write from the process into the operating system.The transfer from operating system 
memory to the underlying persistent storage happens asynchronously. 

默认情况下，所有对leveldb的写是异步的：leveldb在推送一个进程对操作系统的写后返回。从操作系统内存到底层持久性存储的传输是异步的。

The sync flag can be turned on for a particular write to make the write operation
not return until the data being written has been pushed all the way to
persistent storage. (On Posix systems, this is implemented by calling either
`fsync(...)` or `fdatasync(...)` or `msync(..., MS_SYNC)` before the write
operation returns.)

对于特定的写入操作，可以开启同步标志，这样在被写入的数据被推送到持久性存储之前，写入操作不会返回。
（在Posix系统上，这是通过在写操作返回之前调用`fsync(...)` or `fdatasync(...)` or `msync(..., MS_SYNC)`实现的。）

```c++
leveldb::WriteOptions write_options;
write_options.sync = true;
db->Put(write_options, ...);
```

Asynchronous writes are often more than a thousand times as fast as synchronous
writes. The downside of asynchronous writes is that a crash of the machine may
cause the last few updates to be lost. Note that a crash of just the writing
process (i.e., not a reboot) will not cause any loss since even when sync is
false, an update is pushed from the process memory into the operating system
before it is considered done.

异步写通常比同步写快一千倍。异步写的缺点是机器崩溃可能会导致最后几条更新丢失。注意，仅仅是写进程崩溃，并不会导致任何数据丢失，因为即使是在同步标记设置为假的情况下，在一次更新操作被视为完成之前，其相关数据也会从进程内存推送到操作系统中。

Asynchronous writes can often be used safely. For example, when loading a large
amount of data into the database you can handle lost updates by restarting the
bulk load after a crash. A hybrid scheme is also possible where every Nth write
is synchronous, and in the event of a crash, the bulk load is restarted just
after the last synchronous write finished by the previous run. (The synchronous
write can update a marker that describes where to restart on a crash.)

异步写通常可以安全使用。例如，在将大量数据写入到数据库时，你可以通过在系统崩溃后重新启动批量加载操作来处理丢失的内容。一种混合方案也是可行的，即每第N次写入采用同步方式，并且在发生崩溃的情况下，就在上一次运行完成的最后一次同步写入之后重新启动批量加载操作。（同步写入可以更新一个标记，该标记描述了在崩溃时应从何处重新启动。）

`WriteBatch` provides an alternative to asynchronous writes. Multiple updates
may be placed in the same WriteBatch and applied together using a synchronous
write (i.e., `write_options.sync` is set to true). The extra cost of the
synchronous write will be amortized across all of the writes in the batch.

‘WriteBatch’为异步写入提供了一种替代方案。可以将多个更新操作放在同一个WirteBatch中，然后通过同步写入（即把write_options.sync设置为true）一起执行这些更新操作。同步写入的额外消耗将分摊到改批次的所有写入操作上。

## Concurrency  并发

A database may only be opened by one process at a time. The leveldb
implementation acquires a lock from the operating system to prevent misuse.
Within a single process, the same `leveldb::DB` object may be safely shared by
multiple concurrent threads. I.e., different threads may write into or fetch
iterators or call Get on the same database without any external synchronization
(the leveldb implementation will automatically do the required synchronization).
However other objects (like Iterator and `WriteBatch`) may require external
synchronization. If two threads share such an object, they must protect access
to it using their own locking protocol. More details are available in the public
header files.

一个数据库同一时间只能被一个进程打开。实现从操作系统获取一个锁以防止误用。在单个进程内，同一个leveldb：DB可以被多个并发线程安全共享。也就是说，不同的线程可以在无需任何外部同步机制的情况下，对同一个数据库进行写入操作、获取迭代器或调用Get操作（LevelDB的实现会自动完成所需的同步操作）。然而，其他对象（如Iterator和`WriteBatch`)可能需要外部同步。如果两个线程共享这样一个对象，它们必须使用自己的加锁协议来保护对象的访问。更多详细信息可在公开的头文件中获取。

## Iteration 迭代器

The following example demonstrates how to print all key,value pairs in a
database.

下面的例子展示了如何打印一个数据库中所有的key/value对。

```c++
leveldb::Iterator* it = db->NewIterator(leveldb::ReadOptions());
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  cout << it->key().ToString() << ": "  << it->value().ToString() << endl;
}
assert(it->status().ok());  // Check for any errors found during the scan
delete it;
```

The following variation shows how to process just the keys in the range
[start,limit):

下面的变化展示了如何处理只在[start,limit)范围的键。

```c++
for (it->Seek(start);
   it->Valid() && it->key().ToString() < limit;
   it->Next()) {
  ...
}
```

You can also process entries in reverse order. (Caveat: reverse iteration may be
somewhat slower than forward iteration.)

你还可以按相反的顺序处理条目。（警告：反向迭代器可能慢于正向迭代器。）

```c++
for (it->SeekToLast(); it->Valid(); it->Prev()) {
  ...
}
```

## Snapshots 快照

Snapshots provide consistent read-only views over the entire state of the
key-value store.  `ReadOptions::snapshot` may be non-NULL to indicate that a
read should operate on a particular version of the DB state. If
`ReadOptions::snapshot` is NULL, the read will operate on an implicit snapshot
of the current state.

快照可为键值存储的整个状态提供一致的只读视图。ReadOptions::snapshot可能不为空，这表示一次读取操作应当基于数据库状态的特定版本来进行。如果ReadOptions::snapshot为空，那么读取操作将基于当前状态的隐式快照来进行。

Snapshots are created by the `DB::GetSnapshot()` method:

快照通过DB::GetSnapshot()方法创建：

```c++
leveldb::ReadOptions options;
options.snapshot = db->GetSnapshot();
... apply some updates to db ...
leveldb::Iterator* iter = db->NewIterator(options);
... read using iter to view the state when the snapshot was created ...
delete iter;
db->ReleaseSnapshot(options.snapshot);
```

Note that when a snapshot is no longer needed, it should be released using the
`DB::ReleaseSnapshot` interface. This allows the implementation to get rid of
state that was being maintained just to support reading as of that snapshot.

注意，当不在需要某个快照时，应当使用DB::ReleaseSnapshot接口来释放它。这样一来，实现就可以清理掉那些仅仅为了支持基于该快照进行读取操作而维护的状态信息。

## Slice 切片

The return value of the `it->key()` and `it->value()` calls above are instances
of the `leveldb::Slice` type. Slice is a simple structure that contains a length
and a pointer to an external byte array. Returning a Slice is a cheaper
alternative to returning a `std::string` since we do not need to copy
potentially large keys and values. In addition, leveldb methods do not return
null-terminated C-style strings since leveldb keys and values are allowed to
contain `'\0'` bytes.

上述it->key()和it->value()调用的返回值是leveldb::Slice类型的实例。Slice是一个简单的结构体，他包含一个长度和一个指向外部字节数组的指针。返回Slice相比于返回std::string是一种成本更低的选择，因为我们不需要复制可能很大的键和值。此外，leveldb的方法不会返回以空字符结尾的C风格字符串，因为leveldb的键和值是允许包含'\0'字节的。

C++ strings and null-terminated C-style strings can be easily converted to a
Slice:

c++字符串和0结尾的c风格字符串可以很容易地转化为一个Slice：

```c++
leveldb::Slice s1 = "hello";

std::string str("world");
leveldb::Slice s2 = str;
```

A Slice can be easily converted back to a C++ string:

一个Slice可以很容易转换回一个C++字符串：

```c++
std::string str = s1.ToString();
assert(str == std::string("hello"));
```

Be careful when using Slices since it is up to the caller to ensure that the
external byte array into which the Slice points remains live while the Slice is
in use. For example, the following is buggy:

使用Slice是要格外小心，因为在Slice被使用期间，确保其所指向的外部字节数组一直有效是调用者的责任。例如，以下代码是 有问题的：

```c++
leveldb::Slice slice;
if (...) {
  std::string str = ...;
  slice = str;
}
Use(slice);
```

When the if statement goes out of scope, str will be destroyed and the backing
storage for slice will disappear.

当if语句的作用域结束时，str将会被销毁，而Slice所依赖的存储也将不复存在。

## Comparators 比较器

The preceding examples used the default ordering function for key, which orders
bytes lexicographically. You can however supply a custom comparator when opening
a database.  For example, suppose each database key consists of two numbers and
we should sort by the first number, breaking ties by the second number. First,
define a proper subclass of `leveldb::Comparator` that expresses these rules:

前面的示例对键是用了默认的排序函数，该函数按照字节的字典序进行排序。然而，在打开数据库时你可以提供一个自定义的比较器。例如，假设每个数据库键由两个数字组成，并且我们应该按照第一个数字来进行排序，在第一个数字相同的情况下再按照第二个数字来打破平局。首先，定义一个leveldb:Comparator的合适子类来表述这些规则：

```c++
class TwoPartComparator : public leveldb::Comparator {
 public:
  // Three-way comparison function:
  //   if a < b: negative result
  //   if a > b: positive result
  //   else: zero result
  int Compare(const leveldb::Slice& a, const leveldb::Slice& b) const {
    int a1, a2, b1, b2;
    ParseKey(a, &a1, &a2);
    ParseKey(b, &b1, &b2);
    if (a1 < b1) return -1;
    if (a1 > b1) return +1;
    if (a2 < b2) return -1;
    if (a2 > b2) return +1;
    return 0;
  }

  // Ignore the following methods for now:
  const char* Name() const { return "TwoPartComparator"; }
  void FindShortestSeparator(std::string*, const leveldb::Slice&) const {}
  void FindShortSuccessor(std::string*) const {}
};
```

Now create a database using this custom comparator:

现在创建一个数据库用于这个自定义的比较器：

```c++
TwoPartComparator cmp;
leveldb::DB* db;
leveldb::Options options;
options.create_if_missing = true;
options.comparator = &cmp;
leveldb::Status status = leveldb::DB::Open(options, "/tmp/testdb", &db);
...
```

### Backwards compatibility 向后兼容性

The result of the comparator's Name method is attached to the database when it
is created, and is checked on every subsequent database open. If the name
changes, the `leveldb::DB::Open` call will fail. Therefore, change the name if
and only if the new key format and comparison function are incompatible with
existing databases, and it is ok to discard the contents of all existing
databases.

比较器的Name方法的结果在数据库创建时会与数据库相关联，并且在后续每次打开数据库时都会进行检查。如果名称发生了变化，那么leveldb::DB::Open调用将会失败。因此，当且仅当新的键格式和比较函数与现有的数据库不兼容，并且可以丢弃所有先用数据库的内容时，才可以更改名称。

You can however still gradually evolve your key format over time with a little
bit of pre-planning. For example, you could store a version number at the end of
each key (one byte should suffice for most uses). When you wish to switch to a
new key format (e.g., adding an optional third part to the keys processed by
`TwoPartComparator`), (a) keep the same comparator name (b) increment the
version number for new keys (c) change the comparator function so it uses the
version numbers found in the keys to decide how to interpret them.

不过，只要你稍加预先规划，你仍然可以随着时间逐步演进你的键格式。例如，你可以在每个键的末尾存储一个版本号（对大多数用途来说，一个字节就够了）。当你希望切换到一种新的键格式时（比如，给由TwoPartComparator处理的键添加一个可选的第三部分），需做到以下几点：
(a) 保持比较器名称不变；
(b) 为新键增加版本号；
(c) 更改比较器函数，使其能利用在键中找到的版本号来决定如何对这些键进行解读。

## Performance 性能

Performance can be tuned by changing the default values of the types defined in
`include/options.h`.

可以通过更改在`include/options.h`中定义的类型的默认值来调整性能。

### Block size 块大小

leveldb groups adjacent keys together into the same block and such a block is
the unit of transfer to and from persistent storage. The default block size is
approximately 4096 uncompressed bytes.  Applications that mostly do bulk scans
over the contents of the database may wish to increase this size. Applications
that do a lot of point reads of small values may wish to switch to a smaller
block size if performance measurements indicate an improvement. There isn't much
benefit in using blocks smaller than one kilobyte, or larger than a few
megabytes. Also note that compression will be more effective with larger block
sizes.

LevelDB会将相邻的键分组到同一个数据块中，这样一个数据块就是与持久化存储进行数据传输的单位。
默认的数据块大小约为4096个未压缩字节。对于大多数情况下会对数据库内容进行批量扫描的应用程序来说，
可能希望增大这个数据块的大小。而对于那些经常对小值进行点查询的应用程序来说，如果性能测试表明有所改善，可能
希望切换到更小的数据块大小。使用小于1千字节或大于几兆的数据块并没有太多益处。还要注意的是，数据块越大，压缩效果会越好。

### Compression 压缩

Each block is individually compressed before being written to persistent
storage. Compression is on by default since the default compression method is
very fast, and is automatically disabled for uncompressible data. In rare cases,
applications may want to disable compression entirely, but should only do so if
benchmarks show a performance improvement:

每个数据块在被写入持久化存储之前都会单独进行压缩。由于默认的压缩方法速度很快，所以默认是开启压缩的，并且对于不可压缩的数据会自动禁用压缩功能。在极少数情况下，应用程序可能想要完全禁用压缩，但只有在性能基准测试表明这样做能提高性能时才应该这么做：

```c++
leveldb::Options options;
options.compression = leveldb::kNoCompression;
... leveldb::DB::Open(options, name, ...) ....
```

### Cache 缓存

The contents of the database are stored in a set of files in the filesystem and
each file stores a sequence of compressed blocks. If options.block_cache is
non-NULL, it is used to cache frequently used uncompressed block contents.

数据库的内容存储在文件系统中的一组文件里，每个文件存储一系列经过压缩的块。如果options.bloc_cache不为空,它就会被用于缓存经常使用的未压缩数据块的内容。

```c++
#include "leveldb/cache.h"

leveldb::Options options;
options.block_cache = leveldb::NewLRUCache(100 * 1048576);  // 100MB cache
leveldb::DB* db;
leveldb::DB::Open(options, name, &db);
... use the db ...
delete db
delete options.block_cache;
```

Note that the cache holds uncompressed data, and therefore it should be sized
according to application level data sizes, without any reduction from
compression. (Caching of compressed blocks is left to the operating system
buffer cache, or any custom Env implementation provided by the client.)

注意，该缓存保存的是未压缩的数据，因此大小应根据引用程序级别的数据大小来设置，且不会因压缩而有任何缩减。（压缩数据块的缓存工作则交由操作系统的缓冲区缓存或客户端提供的任何自定义环境实现来完成。）

When performing a bulk read, the application may wish to disable caching so that
the data processed by the bulk read does not end up displacing most of the
cached contents. A per-iterator option can be used to achieve this:

在执行批量读取操作时，应用程序可能希望禁用缓存，这样批量读取所处理的数据就不会最终替换掉大部分已缓存的内容。可以使用一个针对迭代器的选项来实现这一点：

```c++
leveldb::ReadOptions options;
options.fill_cache = false;
leveldb::Iterator* it = db->NewIterator(options);
for (it->SeekToFirst(); it->Valid(); it->Next()) {
  ...
}
```

### Key Layout 键布局

Note that the unit of disk transfer and caching is a block. Adjacent keys
(according to the database sort order) will usually be placed in the same block.
Therefore the application can improve its performance by placing keys that are
accessed together near each other and placing infrequently used keys in a
separate region of the key space.

注意，磁盘传输和缓存的单位是一个数据块，相邻的键（按照数据库的排序顺序）通常会被防止在同一个数据块中。因此，应用程序可以通过将经常一起被访问的键彼此放置得靠近一些，并将不常用的键放置在键空间的一个单独取余，来提高其性能。

For example, suppose we are implementing a simple file system on top of leveldb.
The types of entries we might wish to store are:

例如，假设我们要再LevelDB之上实现一个简单的文件系统。我们可能希望存储的条目类型有：

    filename -> permission-bits, length, list of file_block_ids
    file_block_id -> data

We might want to prefix filename keys with one letter (say '/') and the
`file_block_id` keys with a different letter (say '0') so that scans over just
the metadata do not force us to fetch and cache bulky file contents.

我们可能想要给文件名键假设一个前缀（比如'/')，给文件块ID键加上一个不同的前缀（比如'0')，这样仅对元数据进行扫描时就不会迫使我们去获取大量的文件内容。

### Filters 过滤器

Because of the way leveldb data is organized on disk, a single `Get()` call may
involve multiple reads from disk. The optional FilterPolicy mechanism can be
used to reduce the number of disk reads substantially.

由于LevelDB数据在磁盘上的组织方式，一次单独的Get()调用可能涉及到从磁盘进行多次读取。可选的过滤器策略可用于大幅减少磁盘读取的次数。

```c++
leveldb::Options options;
options.filter_policy = NewBloomFilterPolicy(10);
leveldb::DB* db;
leveldb::DB::Open(options, "/tmp/testdb", &db);
... use the database ...
delete db;
delete options.filter_policy;
```

The preceding code associates a Bloom filter based filtering policy with the
database.  Bloom filter based filtering relies on keeping some number of bits of
data in memory per key (in this case 10 bits per key since that is the argument
we passed to `NewBloomFilterPolicy`). This filter will reduce the number of
unnecessary disk reads needed for Get() calls by a factor of approximately
a 100. Increasing the bits per key will lead to a larger reduction at the cost
of more memory usage. We recommend that applications whose working set does not
fit in memory and that do a lot of random reads set a filter policy.

前面的代码将一个基于布隆过滤器的过滤策略与数据库相关联。基于布隆过滤器的过滤依赖于在内存中为每个键保留一定数量的数据位（在这种情况下，每个键保留10位数据，因为这是我们传递给NewBloomFilterPolicy函数的参数）。这种过滤器将把Get()调用所需的不必要磁盘读取次数减少大约100倍。增加每个键的位数将以占用更多内存为代价带来更大程度的减少。我们建议那些工作集无法装入内存且进行大量随机读取的应用程序设置一个过滤策略。

If you are using a custom comparator, you should ensure that the filter policy
you are using is compatible with your comparator. For example, consider a
comparator that ignores trailing spaces when comparing keys.
`NewBloomFilterPolicy` must not be used with such a comparator. Instead, the
application should provide a custom filter policy that also ignores trailing
spaces. For example:

如果你使用自定义的比较器，那么你应该确保过滤策略与该比较器兼容。例如，考虑一种在比较键时会忽略末尾空格的比较器。
`NewBloomFilterPolicy`函数绝不能与这样的比较器一起使用。相反，应用程序应该提供一个同样会忽略末尾空格的自定义过滤策略。例如：

```c++
class CustomFilterPolicy : public leveldb::FilterPolicy {
 private:
  FilterPolicy* builtin_policy_;

 public:
  CustomFilterPolicy() : builtin_policy_(NewBloomFilterPolicy(10)) {}
  ~CustomFilterPolicy() { delete builtin_policy_; }

  const char* Name() const { return "IgnoreTrailingSpacesFilter"; }

  void CreateFilter(const Slice* keys, int n, std::string* dst) const {
    // Use builtin bloom filter code after removing trailing spaces
    std::vector<Slice> trimmed(n);
    for (int i = 0; i < n; i++) {
      trimmed[i] = RemoveTrailingSpaces(keys[i]);
    }
    return builtin_policy_->CreateFilter(trimmed.data(), n, dst);
  }
};
```

Advanced applications may provide a filter policy that does not use a bloom
filter but uses some other mechanism for summarizing a set of keys. See
`leveldb/filter_policy.h` for detail.

高级应用程序可能会提供一种不使用布隆过滤器，而是使用其他某种机制来汇总一组键的过滤策略。详情请参阅`leveldb/filter_policy.h`。

## Checksums 校验和

leveldb associates checksums with all data it stores in the file system. There
are two separate controls provided over how aggressively these checksums are
verified:

LevelDB会为其存储在文件系统中的所有数据关联校验和。对于校验和的验证强度，有两个独立的控制设置：

`ReadOptions::verify_checksums` may be set to true to force checksum
verification of all data that is read from the file system on behalf of a
particular read.  By default, no such verification is done.

可以将`ReadOptions::verify_checksums`设置为true，以便于针对特定的读取操作，强制对从文件系统读取的所有数据进行校验和验证。默认情况下，不会进行此类验证。

`Options::paranoid_checks` may be set to true before opening a database to make
the database implementation raise an error as soon as it detects an internal
corruption. Depending on which portion of the database has been corrupted, the
error may be raised when the database is opened, or later by another database
operation. By default, paranoid checking is off so that the database can be used
even if parts of its persistent storage have been corrupted.

在打开数据库之前，可以将`Options::paranoid_checks`设置为true，这样一旦数据库实现检测到内部损坏，就会立即引发错误。根据数据库中哪部分已损坏，错误可能会在打开数据库时引发，也可能会在之后的其他数据库操作中引发。默认情况下，偏执检查是关闭的，这样即使其持久化存储的部分内容已损坏，数据库仍可使用。

If a database is corrupted (perhaps it cannot be opened when paranoid checking
is turned on), the `leveldb::RepairDB` function may be used to recover as much
of the data as possible

如果数据库已损坏（可能在开启偏执检查时无法打开），则可以使用`leveldb::RepairDB`函数来尽可能多地恢复数据。

## Approximate Sizes 近似大小

The `GetApproximateSizes` method can used to get the approximate number of bytes
of file system space used by one or more key ranges.

`GetApproximateSizes`方法可用于获取一个或多个键范围所占用的文件系统空间近似字节数。

```c++
leveldb::Range ranges[2];
ranges[0] = leveldb::Range("a", "c");
ranges[1] = leveldb::Range("x", "z");
uint64_t sizes[2];
db->GetApproximateSizes(ranges, 2, sizes);
```

The preceding call will set `sizes[0]` to the approximate number of bytes of
file system space used by the key range `[a..c)` and `sizes[1]` to the
approximate number of bytes used by the key range `[x..z)`.

前面的调用会将sizes[0]设置为键范围`[a..c)`所占用的文件系统空间的大致字节数，sizes[1]设置为键范围`[x..z)`所占用的大致字节数。

## Environment 环境

All file operations (and other operating system calls) issued by the leveldb
implementation are routed through a `leveldb::Env` object. Sophisticated clients
may wish to provide their own Env implementation to get better control.
For example, an application may introduce artificial delays in the file IO
paths to limit the impact of leveldb on other activities in the system.

LevelDB实现所发起的所有文件操作（以及其他操作系统调用）都通过一个`leveldb::Env`对象进行路由。复杂的客户端
可能希望提供自己的Env实现以便更好地进行控制。 
例如，一个应用程序可能会在文件IO路径中引入人为的延迟，以限制LevelDB对系统中其他活动的影响。

```c++
class SlowEnv : public leveldb::Env {
  ... implementation of the Env interface ...
};

SlowEnv env;
leveldb::Options options;
options.env = &env;
Status s = leveldb::DB::Open(options, ...);
```

## Porting 移植

leveldb may be ported to a new platform by providing platform specific
implementations of the types/methods/functions exported by
`leveldb/port/port.h`.  See `leveldb/port/port_example.h` for more details.

通过提供`leveldb/port/port.h`所导出的类型/方法/函数的特定于平台的实现，可以将LevelDB移植到一个新的平台上。更多的细节请参见`leveldb/port/port_example.h`。

In addition, the new platform may need a new default `leveldb::Env`
implementation.  See `leveldb/util/env_posix.h` for an example.

此外，新平台可能需要一个新的默认`leveldb:Env`实现。可参考`leveldb/util/env_posix.h`中的示例。

## Other Information 其他信息

Details about the leveldb implementation may be found in the following
documents:

1. [Implementation notes](impl.md)
2. [Format of an immutable Table file](table_format.md)
3. [Format of a log file](log_format.md)

有关LevelDB实现的详细信息可在以下文档中找到：

1. [实现说明](impl.md)
2. [不可变表文件的格式](table_format.md)
3. [日志文件的格式](log_format.md)