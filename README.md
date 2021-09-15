# westore - 小程序 setData 解决方案

> 为简化、提速和消灭 setData 而设计

##  背景

### setData

setData 是小程序开发中使用最频繁的接口，也是最容易引发性能问题的接口。在介绍常见的错误用法前，先简单介绍一下 setData 背后的工作原理。

```js
this.setData({
  'array[0].text':'changed data'
})
```

### 工作原理

小程序的视图层目前使用 WebView 作为渲染载体，而逻辑层是由独立的 JavascriptCore 作为运行环境。在架构上，WebView 和 JavascriptCore 都是独立的模块，并不具备数据直接共享的通道。当前，视图层和逻辑层的数据传输，实际上通过两边提供的 evaluateJavascript 所实现。即用户传输的数据，需要将其转换为字符串形式传递，同时把转换后的数据内容拼接成一份 JS 脚本，再通过执行 JS 脚本的形式传递到两边独立环境。

而 evaluateJavascript 的执行会受很多方面的影响，数据到达视图层并不是实时的。

### 常见的 setData 操作错误

* 频繁的去 setData
* 每次 setData 都传递大量新数据
* 后台态页面进行 setData


### 总结下小程序的痛点

* 反直觉:使用 this.data 可以获取内部数据和属性值，但不要直接修改它们，应使用 setData 修改
* 编程体验不好: 很多场景直接赋值更加直观方便，setData 经常要在脑海里构造 path
* setData 卡卡卡慢慢慢，JsCore 和 Webview 数据对象来回传浪费计算资源和内存资源


## 解决方案

> 使用 westore 代理 setData 

1. 安装

```
npm i westore --save
```

2. 在小程序里构建 npm


3. 使用

```js
const { update } = require('westore')

...
...
  getUserInfo(e) {
    this.data.userInfo = e.detail.userInfo
    this.data.hasUserInfo = true
    update(this)
    // 等同于下面代码:
    // this.setData({
    //   userInfo: e.detail.userInfo,
    //   hasUserInfo: true
    // })
  }

```

## 特性

* 对小程序 0 入侵
* 超小的代码尺寸(包括 json diff 共100多行)
* 消除 setData， 提升编程体验，赋值 > update 的编程体验更直观和符合直觉

所以没使用 westore 的时候经常可以看到这样的代码:

![not-westore](./asset/not-westore.png)

使用完 westore 之后:

```js
this.data.a.b[1].c = 'f'
update(this)
```

## 实战

拿官方模板示例的页面作为例子:

欢迎页：

```js
// index.js
// 获取应用实例
const app = getApp()
const { update } = require('westore')

Page({
  data: {
    motto: 'Hello World',
    userInfo: {},
    hasUserInfo: false,
    canIUse: wx.canIUse('button.open-type.getUserInfo'),
    canIUseGetUserProfile: false,
    canIUseOpenData: false // 如需尝试获取用户信息可改为false
  },
  // 事件处理函数
  bindViewTap() {
    wx.navigateTo({
      url: '../logs/logs'
    })
  },
  onLoad() {
    if (wx.getUserProfile) {
      this.data.canIUseGetUserProfile = true
      update(this)
    }
  },
  getUserProfile(e) {
    // 推荐使用wx.getUserProfile获取用户信息，开发者每次通过该接口获取用户个人信息均需用户确认，开发者妥善保管用户快速填写的头像昵称，避免重复弹窗
    wx.getUserProfile({
      desc: '展示用户信息', // 声明获取用户个人信息后的用途，后续会展示在弹窗中，请谨慎填写
      success: (res) => {
        console.log(res)
        this.data.userInfo = res.userInfo
        this.data.hasUserInfo = true
        update(this)
      }
    })
  },
  getUserInfo(e) {
    // 不推荐使用getUserInfo获取用户信息，预计自2021年4月13日起，getUserInfo将不再弹出弹窗，并直接返回匿名的用户个人信息
    console.log(e)
    this.data.userInfo = res.userInfo
    this.data.hasUserInfo = true
    update(this)
  }
})

```

日志页：

```js
// logs.js
const util = require('../../utils/util.js')
const { update } = require('westore')

Page({
  data: {
    logs: []
  },
  onLoad() {
    this.data.logs = (wx.getStorageSync('logs') || []).map(log => {
      return {
        date: util.formatTime(new Date(log)),
        timeStamp: log
      }
    })
    update(this)
  }
})


```
<!-- 
### JSON Diff

先看一下我为 westore 专门定制开发的 [JSON Diff 库](https://github.com/dntzhang/westore/blob/master/packages/westore/utils/diff.js) 的能力:

``` js
diff({
    a: 1, b: 2, c: "str", d: { e: [2, { a: 4 }, 5] }, f: true, h: [1], g: { a: [1, 2], j: 111 }
}, {
    a: [], b: "aa", c: 3, d: { e: [3, { a: 3 }] }, f: false, h: [1, 2], g: { a: [1, 1, 1], i: "delete" }, k: 'del'
})
```

Diff 的结果是:

``` js
{ "a": 1, "b": 2, "c": "str", "d.e[0]": 2, "d.e[1].a": 4, "d.e[2]": 5, "f": true, "h": [1], "g.a": [1, 2], "g.j": 111, "g.i": null, "k": null }
```

![diff](./asset/diff.jpg)

Diff 原理:

* 同步所有 key 到当前 store.data
* 携带 path 和 result 递归遍历对比所有 key value

``` js
export default function diff(current, pre) {
    const result = {}
    syncKeys(current, pre)
    _diff(current, pre, '', result)
    return result
}
```

同步上一轮 state.data 的 key 主要是为了检测 array 中删除的元素或者 obj 中删除的 key。
 -->

## 其他

1. setData调用频率
setData接口的调用涉及逻辑层与渲染层间的线程通信，通信过于频繁可能导致处理队列阻塞，界面渲染不及时而导致卡顿，应避免无用的频繁调用。

> 每秒调用setData的次数不超过 20 次

2. setData数据大小
由于小程序运行逻辑线程与渲染线程之上，setData的调用会把数据从逻辑层传到渲染层，数据太大会增加通信时间。

> setData的数据在JSON.stringify后不超过 256KB

3. 避免setData数据冗余
setData操作会引起框架处理一些渲染界面相关的工作，一个未绑定的变量意味着与界面渲染无关，传入setData会造成不必要的性能消耗。

> setData传入的所有数据都在模板渲染中有相关依赖

4. Page 的 setData 函数用于将数据从逻辑层发送到视图层（异步），同时改变对应的 this.data 的值（同步）。

5. Component 的 setData 是同步的操作


## License

MIT [@dntzhang](https://github.com/dntzhang)