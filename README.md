# xray_splice-

最近我天天读代码，已经懵了。终于渐进到核心区域？！据说splice才是xray大杀器，读一下！


## splice到底啥原理？


首先，在xray代码中直接搜索splice

https://github.com/XTLS/Xray-core/search?q=splice


先关注 splice 和 ReadV关联的代码。

到底啥是ReadV啥是splice我可不懂，但是最开始xtls- 那个文章已经指出了，v2ray也是被提交过PR的，导致v2ray现在也是有readv的。如果v2ray也能用splice那是不是性能也能大增？所以这里是个重点



搜索 ReadV 有六个文件冒出来

1. common/buf/readv_reader.go
2. common/buf/readv_reader_wasm.go
3. common/buf/readv_test.go
4. common/buf/io.go
5. proxy/vless/encoding/encoding.go
6. proxy/trojan/protocol.go


首先，排除test文件 和 wasm文件； 那么核心就是 readv_reader.go，common/buf/io.go，proxy/vless/encoding/encoding.go， 然后trojan估计和vless类似。


