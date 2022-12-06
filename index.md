## 性能监控

网站性能监控的几个指标：
  FP: first paint; 第一个像素渲染的时间
  FCP: first contentful paint; 第一个字节渲染的时间
  DCL: dom content loaded; dom渲染完成
  LCP: largest contentful paint; 最大体积内容渲染时间

onload执行时期的性能监控

window.performance.timing是浏览器提供的能够精确衡量网站性能的数据。

几个流程
performance before unload:
  timing：navigationStart
    -- navigationStart是在同一个浏览器上下文中，前一个网页（于当前网页不同域）unload的时间戳，于fetchStart值相等。

redirect：
  timing：redirectStart；redirectEnd
    -- redirectStart：第一个http重定向发生的时间。有跳转且是同域名内的重定向才算，否则为0
    -- redirectEnd：最后一个http重定向完成时的时间。有跳转且同域名内的重定向才算，否则为0

app Cache:
  timing: fetchStart
    -- fetchStart：浏览器准备好使用http请求抓取文档的时间，发生在检查本地缓存之前。

DNS:
  timing: domainLookupStart; domainLookupEnd
    -- start：DNS域名查询开始时间，若使用了本地缓存（无DNS查询）或持久连接，则为0
    -- end：DNS域名查询完成时间，若使用了本地缓存（无DNS查询）或持久连接，则为0

TCP:
  timing: connectStart; sucureConnectStart; connectEnd
    -- connectStart：tcp开始建立连接的时间，若持久连接，与fetchStart相同。
    -- sucureConnectStart：https建立连接时间，如不是安全连接，则为0
    -- connectEnd: tcp完成建立连接的时间，即握手结束

request:
  timing: requestStart
    -- http请求 读取真实文档 开始的时间（完成建立连接），包含从本地读取缓存

response:
  timing: responseStart; responseEnd
    -- responseStart：http开始接收响应的时间（第一个字节）包含本地缓存读取
    -- responseEnd：http全部接收完的时间（最后一个字节）包含本地缓存读取

processing:
  timing: domLoading; domInteractive; domContentLoadedEventStart;domContentLoadedEventEnd;domComplete
    -- domLoading：开始解析渲染DOM树的时间，此时document.readState => loading,并且抛出readyStateChange相关事件。
    -- domInteractive：完成dom解析的时间，此时document.readyState => interactive, 抛出readyStateChange事件，还没有加载网页资源。
    -- domContentLoadedEventStart: DOM解析完成，网页内资源加载开始的时间。
    -- domContentLoadedEventEnd: DOM解析完成后，网页内资源 加载完成的时间。
    -- domComplete: DOM树解析完成，且资源也准备就绪，document.readyState => complete,抛出readyStateChange事件。 
load:
  timing: loadEventStart; loadEventEnd
    -- loadEventStart：load事件发送给文档，也就是load回调函数开始执行的时间
    -- loadEventEnd：load事件回调结束的时间

unload:
  timing: unloadEventStart; unloadEventEnd
    -- unloadEventStart：前一个网页（与当前网页同域）unload的时间，如果没有前一个网页，或者前一个网页不同域，则值为0
    -- unloadEventEnd：返回前一个网页unload时间回调函数执行完毕的时间

### 使用performance.timing信息简单计算出网页性能数据
  const timing = window.performance.timing
  const times = {}
  // 页面加载完成时间。几乎代表了用户等待页面可用的时间
  times.loadPage = timing.loadEventEnd - navigationStart

  // 解析dom树结构的时间。衡量dom嵌套是不是太多，时间过长
  times.domReady = timing.domComplete - timing.domLoading

  // 重定向时间 拒绝重定向，比如 http://xx.com/ 就不该写成http://xx.com
  time.redirect = timing.redirectEnd - timing.redirectStart

  // dns解析时间 页面内是不是用了太多不同域名导致
  time.dns = timing.domainLookupEnd - timing.domainLookupStart

  // 用户获取第一个字节的时间 理解为用户拿到资源的占用的时间，CND分发，CPU运算
  time.ttfb = timing.responseStart - timing.navigationStart

  // 内容加载完成时间 js/css等有没有压缩
  time.request = timing.responseEnd - timing.responseStart

  // onload回调执行的时间，是不是onload里面的东西太多了。考虑延迟加载，按需加载
  time.loadEvent = timing.loadEventEnd - timing.loadEventStart

  // dns缓存时间
  time.dnsCache = timing.domianLookupEnd - timing.fetchStart

  // 卸载页面时间
  time.unload = timing.unloadEventEnd - timing.unloadEventStart

  // tcp握手时间
  time.tcp = timing.connectEnd - timing.connectStart


### performance.getEntries()获取所有资源请求时间数据

 window.performance.getEntries() = [PerformanceNavigationTiming, PerformancePaintTiming, PerformanceMark]

 PerformanceNavigationTiming和PerformancePaintTiming与上述属性不同之处在少了dom相关属性；多了几个属性
 name: 资源名称，资源的绝对路径
 entryType: 'resource','mark','paint'
 initiatorType: 'link', 'script', 'redirect'
 duration: 加载时间

### performance.now() 精确计算程序执行时间
  performance.now() 是从performance.timing.navigationStart开始算，不受系统时间影响，更精确

### performance.mark(name) 也可以精确计算

  window.performance.mark(name1)
  window.performance.mark(name2)
  { duration, entryType: mark, name, startTime }
  计算时间
  window.performance.measure(name, markName1, markName2)

  清除标记：
  window.performance.clearMarks(name) name不传则清除所有
  window.performance.clearMeasures(name) name不传则清除所有

### 监控performance，使用观察者模式取实时的获取数据值
  const cb = function(list, observer) {
    list.getEntries().forEach(l => {
      // PerformanceSomething
    })
  }

  const observer = new PerformanceObserver(cb)

  observer.observe({
    entryTypes: ['resource', 'paint', 'navigation']
  })