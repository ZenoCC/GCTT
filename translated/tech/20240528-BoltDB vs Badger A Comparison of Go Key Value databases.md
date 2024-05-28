BoltDB vs Badger: A Comparison of Go Key-Value databases
 Tuesday, Jan 29, 2019  Tim Shannon  7 min  7 Comments

# BoltDB vs Badger: Go 键值数据库对比
周二，2019/1/29 Tim Shannon

When I first started working on BoltHold (a simple querying and index engine that sits on top of BoltDB), Badger didn’t yet exist, and BoltDB was the clear leader of the pack for key-value, pure-go, embeddable databases.

当我第一次使用BoltHold(一个位于BoltDB之上的简单查询和索引引擎)时，Badger还不存在，BoltDB当时是键值、XX、可嵌入数据库的明确领导者。

Then Badger was released, and it was shown to be more than just a pure-go version of LSM-tree stores like RocksDB / LevelDB, it actually was faster than RocksDB. Much faster. I knew I wanted to build something with Badger in the future, and when an issue was opened to add Badger support to Bolthold, I jumped on it.

后来随着Badger的发布，事实证明它不仅仅是类似RocksDB、LevelDB等通过LSM树存储 的纯go版，它实际上比RocksDB更快，快得多。当 Bolthold可支持Badger 这个问题待解决时，我决定参与到其中。

The first thing I had to consider in adding Badger support was whether I wanted to add Badger as another dependency and allow users to switch between the two via a configuration flag, or build an entirely separate library. I ended up following the Go proverb:

在支持Badger时，我必须考虑的一件事是：将支持Badger为另一个依赖项，允许用户通过配置标志在两者之间切换，还是构建一个完全独立的库。我最终遵循了众所周知的的Go语言的特性：

A little copying is better than a little dependency
一点复制总比一点依赖性好

I decided to make BadgerHold a separate library, and now that I have 88% test coverage using the exact same tests as BoltHold, I believe that I made the right choice.

我决定让BadgerHold成为一个单独的库，现在我使用与BoltHold完全相同的测试完成了88%的测试覆盖率，我相信我做出了正确的选择

In implementing the exact same library on two different KV stores, I like to think I have gained some perspective on comparing these two databases. Dgraph have already done an excellent comparison of performance between these two libraries. This post instead will focus on comparing criteria other than performance.

在两个不同的KV存储上实现完全相同的库时，我想我已经从比较这两个数据库中获得了一些观点。Dgraph数据库已经对这两个库之间的性能进行了出色的比较。相反，这篇文章将专注于比较性能以外的标准。

Manish Jain from Dgraph has also been kind enough to offer some responses to my limited perspectives in my comparisons below. I’ve included them in-line.

Dgraph的Manish Jain也很友好地对我有限的观点做出了一些回应。我已经把它们包括在内了

Superficial Differences  浅显的差异

When I first approached writing BadgerHold, I was already familiar with BoltDB and had used it on several other projects. Unfortunately, this meant that the onus was on Badger to meet or conflict with my expectations upon first using the library. With that in mind, I found the following differences to be superficial, and not a major impact in my usage of both libraries. That being said, I felt I should list them in case they prove to be more important to you.

当我首次使用BadgerHold时，我已经很熟悉BoltDB并已经在几个项目中使用过它。不幸的是，这意味着在首次使用这个库时，Badger需要满足或与我的期望相冲突。考虑到这一点，我发现以下差异是很浅显的，对我对两个库的使用没有重大影响。话虽如此，我觉得还是我应该把它们列出来，以防这些差异对你来说更重要。

## File Handling  文件处理（句柄）

With BoltDB, you open a database by specifying a file, and passing in some options:

使用BoltDB时，您可以通过指定文件并传递一些选项来打开数据库

`db, err := bolt.Open("my.db", 0600, options)
`
Badger, on the other hand, requires you to specify a multiple folders as part of an options struct instead of a single file:

另一方面，Badger要求您指定多个文件夹作为选项结构的一部分，而不是单个文件：

```opts := badger.DefaultOptions
opts.Dir = "/tmp/badger"
opts.ValueDir = "/tmp/badger"
db, err := badger.Open(opts)
```

This is due to the nature of LSM-tree databases, where multiple levels are stored across multiple files. I was hard pressed to come up with a legitimate reason why a single file would matter over a directory of files, but Manish explained the reasoning behind the choice.

这是因为LSM-tree 数据库的性质如此，需要多个级别存储在多个文件中。我很难想出一个合理的理由来解释为什么单个文件会比文件目录更重要，但Manish解释了选择背后的理由。

The benefit of having files be immutable is that they get rsync friendly. That was an explicit requirement for LevelDB at Google.

– Manish Jain

让文件不可变的好处是它们变得对rsync（文件同步）友好。这是谷歌LevelDB的明确要求
– Manish Jain

Unexpected API Choices。 意想不到的API选择

In general I found BoltDB to fall more in line with what I would expect from a Go API compared to Badger. My expectations being defined by those that I am used to seeing in the Go standard library.

总的来说，我发现与Badger相比，BoltDB更符合我对Go API的期望。我的期望来自我已经习惯在Go标准库中看到的那些。

For example, a common way to handle options in the Go standard library is to default values if not specified (see http.Server, and sql.Conn).

例如，处理Go标准库中选项的常见方法是，如果未指定，则为默认值（请参阅http.Server和sql.Conn）。

Bolt’s Open method will accept a nil option and open up the data file with defaults.

Bolt的Open方法将接受零选项，并以默认值打开数据文件。

db, err := bolt.Open("my.db", 0600, nil)

Badger’s option argument doesn’t accept a pointer, so a struct needs to be passed in. If you want to use the default options, you need to make a copy of an exported struct that contains defaults. Furthermore, you’ll always need to change those defaults because the data directories are stored in the options type.

Badger的选项参数不接受指针，因此需要传递一个结构。如果您想使用默认选项，您需要制作包含默认值的导出结构的副本。此外，因为数据目录存储在选项类型中，您始终需要更改这些默认值。

```
opts := badger.DefaultOptions
opts.Dir = "/tmp/badger"
opts.ValueDir = "/tmp/badger"
db, err := badger.Open(opts)

```
That’s intended. It allows flexibility in how you want Badger to behave, and we want users to have a look at the default options and tweak them.

– Manish Jain

那是故意的。它允许在您希望Badger的行为方式上保持灵活性，并且我们希望用户查看默认选项并调整它们。
– Manish Jain

In the standard Go library, constants and enumerations are usually stored within the same package where they are used, or local enumerations defined from external packages exported enumerations (see gzip and os).

在标准Go库中，常量和枚举通常存储在使用它们的同一软件包中，或是从导出的外部软件包中定义的本地枚举（见gzip和os）。

Badger requires you to import the badger/options package to get the enumerations for FileLoadingMode.

Badger要求您导入bagger或选项包 以获取FileLoadingMode的枚举。

Behavior Differences  行为差异

The following are more significant differences between how BoltDB and Badger behave.

以下是BoltDB和Badger行为之间更显著的差异。

## Buckets 

BoltDB has a concept of buckets in a single database. Comparable to tables in a relational database, this allows you to store different types of data in the same database, and not worry about row conflicts.

BoltDB在单个数据库中有一个桶的概念。与关系数据库中的表相比，这允许您在同一数据库中存储不同类型的数据，并且不必担心行冲突。

In BoltHold, I used different buckets to store different types. Since Badger doesn’t have similar functionality, I had to replicate it by prefixing keys with the type being stored, after which I could use Badger’s ValidForPrefix method when iterating to make sure queries only apply to the proper type. This allows the behavior between BoltHold and BadgerHold to be the same when storing different types in the same database. BoltDB most likely does something similar to create it’s bucket functionality, but it’s nice not to have to re-implement it.

在BoltHold中，我使用不同的桶来存储不同的类型。由于Badger没有类似的功能，我不得不通过将键与所存储的类型的前缀进行复制，之后我可以使用Badger的ValidForPrefix方法，以确保遍历查询仅适用于正确的类型。这允许在同一数据库中存储不同类型时，BoltHold和BadgerHold之间的行为是相同的。BoltDB很可能做类似的事情来创建它的桶功能，但不必重新实现它是件好事。


## Iterators 迭代器

BoltDB allows you to create a number of Cursors on any given transaction. Badger, on the other hand, only allows one iterator at a time in Read / Write transactions.

BoltDB允许您在任何给定的处理上创建多个游标。另一方面，Badger在读/写处理中一次只允许一个iterator。

In BoltHold and BadgerHold, you have the ability to write subqueries, which allow you to filter records based on another query from within that same transaction.

在BoltHold和BadgerHold中，您可以编写子查询，这允许您根据同一处理中的另一个查询过滤记录。

```bh.Where("Name").MatchFunc(func(ra *bh.RecordAccess) (bool, error) {
	// find where name exists in more than one category
	record, ok := ra.Record().(*ItemTest)
	if !ok {
		return false, fmt.Errorf("Record is not ItemTest, it's a %T", ra.Record())
	}

	var result []ItemTest

	err := ra.SubQuery(&result,
		bh.Where("Name").Eq(record.Name).And("Category").Ne(record.Category))
	if err != nil {
		return false, err
	}

	if len(result) > 0 {
		return true, nil
	}

	return false, nil
})
```

This type of query is much harder to implement in Badger for Update and Delete queries than in BoltDB due to this single iterator limit from within R/W transactions. I’ve since opened an issue, as this limitation may be fixable.
由于读/写处理的单个迭代器限制，相较在BoltDB中的更新删除查询，在Badger中更难实现。我已经打开了一个问题，因为这个限制是可以解决的。

## Memory Usage 内存使用

The single most surprising difference for me between BoltDB and Badger, was the disparity in memory usage. In my testing I found Badger to use significantly more memory than BoltDB. So much so that I had to rewrite my BadgerHold tests from using a separate database for each test to using a single shared database for all tests. I couldn’t complete the entire suite of tests on my 8GB VM due to out of memory errors. Even after changing all of Badger’s options to their lowest memory settings, I was unable to get the full suite to finish.

对我来说，BoltDB和Badger之间最令人惊讶的区别是内存使用的差异。在测试中，我发现Badger使用的内存比BoltDB多得多。以至于我不得不重写我的BadgerHold测试，从对每个测试使用单独的数据库到对所有测试使用单个共享数据库。由于内存不足错误，我无法在我的8GB虚拟机上完成整套测试。即使在将Badger的所有选项更改为最低内存设置后，我也无法完成整个套件。

To be fair, this is most likely by design, and the source of a lot of the impressive performance differences between BoltDB and Badger. The memory usage was also exasperated by opening multiple databases, which meant there needed to be enough memory for each of those database’s buffer cache. There are many scenarios where this trade off of memory for speed is well worth the cost, however, I was personally surprised by the extent of it, as I hadn’t seen it mentioned in any of the documentation or blog posts.

公平地说，这很可能是设计上的，也是BoltDB和Bdger之间许多令人印象深刻的性能差异的根源。打开多个数据库也加剧了内存使用情况，这意味着每个数据库的缓冲区缓存都需要有足够的内存。在许多情况下，这种将内存换取速度是非常值得的，然而，我个人对它的程度感到惊讶，因为我没有在任何文档或博客文章中提到过它。

## Summary 摘要

Criteria	     BoltDB	               Badger
File Handling	Single File	     Multiple Directories
API	Idiomatic	Unexpected
Bucket Support	Yes	No
Iterators	No Restrictions	One R/W iterator at a time
Memory Usage	Low	High


|  标准    |   Bolt DB     |    Bdger   |
| ------  |   ---------   |------- |
| 文件处理|    单个文件     |   多个目录    |
| API    |      惯用标准   |   异常较多    |
| Bucket |     支持       |     不支持  |
| 迭代器  |     无限制      |    一次仅支持1个   |
| 内存使用 |     低         |    高   |


The biggest differences between BoltDB and Badger, stem from their underlying technologies, and not their implementation details. If you are picking one over the other based solely on performance metrics, I would closely consider your usage case instead.

BoltDB和Badger之间最大的区别来自它们的基础技术，而不是它们的实施细节。如果您仅根据性能指标选择一个，我会仔细考虑您的使用案例。

My recommendation would be to use BoltDB for scenarios where you are more memory constrained or where the performance gain isn’t worth the cost in memory usage. Examples of this may be projects on mobile phones, tablets, Raspberry Pi-like devices or command line applications where you need some persistent storage.

我建议在内存受限或性能增益不值得内存使用成本的情况下使用BoltDB。这方面的例子可能是手机、平板电脑、类似树莓派的设备或命令行应用程序上的项目，您需要一些持久存储。

For micro-services and projects you’d tend to run on a server where more memory is available, or projects where you expect a lot of writes like for logging, I would recommend Badger.

对于您倾向于在有更多内存的服务器上运行的微服务和项目，或者您期望像日志记录这样的大量写入的项目，我推荐Badger。

I would also recommend using one shared Badger database, rather then opening multiple separate databases to limit your memory usage. Using a library like BadgerHold, or using similar key-prefixing methods in your own code, should allow you to manage your data types separately within the same database.

我还建议使用一个共享Badger数据库，而不是打开多个单独的数据库来限制您的内存使用。使用像BadgerHold这样的库，或在自己的代码中使用类似的密钥前缀方法，应该允许您在同一数据库中单独管理数据类型。

If you choose to use BoltHold, or BadgerHold, I would recommend aliasing the package import as bh and since both library’s APIs are nearly identical, you could easily switch your project from one to another if you find that one library will work better for you.

如果您选择使用BoltHold或BadgerHold，我建议将软件包导入别名为bh，因为两个库的API几乎相同，如果您发现一个库更适合您，您可以轻松地将项目从一个切换到另一个。


```import (
	bh "github.com/timshannon/badgerhold"
)

// this code will run the same on BoltHold and BadgerHold

store.UpdateMatching(&Person{}, bh.Where("Death").Lt(bh.Field("Birth")), func(record interface{}) error {
	update, ok := record.(*Person)
	if !ok {
		return fmt.Errorf("Record isn't the correct type!  Wanted Person, got %T", record)
	}

	update.Birth, update.Death = update.Death, update.Birth

	return nil
})

```

---

via: https://tech.townsourced.com/post/boltdb-vs-badger/

作者：[Tim Shannon](https://twitter.com/townsourced)
译者：[译者ID](https://github.com/ZenoCC)
校对：[校对者ID](https://github.com/校对者ID)

本文由 [GCTT](https://github.com/studygolang/GCTT) 原创编译，[Go 中文网](https://studygolang.com/) 荣誉推出