# readv-

在上一篇解析splice文章中，看到了这么一句

```
reader = buf.NewReadVReader(conn.Connection, rawConn, nil)
```

虽然splice和readv毫无关系，但是在ReadV函数中，如果无法执行splice，就是会执行readv的，所以有必要研究一下readv

## readv到底啥原理？

https://zhuanlan.zhihu.com/p/341366946

相比read和write一次只能下发一个IO请求，并将数据读写到一个指定的缓冲区，readv和writev可以一次性指定多个缓冲区进行读写。而且readv专门需要提供一个fd

https://github.com/v2fly/v2ray-core/issues/416#issuecomment-726743503

没想到效果这么好

关于readv和splice：

https://github.com/v2fly/v2ray-core/issues/416#issuecomment-727140721

似乎这条评论就是xtls的splice的发源地

那我们看一下代码

## 代码分析


解读 MultiBuffer

common/buf/multi_buffer.go

MultiBuffer 只不过是 一串 [][]byte

里的 SplitBytes 函数，如下
func SplitBytes(mb MultiBuffer, b []byte) (MultiBuffer, int) {


实际上 分别依次从 mb 的各个buf 里读取数据，然后存放到b中

然后读完之后，mb的数组就会变成0 长度。

总之这个 SplitBytes 名字起的不好，和实际操作不符


readv_reader.go

NewReadVReader 传入 reader io.Reader, rawConn syscall.RawConn
然后内部还会新增一个  multiReader

而 newMultiReader 这个函数是 系统相关的

分别存在于 readv_posix.go, readv_unix.go, readv_windows.go

参考 `_unix.go`, 就是 这个 multiReader 就是一个  `[][]byte` 结构

而Read的话，直接使用 golang.org/x/sys/unix.Readv

它的Read的参数是一个 fd uintptr，就是说是要读取一个文件的。

而 ReadVReader会调用 r.RawConn.Read, 而这个Read的参数 是一个函数！

这个函数这里直接就是从 multiReader里读，而那个文件符号则是 RawConn会提供的

而读取的值是可以直接获取的，因为它提前用了 `r.mr.Init(bs)`，也就是说，multiReader的缓存是我们自己提供的！

然后buffer.go 里，的Buffer 结构，和 bytes.Buffer是类似的，不过它是直接内置了从Pool获取的过程

ReadVReader.readMulti 就是从 之前初始化时 传入的 rawConn里通过 multiReader 读取数据，然后返回 一个 MultiBuffer

ReadMultiBuffer
如果 allocStrategy=1的话，就不使用multiReader，而是直接从 之前初始化时 传入的 reader里读取

而不为1的话，就调用 readMulti 


## writev

直接 搜索 writev是搜不到代码的。但是 在writer.go 中，使用了 `nb := net.Buffers(bs)`, 而golang官方的文档是：

```
type Buffers [][]byte
net.Buffers on pkg.go.dev

Buffers contains zero or more runs of bytes to write.

On certain machines, for certain types of connections, this is optimized into an OS-specific batch write operation (such as "writev").
```

就是说这个操作 和 splice的 ReadFrom道理一样，都是让golang去处理底层问题。 不错哦。










