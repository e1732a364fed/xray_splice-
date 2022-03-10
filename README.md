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

到底啥是ReadV啥是splice我可不懂，但是最开始xtls- 那个文章已经指出了，v2ray也是被提交过PR的，导致v2ray现在也是有readv的。（不过经过后面仔细观察，发现现在v2ray的readv只是普通的readv函数，不是开头末尾都大写的 ReadV，ReadV才是关键！）如果v2ray也能用splice那是不是性能也能大增？所以这里是个重点


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

还是两个都是读取？反正这个似乎我不太懂，好像都是读取，可以接着往下文看，“读取” 是很关键的。

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

总之，只适用于 tcp之间，或者tcp和 unix domain socket 之间的数据传输，最后会用到 poll.Splice，poll你一点就会跳转到 https://pkg.go.dev/internal/poll

## 如何在其他地方实现 splice

那么虽然没用到readv，但是我们的连接都是基于tcp的啊，所以v2ray肯定是有可能实现splice的！

等等，既然这个ReadFrom这么好用，为什么不都用呢？

再查看ReadFrom的定义：

`ReadFrom implements the io.ReaderFrom ReadFrom method.`

再看 io.ReaderFrom

```
ReaderFrom is the interface that wraps the ReadFrom method.

ReadFrom reads data from r until EOF or error. The return value n is the number of bytes read. Any error except EOF encountered during the read is also returned.

The Copy function uses ReaderFrom if available.
```

读r 这个 Reader，一直读到 EOF 或者error，然后io.Copy 能用则也会用到。

哎？好像没啥的样子，比如我最新的 v2ray_simple项目的 流量转发就是很简单的两个语句：

```go
go io.Copy(wrc, wlc)
io.Copy(wlc, wrc)
```
是不是直接就用到了 Copy，然后就用到了 ReadFrom?

显然不是，为啥？因为 按理说，Copy只会去查看 wlc，wrc是没实现 ReaderFrom，而不知道你用没用到底层的tcp，也不会魔幻地去调用你底层tcp的ReadFrom，这就是关键

也就是说，想要转发流量，不能Copy最外层 的 Reader，除非你实现了 ReaderFrom，然后 直接或者间接地 调用了 `*net.TCP` 的 ReadFrom 函数才行

总之，就是实现一个ReadFrom 函数而已，也不难 啊！ 什么splice流控，怎么不叫 ReadFrom流控。。

然而，事实是这么简单吗？如果这样，为什么只有xtls有splice。是否和tls有关？

xtls之所以搞了一个Connection，实际上就是把tls的底层的connection暴露了出来，应该就是tcp，所以实际上就是两个tcp互相拷贝，用到splice函数

那么，我们摒弃xtls的其他不良代码，直接 复制golang的源代码，然后把Connection也暴露出来，不就一样了吗？

不过，仔细想想，如果这样，那么就没有经历过tls加密，实际上完全不需要 魔改tls，因为这就和 vless裸奔是一个概念，当然快了！

怪不得据说xtls快，因为它就是在用direct方式 下的 裸奔啊！ 二者缺一不可，1是必须direct，2是把底层连接暴露出来。这次Connection暴露出的根本原因终于知晓。


## 重新回顾xtls

根据简单分析，splice等同于裸奔。只不过由于裸的是https，所以是没关系的，所以还真就只能xtls自己用

总之，如果是裸奔的话，不用任何处理，golang默认就会调用splice函数

我再回顾一下xtls吧！ 看看有没有微弱的可能， 在绕过xtls的缺点的情况下，直接使用direct 方式。rprx自己说过是可能的？

基础不够的同学，还是再读一下我之前的文章： https://github.com/hahahrfool/xtls-/blob/main/README.md

首先，回顾splice发生的位置，是在 DirectIn，反过来说，DirectOut是不用管的。 而我暴露出的23 3 3 循环问题，是判断DirectOut的。似乎有戏？

重新回到 https://github.com/XTLS/Go/blob/main/conn.go ， 搜索DirectIn，发现一共出现四处，就是在Conn.Read中出现的

不过，它把DirectIn 重新设置成false有什么用吗？谁还会去调用Read函数呢？现在都已经直接开始裸奔对拷了

总之，关键不是设成false，而是判断true，true的条件是 c.DirectPre， 而 它发生于 readRecordOrCCS， 总之这一段是没有什么大问题的

但是，仅仅这样够不够？应该不够吧，一个巴掌拍不响啊，如果不判断 directout，直接判断 directin，那就相当于 发送时加密，读取时不加密，完全乱套了

不过rprx也说了

>Direct 其实并不需要魔改 TLS 库就可以实现，它不需要读过滤，甚至传输 TLSv1.3 时连写过滤都不需要，非常简洁、高效

不过其实我还是没搞懂，怎么过滤都不需要了呢？？是因为，只要 判断了 “写过滤”， 读过滤就是自然可以 推定的？ 

那么假设 只需判断 “写过滤”，为什么传输 TLS v1.3 时不需要写过滤？我 文章不是已经判断出了，所有的首包都要被判断吗？

总之，上文的意思似乎指明，只要能找到传输 TLS v1.3 时 不需任何过滤的办法，就可以搞定一切


写过滤，也就是write发生的过滤，也就是 服务端向客户端发送 远程https主机的 回应时 的过滤

问题就是，这里就是233 漏洞发生的核心部位啊！

rprx的意思是，取消这里也是可以的？？

我怀疑, 可能 和握手有关；也许直接判断握手包，就可以推定 所传递信息的内容，自然也就不需要对数据包部分 进行任何过滤

想一想也是有道理的哦？比如一些抓包软件，比如wireshark，tcpdump之类，不是可以显示应用层到底是什么协议的吗？它们是怎么探测的我们就怎么来呗。它们肯定也是能探测握手包的吧

既然rprx说了 tls1.3 可以做到不过滤，那么我们就直接研究tls1.3 好了。不过，首先要明白的是，如果我们只判断tls1.3，那么就只有在传输tls1.3是才能开启裸奔。

其他的也是一个道理的，必须能够判断 承载流量 是加密的，才能裸奔。

总之，我们先假设我们能通过 握手包过滤出来tls1.3流量，能不能不魔改tls 就做到直接 发送？

## 不魔改tls的情况

实际上，如果不魔改tls，那么本质上，我们就直接不使用tls

比如，我们直接用另一个 假想的 "tls_handshake" 包，建立好客户端和服务端的连接，然后模仿tls加密的办法，去加密 客户端发送的 clienthello，也就是第一条消息，然后发送到服务端

然后服务端获取到 后，解密 出来，并 研究发现它是clienthello握手包，然后 向远程服务器发送 clienthello；然后我们会收到serverhello，也能判断出来，


我暂时猜测，可能tls1.3 的0-rtt方式的话，只需要 两端的hello 包就能确认后面都是加密流量了，然后就直接裸连了？但是情况越来越苛刻


更关键的是，如果底层不是加密流量的话，还是要 手动去加密的，而且为了防探测还是要用tls加密。也就是说，只能魔改tls 来达到这种灵活的效果的

## tls 1.3

首先搜索到这个：
https://halfrost.com/https_tls1-3_handshake/

https://github.com/halfrost/Halfrost-Field/blob/master/contents/Protocol/TLS_1.3_Handshake_Protocol.md



总之，ClientHello 的结构体是已知的，可以直接一个clienthello就推定，然后其它就不管了？？





