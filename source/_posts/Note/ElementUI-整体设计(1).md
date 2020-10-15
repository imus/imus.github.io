---
title: ElementUI-整体设计(1)
categories:
  - Note
tags:
  - element-ui
  - web
date: 2020-10-15 15:26:10
---

如何学习ElementUI源码：https://www.zhihu.com/question/60706223 （知乎）

------

### 整体设计2.13.2

#### element-ui的开发考虑的需求如下：

1. 功能： 组件，自定义主题，国际化
2. 文档：官方文档和demo，低成本，多语。
3. 使用方式： cdn、npm，支持按需引入。
4. 工程化：开发、测试、构建、部署，CI。

#### 需求实现：

##### 	组件

​		组件丰富，分为6大类： 基础、表单类、数据类、提示类、导航类和其他组件。

​		目录：`/packages`

##### 	自定义主题

​		支持在线主题编辑：

​			思路：主题通过配置维护，修改后把新的配置交给server，server根据配置返回新的CSS（变量替换，编译），覆						盖默认样式。？？？<u>具体如何实现，探索下</u>

​			在主题编辑页的网络面板中可以看到获取配置getVariable和更新配置updateVariable的请求，代码目录：

```
	/examples/components/theme/loader/index.vue // 监听事件，onAction更新主题，拿到配置applyStyle覆盖默认样式
	getVarible, 主题配置右侧面板数据
	localStorage也会保存一份用户修改的配置，可能会有多个主题
	\examples\components\theme-configurator\index.vue // onAction, 触发ACTION_APPLY_THEME事件
```

​			

​		目录：

```
	/packages/theme-chalk //组件样式、公共样式
	/packages/theme-chalk/src/common/var.scss // 组件中引入的样式变量的定义
```

##### 	国际化

​		element-ui未使用vue-i18n。

​		国际化方案会用到语言包，目录：`element\src\locale\lang`

​		pagination组件中Jumper组件中t函数：

```
import Locale from 'element-ui/src/mixins/locale'; // t函数定义位置，最终位置\src\locale\index.js
mixin: [Locale],
 render(h) {
        return (
          <span class="el-pagination__jump">
            { this.t('el.pagination.goto') }
          </span>
        );
      }
  语言包中配置：
    pagination: {
      goto: '前往',
      pagesize: '条/页',
      total: '共 {total} 条',
      pageClassifier: '页'
    },
```

​	？？？<u>字符串格式化函数format

```
/* 使用语言包时，需要注册 */
import lang from 'element-ui/lib/locale/lang/en'
import locale from 'element-ui/lib/locale'

// 设置语言
locale.use(lang)
```

##### 	文档（值得借鉴）

​	目录： `element\examples\docs`

​	格式.md。

​	route.config.js // 路由配置，加载md文档

​		registerRoute函数（重点）

​		addRoute中的loadDocs(lang, page.path) // 获取对应.md组件

​	webpack.demo.js中module配置解析.md格式的文档：

```
 {
        test: /\.md$/,
        use: [
          {
            loader: 'vue-loader',
            options: {
              compilerOptions: {
                preserveWhitespace: false
              }
            }
          },
          // loader是从后往前执行的，使用md-loader处理.md文档转化为.vue格式字符串
          {
            loader: path.resolve(__dirname, './md-loader/index.js') // build/md-loader/index.js
          }
        ]
      }
```

​	md-loader工作： ？？？<u>md-loader处理过程</u>

​		md.render(source)` 对 `md` 文档解析，提取文档中 `:::demo {content} :::` 内容，分别生成一些 Vue 的模板字符串，然后再从这个模板字符串中循环查找 `<!--element-demo:` 和 `:element-demo-->` 包裹的内容，从中提取模板字符串到 `output` 中，提取 script 到 `componenetsString` 中，然后构造 `pageScript

##### 	使用方式

​	CDN： 打包一份CSS和JS,全量引入，体积大

​	npm 全量：引入js,css

​	npm按需引入: 

```
import {Button} from 'element-ui'
Vue.component(Button.name, Button)
```

​		为什么它可以按需引入？需要借助babel-plugin-component, 然后修改.babelrc

```
{
  "presets": [["es2015", { "modules": false }]],
  "plugins": [
    [
      "component",
      {
        "libraryName": "element-ui",
        "styleLibraryName": "theme-chalk"
      }
    ]
  ]
}
```

​		转化为：

```
var button = require('element-ui/lib/button')
require('element-ui/lib/theme-chalk/button.css')
```

​	产生组件依赖并且同时按需引入的时候，代码会有冗余。于是，在build/config配置webpack的externals选项，防止将这些 import 的包打包到 bundle 中，并在运行时再去从外部获取这些扩展依赖。(js)。

​	lib/table.js对 `CheckBox` 组件的依赖引入如下：`570 module.exports = require("element-ui/lib/checkbox");`

​	但是， css样式并未做冗余处理。

​	要解决按需引入的 JS 和 CSS 的冗余问题并非难事，可以用后编译的思想，即依赖包提供源码，而编译交给应用处理，这样不仅不会有组件冗余代码，甚至连编译的冗余代码都不会有。参考滴滴cube-ui:（webpack 应用编译优化之路） https://juejin.im/post/6844903502586593288

##### 	工程化

​	ESLint, webpack HotReload热重载, karma测试框架，Travis CI集成。

#### 构建

​	npm run dist， 输出lib目录

​	？？？ <u>dist整体流体需要详细分析下</u>

​	`build:file` 运行 `build` 目录下几个命令，包括对 `icon`、`entry`、`i18n`、`version` 等初始化（根据一些规则做文件的 IO）

​	pub: 部署，通过运行一系列的 bash 脚本，实现了代码的提交、合并、版本管理、npm 发布、官网发布等，让整个发布流程自动化完成。

#### 目录结构

​	build // 构建

​	lib // dist构建输出的目录

​	examples // demo源码，包括在线定制主题前端交互部分

​		entry.js // demo入口

​	packages  // 组件源码

​		theme-chalk // 组件样式、公共样式（可独立发布）

​			src/common/var.scss // 变量定义： 组件样式中的颜色、字体、线条等通过变量方式引入

​	src // 组件依赖的一些公共模块

​		local/lang // 国际化方案组件库语言包， 关键t函数

​	test // 测试

​	types // 组件类型ts

​	FAQ.md

​	···