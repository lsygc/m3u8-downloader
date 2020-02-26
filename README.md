# 带备注的 JSON 重排版工具
![界面](http://upyun.luckly-mjw.cn/Assets/m3u8-download/01.jpeg)
### [工具在线地址](http://blog.luckly-mjw.cn/tool-show/m3u8-downloader/index.html)

### 研发背景
- m3u8视频格式简介
    - m3u8视频格式：将完整的视频文件拆分成多个视频碎片文件(.ts)，.m3u8 文件详细记录每个视频片段的地址。
    - 视频播放时，会先读取 .m3u8 文件，再逐个下载，播放视频片段(.ts)文件。
    - 常用于直播业务，也常用该方法规避视频窃取的风险。加大视频窃取难度。
- 鉴于 m3u8 的以上特点，无法通过视频链接简单下载，需使用特定下载软件。
- 但软件下载过程繁琐，试错成本高。
- 使用软件的下载情况不稳定，常出现浏览器正常播放，但下载速度很慢，甚至无法正常下载的情况。
- 软件被编译打包，无法了解内部运行机制，不清楚里面到底发生了什么。
- 基于以上原因，开发了本工具。

### 工具特点
- 无需安装，网页工具，打开网页即可用。
- 强制下载现有片段，无需等待完整视频下载完成。提前下载片段，袋袋平安。
- 操作直观，精确到视频碎片的操作。

### 功能说明
- 【重新载入】刷新工具页面，重置。
- 【解析下载】输入 m3u8 文件链接后，点击下载视频。
- 【跨域复制代码】当资源出现跨域限制时，点击复制页面代码，在视频页面的控制台输入。将工具注入到视频页面中。解决跨域问题。
- 【重新下载错误片段】当部分视频片段下载失败时，点击该按钮，重新下载错误片段。
- 【强制下载现有片段】将已经下载好的视频片段强制整合成一个下载。可以提前观看已经下载的片段。该操作不会影响当前下载进程。
- 【片段Icon】对应每一个 .ts 视频片段的下载情况。「灰色」：待下载，「绿色」：下载成功，「红色」：下载失败。点击红色可重新下载对应错误片段。

### 使用说明
- 打开视频目标网页，右键「检查」，或者「开发者工具」，或者按下键盘的「F12」键
- 找到 network，输入 m3u8，过滤 m3u8 文件。
- 刷新页面，监听 m3u8 文件。
- 找到目标文件，查看文件内容，是否符合格式。
    - 碎片文件较多，一般由很多项目的才是。
- 拷贝这个 m3u8 文件的链接。
- 打开工具页面，输入链接，点击「解析下载」
- 出现片段 Icon，则证明操作成功，耐心等待视频下载
- 片段全部下载成功，将触发浏览器自动下载整合后的片段。
    - 有片段下载失败，则点击对应片段，或点击「重新下载错误片段」按钮。重新下载错误片段。
- 修改下载成功的视频文件后缀，改成「.ts」文件。
- 使用视频播放器正常播放。

### 异常情况
- 【无法下载，没有显示片段Icon】
    - 一般是由于跨域造成。
    - 点击「跨域复制代码」按钮
    - 打开视频目标网页的「开发者工具界面」，找到 console 栏
    - 粘贴刚刚复制的内容
    - 滚动页面到底部，发现工具显示在底部。然后在注入的工具中正常使用。

- 【下载后的视频资源不可看】
    - 对视频进行了加密操作。不同的视频网站有不同的算法操作。无法通用处理。
    - 一般网站不会有这种情况。爱奇艺，腾讯等大视频网站才会有该安全措施。

### 实现思路
- 【下载并解析m3u8】
    - 直接通过 ajax 的 get 请求 m3u8 文件。得到 m3u8 文件的内容字符串。
    - 需要注意的是，m3u8 文件不是 json 格式，不要将 dataType 设置为 json。
- 【队列下载ts视频片段】
    - 同样使用 ajax 的 get 请求视频碎片，一个 ajax 请求一个 ts 视频碎片，但关键点在于，下载的是视频文件，属于二进制数据，需要将 responseType 请求头设置为 「arraybuffer」
    - 使用队列下载，是因为视频碎片太多，不可能一次性请求全部。需要分批下载。
    - 同时由于浏览器同源并发限制，视频同时请求数不能过多。本工具设置为并发数 10。
- 【组合 ts 视频片段】
    - 看似很难，但其实使用 [Blob](https://developer.mozilla.org/zh-CN/docs/Web/API/Blob) 对象即可将多个 ts 文件整合成一个文件。new Blob()，传入 ts 文件数组。```const fileBlob = new Blob(fileDataList) ```
- 【自动下载】
    - 下载，当然先要获得文件链接，即刚生成的 Blob 文件链接。
    - 使用 [URL.createObjectURL](https://developer.mozilla.org/zh-CN/docs/Web/API/URL/createObjectURL)，即可得到浏览器内存中，Blob 的文件链接。```URL.createObjectURL(fileBlob)```
    - 最后，使用 a 标签的 [a.download](https://developer.mozilla.org/zh-CN/docs/Web/HTML/Element/a) 属性，将 a 标签设置为下载功能。主动调用 click 事件```a.click()```。完成文件自动下载。

### 核心代码
- [源码链接]()
- 【整合及自动下载】
```
    // 下载整合后的TS文件
    downloadFile(fileDataList, fileName, fileType) {
      this.tips = 'ts 碎片整合中，请留意浏览器下载'
      const fileBlob = new Blob(fileDataList) // 创建一个Blob对象
      const a = document.createElement('a')
      a.download = fileName + '.' + fileType
      a.href = URL.createObjectURL(fileBlob)
      a.style.display = 'none'
      document.body.appendChild(a)
      a.click()
      a.remove()
    },
```















































