### React学习篇-JSX-手写一个JSX的插件


学习和阅读 vue 源码有段时间了，最近在尝试去学习 react，由于眼前项目使用不上 react，并不想一股脑的学习它的 API（长时间不用还是会忘），所以此次的学习过程打算换种方式，对于 react 涉及到的每个点尝试逐个深入，了解其解析过程及整个框架的思路。

对于每个点的学习和深入，将以文章的形式产出，主要是对于学习的内容的记录（所以看来内容有点多），方便自己以后是用时查阅和回顾。

### 从demo开始

在此之前，曾多次的在 react 入门的边缘来回试探，每次都止于写一个简单的 demo，我相信下面这个大家肯定很熟悉，本文也是从这里开始的。

```
npx create-react-app my-app
cd my-app
npm install
npm start
```

然后应该就能跑起来（环境和安装没有问题的话），简化下代码，然后面对下面的这个代码陷入了思考，虽然以前见过也写过很多次了。

代码如下：
```js
import React from 'react';

function App() {
  return (
    <div>
        <h1>good good study, day day up</h1>
    </div>
  );
}

export default App;
```

`APP` 返回的乍一看很像 html，当然相信很多人都知道这个是 JSX 的语法。那么问题来了：

JSX 语法写的模版，如何生成真实的 dom？

类比我们先看看`Vue`中`template` - `ast` - `code` -`vnode` - `dom`的实现。

### 简单看下Vue 的parse过程

template 转换成 ast 及 ast 转换成 code 的过程推荐几篇文章:

- [Vue 源码编译思想之parse那些事](https://juejin.im/post/5cfc6ad9e51d4558936aa04d)
- [Vue源码解析之parse](https://juejin.im/post/5d09a4fef265da1b6b1cd96b#heading-0)

以下用一个简单的例子来简单说明下 parse 的过程
#### template 如下：
```html
<template>
    <div>
        <h1>good good study, day day up</h1>
    </div>
</template>
```


#### 对应生成的 ast 如下：

```js
// 简化版，主要是看下结构
{
    //...
    parent: undefined,
    children: [
        {
            parent: {
                //...
                tag: "div",
                type: 1
            },
            children: [
                // ...
                text: "good good study, day day up",
                type: 3
            ],
            tag: "h1",
            type: 1
        }
    ],
    tag: "div",
    type: 1
}
```

对应生成的 `code` 如下：

```js
with(this){
    return _c(
        'div',
        [
            _c(
                'h1',
                [_v("good good study, day day up")]
            )
        ]
    )
}
```
最终得到的结果就是这样的渲染函数。

我们再看看react的实现。首先直接看`npm star`后的`main.chunk.js`文件，可以看到如下的代码（简化版）：

```js
function App() {
    return createElement(
        "div", 
        {
            __source: {
                fileName: _jsxFileName,
                lineNumber: 5
            },
            __self: this
        }, 
        createElement(
            "h1", 
            {
                __source: {
                    fileName: _jsxFileName,
                    lineNumber: 6
                },
                __self: this
            }, 
            "good good study, day day up"
        )
    );
}
```

对比 `Vue` 生成的 code，会发现很像，所以这里可以先总结一下：

> react 也是通过一层转换，把我们写的 JSX 模版，转换成对应的函数。

所以这就算完了？来，接着来，JSX 是如何转换的？

### JSX 的转换过程

了解 Vue `parse`过程的就知道，转换是发生在编译的阶段：在首次`$mount`的时候会执行`compileToFunctions`(其中主要就是模版到渲染函数的过程)。

那 React 呢，尝试去看了 `React` 和 `ReactDOM` 的源码，根本找不到任何转换的代码。而且大家也看到了`main.chunk.js`的代码，我们写的 JSX 已经转换成对应的函数了。所以再此之前，已经完成了转换。

好了不卖关子了，这里用的是 `babel` 解析器（[什么是Babel，Babel能做什么](https://www.jianshu.com/p/3f72085896d9)），我们首先找到工程中配置的地方。

#### 寻找 babel 配置的入口
由于本人对于工程配置及工程化不是很了解，所以我这里也是找了很久，要想找到 babel 配置的入口，需先执行（最好找个demo工程执行，该命令不可逆）
```
yarn eject
```

找到 `/config/webpack.config.js`, 相关代码如下：
```js
module: {
    {
        test: /\.(js|mjs|jsx|ts|tsx)$/,
        include: paths.appSrc,
        loader: require.resolve('babel-loader'),
        options: {
        customize: require.resolve(
            'babel-preset-react-app/webpack-overrides'
        ),
        
        plugins: [
            [
            require.resolve('babel-plugin-named-asset-import'),
            {
                loaderMap: {
                svg: {
                    ReactComponent: '@svgr/webpack?-svgo,+ref![path]',
                },
                },
            },
            ],
        ],
        cacheDirectory: true,
        cacheCompression: isEnvProduction,
        compact: isEnvProduction,
        },
    },
    {
        test: /\.(js|mjs)$/,
        exclude: /@babel(?:\/|\\{1,2})runtime/,
        loader: require.resolve('babel-loader'),
        options: {
        babelrc: false,
        configFile: false,
        compact: false,
        presets: [
            [
            require.resolve('babel-preset-react-app/dependencies'),
            { helpers: true },
            ],
        ],
        cacheDirectory: true,
        cacheCompression: isEnvProduction,
        sourceMaps: false,
    },
}
```

看到这里相信就能知道，这里其实就是配置了 `loader`，试了看各个解析器的源码，但是仍然困难重重（各种引用），这里也是换了种方式来学习解析的过程。

尝试手写一个 JSX 的插件。

### 手写 JSX 的插件

这里大家网上搜应该能搜出一堆关于babel 插件的代码，我这里也是找到一个基础的例子。

#### console的插件的例子

以下是一个将`log`处理成`console.log`的插件的代码：
```
const babel = require('@babel/core')
const t = require('babel-types')

const code = `
    const a = 3 * 103.5 * 0.8;
    log(a);
    const b = a + 105 - 12;
    log(b);
`

const visitor = {
    CallExpression(path) {
        // 这里判断一下如果不是log的函数执行语句则不处理
        if (path.node.callee.name !== 'log') return
        // t.CallExpression 和 t.MemberExpression分别代表生成对于type的节点，path.replaceWith表示要去替换节点,这里我们只改变CallExpression第一个参数的值，第二个参数则用它自己原来的内容，即本来有的参数
        path.replaceWith(t.CallExpression(
            t.MemberExpression(t.identifier('console'), t.identifier('log')),
            path.node.arguments
        ))
    }
}

const result = babel.transform(code, {
    plugins: [{
        visitor: visitor
    }]
})

console.log(result.code)
```

处理结果：
```
const a = 3 * 103.5 * 0.8;
console.log(a);
const b = a + 105 - 12;
console.log(b);
```

看了代码后应该差不多能了解插件的编写过程，大致如下：code 首先会解析成 AST，然后会遍历整个 AST 树，每个节点都有其特定的属性，插件的vistor对象的处理函数会在解析的过程中被调用，插件要做的事情就是在合适的地方（这里是`CallExpression`）,符合条件的情况下（这里是 `path.node.callee.name === 'log'`）,对解析结果进行更改。知道原理以后，尝试着写 JSX 解析的插件。

#### 照葫芦画瓢：
```
const code = `
    var html = <div>
        <h1>good good study, day day up</h1>
    </div>
`

const visitor = {
   
}

const result = babel.transform(code, {
    plugins: [
        {
            visitor: visitor
        }
    ]
})

console.log(result.code)
```

大致的结构就是这样，期望达到的目标code对应的输出如下：
```
var html = React.createElement(
    "div", 
    null, 
    React.createElement("h1", null, "good good study, day day up")
)
```

以上代码执行后，会报错，因为并不是js的标准语法，无法正常解析，所以这里首先需要引入一个插件 `plugin-syntax-jsx`，让解析器其能识别该种语法。

引入插件，修改的代码如下：
```js
babel.transform(code, {
    plugins: [
        '@babel/plugin-syntax-jsx',
        {
            visitor: visitor
        }
    ]
})
```

执行的结果为：

```
var html = <div>
    <h1>good good study, day day up</h1>
</div>;
```

这里能看到我们能正常识别 JSX 模版，只是输出并不是我们需要的，我们需要把它转换成我们的函数。接下来的一步就是需要找到合适的时机。

#### 寻找时机

这里我们只是知道我们能正常识别了，但是在解析的过程中，其对应的 AST 具体长什么样子呢？

这里也是推荐一个网站，[https://astexplorer.net/](https://astexplorer.net/)

![AST的结构图](https://user-gold-cdn.xitu.io/2019/6/22/16b7e40c639221f9?w=2876&h=1782&f=png&s=382692)

这里就能看到整个 AST 树的结构（这里还没去看解析成 AST 生成的过程，目测和 Vue 中 parseHTML 的过程原理一样，这里后续会花点时间看下 babal 生成 AST 的过程），应该很快就能找到我们想要的关键信息-`JSXElement`，对照以上的 AST 和关键信息，就当前这个例子，我们就思考下‘合适的时机‘-JSXElement的变量赋值：

- VariableDeclarator
- `init.type === 'JSXElement'`

#### 加入‘时机’代码
加入‘时机’后代码如下：
```js
const babel = require('@babel/core')

const code = `
    var html = <div>
        <h1>good good study, day day up</h1>
    </div>
`

const visitor = {
    VariableDeclarator(path) {
        if (path.node.init.type === 'JSXElement'){
            console.log('start')
            // deal 
        }
    }
}

const result = babel.transform(code, {
    plugins: [
        '@babel/plugin-syntax-jsx',
        {
            visitor: visitor
        }
    ]
})

console.log(result.code)
```

得到的结果如下：
```
start
var html = <div>
        <h1>good good study, day day up</h1>
    </div>;
```

当然这里只是输入标签的信息，其中还有很多其他的节点信息，其他的信息那么也就是 JSX 的语法规则了，如循环、class、条件语句、逻辑代码等语法规则了。本文只做简单的实现。接下来要做的就是要整合节点的信息，生成对应的函数代码。

#### 生成代码

... 未完待续

(这里涉及到`babel-types`的使用，由于对此块不是很熟悉，文章先进行到这里，后续写好会更新上来)

---


那了解了JSX的解析过程后，我们思考下，这个与vue的`parse`的过程差别在哪？

- vue 是编译阶段生成对应的渲染函数，react 是babel解析阶段就生成了对应的函数
- 看过 vue parse阶段源码的同学应该知道，vue 做了很多处理浏览器‘怪异’行为的操作（为了保持和浏览器行为的一致性），如：标签换行会有空格符、`canBeLeftOpenTag`标签如：p，会补全关闭标签等，也就是大家可以像写普通的html来写`template`。而 react 的 JSX 就有很多的语法规则，如`class`必须写`className`、标签之前的换行后的空格会被忽略等等（仍在学习JSX语法中，后续会继续补充完善这块的区别）。

就第二点区别，可以看出来，如果是原有的html项目，想要迁移成 vue 或 react，迁移成 Vue 的成本会小很多，Vue 不仅在写法上，还有对于浏览器特殊行为的处理上，都保持了和 html 规范的统一。若要迁移成 react ,可能改造成本就会比较大。

以上只是react初学者的主观的看法，更多的特性和优劣需要深入学习后才能了解。




