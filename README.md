# mp-router

mp-router是**小程序路由**的封装，用于页面跳转。

## 下载与安装

[点击这里](https://github.com/wxlite-plus/mp-router/releases)下载`mp-router`的源码，将解压后的文件夹拷贝到小程序项目中的`utils`目录下，之后我们在项目根目录下创建文件夹`router`，新建文件`router/index.js`，引用`mp-router`并初始化：

```javascript
const req = require('../utils/mp-router/index.js');

router.init({
  // ...
});

module.exports = router;
```

init方法接受参数：

* **routes：**即路由配置，我们在接下来的内容会介绍。

如果你想要快速启动模板：[quick-start](https://github.com/jack-Lo/mp-router-quick-start)。

## 使用

页面的跳转存在哪些问题呢？

1. 与接口的调用一样面临url的管理问题；
2. 参数类型单一，只支持string。

第一个问题很好解决，我们做一个集中管理。
第二个问题的情况是，当我们传递的参数argument不是`string`，而是`number`或者`boolean`时，也只能在下个页面得到一个`argument.toString()`值：

```javascript
// pages/index/index.js
wx.navigateTo({
  url: '/pages/a/index?a=true',
});

// pages/a/index.js
Page({
  onLoad(options) {
    console.log(options.a); // => "true"
    console.log(typeof options.a); // => "string"
    console.log(options.a === true); // => false
  },
});
```

上面这种情况想必很多人都遇到过，而且感到很抓狂，本来就想传递一个boolean，结果不管传什么都会变成string。

我们的解决方案是：利用`JSON.stringify+encodeURIComponent`和`decodeURIComponent+JSON.parse`的方式让参数保真。

顺手也替换掉那不好记的`navigate api`，于是就出现了如下方式：

```javascript
// pages/pageA/index.js
const { router } = require('../../router/index.js');

Page({
  onReady() {
    router.push({
      name: 'a',
      data: {
        id: '123',
        type: 1,
      },
    });
  },
});
```

在下一个页面接收上一个页面传递的参数，它就存放在`options`中，我们需要使用`router.extract`方法将其提炼出来：

```javascript
// pages/index/index.js
Page({
  onLoad(options) {
    const data = router.extract(options);
    console.log(this.data); // { id: '123', type: 1 }
  },
});
```

当然，上面的`name: 'a'`肯定也是事先配置好的，要不然鬼知道a到底跳转到哪里。

我们在`router`目录下新建一个文件`routes.js`，并作如下定义：

```javascript
module.exports = {
  // 主页
  home: {
    type: 'tab',
    path: '/pages/index/index',
  },
  a: {
    path: '/pages/a/index',
  },
};
```

然后在`router/index.js`中引用：

```javascript
const router = require('../utils/mp-router/index.js');
const routes = require('./routes.js');

router.init({
  routes,
});

module.exports = router;
```

留意上面的`home`页面配置，`home`其实就是`/pages/index/index`的一个别名，同时因为它是一个**tab**页面，所以我们也顺便指定了`type: 'tab'`，默认是`type: 'page'`。

除了支持别名之外，name也支持直接寻址，比如跳转home还可以写成这样：

```javascript
router.push({
  name: 'index',  // => /pages/index/index
});

router.push({
  name: 'userCenter',  // => /pages/user_center/index
});

router.push({
  name: 'userCenter.phone',  // => /pages/user_center/phone/index
});

router.push({
  name: 'test.debug',  // => /pages/test/debug/index
});
```

> 注意，为了方便维护，我们规定了每个页面都必须存放在一个特定的文件夹，一个文件夹的当前路径下只能存在一个index页面，比如`pages/index`下面会存放`pages/index/index.js`、`pages/index/index.wxml`、`pages/index/index.wxss`、`pages/index/index.json`，这时候你就不能继续在这个文件夹根路径存放另外一个页面，而必须是新建一个文件夹来存放，比如`pages/index/pageB/index.js`、`pages/index/pageB/index.wxml`、`pages/index/pageB/index.wxss`、`pages/index/pageB/index.json`。

router支持微信路由的所有方法，映射关系如下：

```javascript
router.push => wx.navigateTo
router.replace => wx.redirectTo
router.pop => wx.navigateBack
router.relaunch => wx.reLaunch
```

可能你会发现这里少了一个`wx.switchTab`，这不是遗漏，而是被集成到了`router.push`当中去了，因为我们认为，跳转一个页面到底是`page`的方式还是`tab`的方式这类事情，根本与业务无关，它应该被透明化。

你可能会记得上面我们的`home`在路由配置的时候就已经指定了`type: 'tab'`的属性，这样一来我们便可以尽管调用`router.push({name: 'home'})`，至于具体是`wx.navigateTo`还是`wx.switchTab`，程序会自动帮我们处理的。

这里还有一个好处就是，当一个页面需要从`tab`转变为`page`的时候，我们只需要改一下`routes`的定义就可以了，完全不需要去一个个修改业务中的代码。
