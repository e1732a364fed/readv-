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

直接 搜索 writev是搜不到代码的。但是 在writer.go 中，BufferToBytesWriter.WriteMultiBuffer 使用了 `nb := net.Buffers(bs)`, 而golang官方的文档是：

```
type Buffers [][]byte

Buffers contains zero or more runs of bytes to write.

On certain machines, for certain types of connections, 
this is optimized into an OS-specific batch write operation (such as "writev").
```

就是说这个操作 和 splice的 ReadFrom道理一样，都是让golang去处理底层问题。 不错哦。



然后，在 ` nb.WriteTo(w.Writer)` 中，如果w.Writer实现了一个私有的 `writeBuffers(*Buffers) (int64, error)` 函数，就能够调用writev了。

很明显，w.Writer 必须是 net包中定义的那几种 Conn 才可以。

具体搜索net包，发现出现在如下位置

```
./fd_windows.go:121:func (c *conn) writeBuffers(v *Buffers) (int64, error) {
./fd_windows.go:125:	n, err := c.fd.writeBuffers(v)
./fd_windows.go:132:func (fd *netFD) writeBuffers(buf *Buffers) (int64, error) {
./net.go:684:// writeBuffers should fully consume and write all chunks from the
./net.go:687:	writeBuffers(*Buffers) (int64, error)
./net.go:704:		return wv.writeBuffers(v)
./writev_unix.go:15:func (c *conn) writeBuffers(v *Buffers) (int64, error) {
./writev_unix.go:19:	n, err := c.fd.writeBuffers(v)
./writev_unix.go:26:func (fd *netFD) writeBuffers(v *Buffers) (n int64, err error) {

```

然后 common/buf/io.go 中 的 NewWriter 函数中，会返回 BufferToBytesWriter

# readv在xray和v2ray中的使用范围

readv经过搜索，发现只在vless 和trojan中使用过，感觉xray 对于readv使用的还不是很广泛？

也许是因为这两个协议是不带加密的，所以才可以使用吧。但是最好还是能抽象出来，不要放到每一个支持的协议的代码内部

但是应用也基本仅限vless/trojan这种不加密的协议了, 而且实际查看发现只有direct流控时才能启用。因为如果是直连的话，Copy自动就可以splice；

而如果是加密协议的话，没发直接用readv,因为读到的数据还要解密，分裂成多个缓存没有意义。writev同理。

当然 对于不支持splice/sendfile的系统来说，直连时用readv可能也能提升性能

不过似乎也没发轻易从vless割裂出来？因为 vless的头部并不是用readv读取的，也没必要。就是说是要分开处理，头部正常读取，数据部分使用readv读取


而搜索v2ray，则发现readv函数虽然存在，却没有在trojan和vless里被用到。

这也是 我一开始误解认为readv是rprx写的原因。

不过，common/buf/io.go  里的 NewReader 函数 还是有用到的

根据 common/buf/readv_reader.go

readv在如下环境中被启用

```
if (runtime.GOARCH == "386" || runtime.GOARCH == "amd64" || runtime.GOARCH == "s390x") && (runtime.GOOS == "linux" || runtime.GOOS == "darwin" || runtime.GOOS == "windows") {
  useReadv = true
}
```

NewReader 中 readv的使用范围

```
_, isFile := reader.(*os.File)
	if !isFile && useReadv {
		if sc, ok := reader.(syscall.Conn); ok {
			rawConn, err := sc.SyscallConn()
			if err != nil {
				newError("failed to get sysconn").Base(err).WriteToLog()
			} else {
				return NewReadVReader(reader, rawConn)
			}
		}
	}
```

就是说，xray在vless和trojan中 明示了直接使用NewReadVReader，而v2ray是使用普通的 NewReader，然后让内部自行判断是否启用 readv

仔细想想, 真实使用范围似乎很狭窄，v2ray应该用不到
仅限于 直接从 底层连接读取数据，且不需要解密的情况；一旦外面包了一层 tls，那么 我们就无法接触底层连接，而是tls接触 底层连接、解密数据之后再提供给我们

所以说，readv的提升仅限于 不支持 splice 但却 确实是直接 直连读取的情况。 因为readv和writev的对象必须是 底层连接。

