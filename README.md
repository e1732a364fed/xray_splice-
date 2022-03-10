# xray_splice-

最近我天天读代码，已经懵了。终于渐进到核心区域？！据说splice才是xray大杀器，读一下！


## splice到底啥原理？


首先，在xray代码中直接搜索splice

https://github.com/XTLS/Xray-core/search?q=splice

有5个文件冒出来

1. infra/conf/trojan.go
2. proxy/trojan/protocol.go
3. infra/conf/vless.go
4. proxy/vless/encoding/encoding.go
5. proxy/vless/vless.go

这里面，似乎 proxy/vless/encoding/encoding.go 比较重要，而 splice在其中的 ReadV函数中出现


先关注 splice 和 ReadV关联的代码。

到底啥是ReadV啥是splice我可不懂，但是最开始xtls- 那个文章已经指出了，v2ray也是被提交过PR的，导致v2ray现在也是有readv的。如果v2ray也能用splice那是不是性能也能大增？所以这里是个重点


搜索 ReadV，13个结果

1. common/buf/readv_reader.go
2. common/buf/readv_reader_wasm.go
3. common/buf/readv_unix.go
4. common/buf/readv_posix.go
5. common/buf/readv_test.go
6. proxy/vless/encoding/encoding.go
7. proxy/trojan/protocol.go
8. common/buf/io.go
9. proxy/trojan/client.go
10. proxy/vless/outbound/outbound.go
11. proxy/trojan/server.go
12. proxy/vless/inbound/inbound.go
13. testing/scenarios/vmess_test.go


## ReadV函数

我们首先阅读 proxy/vless/encoding/encoding.go 的 ReadV函数

首先看 https://github.com/XTLS/Xray-core/blob/e93da4bd02f2420df87d7b0b44412fbfbad7c295/proxy/vless/encoding/encoding.go#L209

这里 的 panic是 `panic("XTLS Splice: not TCP inbound")`, 果然ReadV和splice是有关系的吧？？

首先里面定义了一个函数，然后返回了一个err，不知何故，为什么不直接放到代码里呢？

首先是一个循环，然后判断 xtls.Conn 的 DirectIn, 这些我在 xtls解读时读过。这里竟然把 conn.DirectIn 又设成了false？

然后是从 context上下文中提取出 inbound，然后确定 iConn 的具体的值，然后如果 是 net.TCPConn 的话，似乎有更多判断，继续看

首先rprx直接就告诉我们，这里就是splice的部分 `fmt.Println(conn.MARK, "Splice")`, 然后紧接着是如下代码

```go
runtime.Gosched() // necessary
w, err := tc.ReadFrom(conn.Connection)
if counter != nil {
  counter.Add(w)
}
if statConn != nil && statConn.WriteCounter != nil {
  statConn.WriteCounter.Add(w)
}
return err
```

核心就是 `tc.ReadFrom(conn.Connection)`, tc就是 上面从上下文提取出的 inbound中 提取出的 conn的具体的值，是 `*net.TCPConn`,

然后它直接从xtls的连接里读取数据到 `*net.TCPConn` 了，然后读完就返回了，和后面的 buf.NewReadVReader 调用一点关系也没有！

好像也没什么神奇的，咋就 splice了呢？？

### 谁调用的ReadV？

重新回到搜索页面，proxy/vless/inbound/inbound.go，proxy/vless/outbound/outbound.go, 以及trojan的server.go, 只有这三个地方调用到了！

我对vless熟悉一点，先读vless

关注这里 https://github.com/XTLS/Xray-core/blob/578d903a9e4c9c0f2b766da45f157613ae83e9b4/proxy/vless/outbound/outbound.go#L239

它是在 getResponse里，和 rawConn 有关系，它整个函数的调用是：

`err = encoding.ReadV(serverReader, clientWriter, timer, iConn.(*xtls.Conn), rawConn, counter, sctx)`


那么再回过头看 readv函数，发现 rawConn根本没用到？？counter似乎是计算流量的，也不用管，前两个 传入的 serverReader 和 clientWriter 在splice过程中也没用到！

也就是说，真正用到的只有 xtls.Conn 和 上下文提出来的 `*net.TCPConn`

那么我们就有所理解了。实际上rprx只是 利用了一下“readv”，把tcp连接的情况隔离了出来，在这种情况下，实际上并不使用readv函数，而是直接用rprx的 `tc.ReadFrom(conn.Connection)` 函数

在inbound.go里的用法应该是类似的，只是是在 postRequest 里。

仔细思考，outbound的 getResponse，实际上就是 客户端 从 服务端读 的过程； inbound 的 postRequest，是向 客户端 发送数据的过程

也就是说，在 ` 客户端 <- 服务端 ` 这个流向里，使用到了splice

## 什么是ReadFrom

到底什么情况？为什么 `tc.ReadFrom(conn.Connection)` 会用到splice？

总之首先要明白，这个conn是 xtls的Conn，然后它的Connection 就是我 xtls- 文章里面说的那个，从一个私有成员升级为公开成员的 那个 Connection

这个ReadFrom要直接查看golang的 net包

具体是这个 https://cs.opensource.google/go/go/+/refs/tags/go1.17.8:src/net/tcpsock_posix.go;drc=refs%2Ftags%2Fgo1.17.8;l=48

这里用到了splice！

总之，现在可以断定，splice与readv一毛钱关系都没有，只是rprx硬放到了readv函数里 进行处理

readfrom的内容：

```go
func (c *TCPConn) readFrom(r io.Reader) (int64, error) {
	if n, err, handled := splice(c.fd, r); handled {
		return n, err
	}
	if n, err, handled := sendFile(c.fd, r); handled {
		return n, err
	}
	return genericReadFrom(c, r)
}
```

splice 的具体内容： https://cs.opensource.google/go/go/+/refs/tags/go1.17.8:src/net/splice_linux.go;drc=refs%2Ftags%2Fgo1.17.8;l=17

```go

// splice transfers data from r to c using the splice system call to minimize
// copies from and to userspace. c must be a TCP connection. Currently, splice
// is only enabled if r is a TCP or a stream-oriented Unix connection.
//
// If splice returns handled == false, it has performed no work.
func splice(c *netFD, r io.Reader) (written int64, err error, handled bool) {
	var remain int64 = 1 << 62 // by default, copy until EOF
	lr, ok := r.(*io.LimitedReader)
	if ok {
		remain, r = lr.N, lr.R
		if remain <= 0 {
			return 0, nil, true
		}
	}

	var s *netFD
	if tc, ok := r.(*TCPConn); ok {
		s = tc.fd
	} else if uc, ok := r.(*UnixConn); ok {
		if uc.fd.net != "unix" {
			return 0, nil, false
		}
		s = uc.fd
	} else {
		return 0, nil, false
	}

	written, handled, sc, err := poll.Splice(&c.pfd, &s.pfd, remain)
	if lr != nil {
		lr.N -= written
	}
	return written, wrapSyscallError(sc, err), handled
}

```




