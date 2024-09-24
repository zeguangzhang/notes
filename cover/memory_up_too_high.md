# 取内容内图片作为封面图的选择列表，另一天来同事的一个id发现内存飙升问题
```
- 主要业务，需要把内容中的图片下载下来，判断哪些符合我们规格，符合的留下。
- 筛选规则：等宽，宽高比 16/9 3/4，宽高都大于400像素，size<5mb
```
## 开始代码关闭释放查看
- 看下代码 ，没有内存泄漏现象，而且我开发完自测时用了几个id测试的图片下载都没出现这种现象
## 内存工具 pprof分析
- 首先，需要在代码中引入pprof，通常你会在 main 函数中进行设置。
```golang
    import (
        "net/http"
        _ "net/http/pprof" // 引入 pprof
    )
    
    func main() {
        // 启动 HTTP 服务监听
        go func() {
            http.ListenAndServe("localhost:6060", nil)  // 默认会暴露 pprof 接口
        }()
    
        // 你的应用代码
    }
```

- 运行main程序，在接口接近完成时设置断点
```text
- 查看所有的pprof调试数据，没太细看，感觉看不懂，应该一般也不需要看
http://localhost:6060/debug/pprof/
- 内存使用情况：
http://localhost:6060/debug/pprof/heap
```

- 导出此时内存快照
```bash
    curl -o heap.out http://localhost:6060/debug/pprof/heap
```

- 启动快照分析
```bash
    go tool pprof -http=:1212 heap.out 
```

- 浏览器打开：http://localhost:1212/ui
```text
在浏览器上如果你收到 Could not execute dot; may need to install graphviz 错误，说明你的系统缺少 Graphviz 工具。
Graphviz 是一个用于生成图形化展示的工具，pprof 需要它来生成图表
 在 macOS 上：你可以通过 Homebrew 来安装 Graphviz
 brew install graphviz
```

- graphviz安装完成后，可以正常浏览的，VIEW筛选Top,SAMPLEX查看发现top ,top显示 decode最大,约1g。
```golang
    config, _, err := image.DecodeConfig(reader)
    if err != nil {
        return 0, 0, nil, 0, err
    }
    width := config.Width
    height := config.Height
```