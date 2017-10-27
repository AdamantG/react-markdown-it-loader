## 背景
由于业务需要展示一份直观的React文档，不但要能处理Markdown文件，还需要能展示源码、运行源码、展示示例和API。
## 调研
通过一系列尝试，发现目前的loader并不完善，无法完全满足业务需求。

* **markdown-it、marked等**：能完整解析Markdown，但无法运行React代码。
* **markdown-it-react-loader**：对Markdown解析不完整，基本table都不支持。
* **react-markdown-loader**：对Markdown解析不完整，无法友好展示源码；仅能渲染html，无法运行jsx。
* **react-for-markdown-loader**：对Markdown解析不完整，且不支持运行代码，仅表示在react中使用。

## loader 的编写
### 1. 如何编写一个 loader

#### 官网描述如下：
> loader 是导出为 function 的 node 模块。
> 当资源应该由此 loader 转换时，调用此函数。
> 在简单的情况下，当只有一个 loader 应用于资源时，调用 loader 有一个参数：作为字符串的资源文件的内容。
> 在 loader 中，可以通过 this 上下文访问 [[loader API | loaders]]。

#### 示例
```
// 定义 loader
module.exports = function(source) {
  return source;
};
```
#### 更多描述：
> 一个同步 loader 可以通过 return 来返回这个值。在其他情况下，loader 可以通过 this.callback(err, values...) 函数返回任意数量的值。错误会被传到 this.callback 函数或者在同步 loader 中抛出。
> 这个 loader 的 callback 应该回传一个或者两个值。第一个值的结果是 string 或 buffer 类型的 JavaScript 代码。第二个可选的值是 JavaScript 对象的 SourceMap。

#### 示例
```
// 支持 SourceMap 的 loader
module.exports = function(source, map) {
  this.callback(null, source, map);
};
```
#### 结论
简单来说，就是我们通过参数source获取资源文件的内容，随便玩耍，处理成我们需要的东西后，抛出即可。

### 2. 解析Markdown
这里解析Markdown使用的是[markdown-it](https://github.com/markdown-it/markdown-it),因为它“[*100% CommonMark support*](https://markdown-it.github.io/)”。
#### 安装
`npm install markdown-it --save`

#### [使用](https://markdown-it.github.io/markdown-it/)
引入`markdown-it`后初始化一个实例`new MarkdownIt([presetName][, options])
`。
```
  html:         false,        // Enable HTML tags in source
  xhtmlOut:     false,        // Use '/' to close single tags (<br />).
                              // This is only for full CommonMark compatibility.
  breaks:       false,        // Convert '\n' in paragraphs into <br>
  langPrefix:   'language-',  // CSS language prefix for fenced blocks. Can be
                              // useful for external highlighters.
  linkify:      false,        // Autoconvert URL-like text to links

  // Enable some language-neutral replacement + quotes beautification
  typographer:  false,

  // Double + single quotes replacement pairs, when typographer enabled,
  // and smartquotes on. Could be either a String or an Array.
  //
  // For example, you can use '«»„“' for Russian, '„“‚‘' for German,
  // and ['«\xA0', '\xA0»', '‹\xA0', '\xA0›'] for French (including nbsp).
  quotes: '“”‘’',

  // Highlighter function. Should return escaped HTML,
  // or '' if the source string is not changed and should be escaped externaly.
  // If result starts with <pre... internal wrapper is skipped.
  highlight: function (/*str, lang*/) { return ''; }
```

presetName是`markdown-it`提供的一种快速配置，它支持三种模式：

* [commonmark](https://github.com/markdown-it/markdown-it/blob/master/lib/presets/commonmark.js)：按照严格[CommonMark](http://commonmark.org/)模式解析。
* [default](https://github.com/markdown-it/markdown-it/blob/master/lib/presets/default.js)：允许所有规则，但是任然没有html、typographer、autolinker的支持。
* [zero](https://github.com/markdown-it/markdown-it/blob/master/lib/presets/zero.js)：所有规则禁用，当你仅使用`bold`和`italic`时可以通过`.enable()`快速启动。

具体options配置及含义如下：

* **html**- `false`.是否允许源码中含有HTML标签。这个配置需要小心，以防止XSS攻击。最好的做法是通过三方插件来实现允许HTML。
* **xhtmlOut**- `false`.设置`true`后通过'/'来关闭空标签。仅在兼容CommonMark时需要。
* **breaks**- `false`.设置`true`后会将`\n`转化为`<br>`。
* **langPrefix**- `language-`.CSS 前缀。
* **linkify**- `false`.设置`true`后自动转换链接。
* **typographer**- `false`.设置`true`后允许一些语言替换（比如单引号、双引号同时使用会变成一对单／双）和引用美化。
* **quotes**- `“”‘’`.设置`true`后会转化为不同语言下的引号。
* **highlight**- `null`.对代码块的高亮函数。

根据需求，首先不允许html，因此，我们不使用`commonmark`，此外，我们不单单仅使用`bold`和`italic`，因此，我们选择`default`配置。
```
// default mode
let md = require('markdown-it')();
```
#### 插件加载
我们还可能使用一些相应插件，其使用方式如下：
```
let md = require('markdown-it')()
            .use(plugin1)
            .use(plugin2, opts, ...)
            .use(plugin3);
```
我们使用的是[markdown-it-anchor](https://github.com/valeriangalliat/markdown-it-anchor)，它是头部的锚。
通过`npm install markdown-it-anchor --save`安装。
我们首先需要的是`permalink`，然后是`slugify`,
函数使用[`transliteration`](https://github.com/andyhu/transliteration)提供的`slugify`(`npm install transliteration --save
`)。锚前面的class或者symbol等，可根据自己的需求配置。
根据其API文档和我们的需求，使用配置如下：
```
const anchor = require('markdown-it-anchor');
const slugify = require('transliteration').slugify;

let md = require('markdown-it').use(anchor, {
    slugify: slugify,
    permalink: true
})
```
#### 代码高亮
首先需要通过`npm install highlight --save
`来安装[highlight](https://github.com/isagalaev/highlight.js)。
可以简单使用：
```
var hljs = require('highlight.js'); // https://highlightjs.org/

// Actual default values
var md = require('markdown-it')({
  highlight: function (str, lang) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return hljs.highlight(lang, str).value;
      } catch (__) {}
    }

    return ''; // use external default escaping
  }
});
```
也可以包装起来：
```
var hljs = require('highlight.js'); // https://highlightjs.org/

// Actual default values
var md = require('markdown-it')({
  highlight: function (str, lang) {
    if (lang && hljs.getLanguage(lang)) {
      try {
        return '<pre class="hljs"><code>' +
               hljs.highlight(lang, str, true).value +
               '</code></pre>';
      } catch (__) {}
    }

    return '<pre class="hljs"><code>' + md.utils.escapeHtml(str) + '</code></pre>';
  }
});
```

我们处理以及含义如下：
```
highlight(content, languageHint){
    let highlightedContent;

    highlight.configure({
        useBR: true,
        tabReplace: '    '//替换TAB为你想要的任意字符,便于排版
    });
    // 使用highlight的getLanguage获取语言
    if (languageHint && highlight.getLanguage(languageHint)) {
        try {
            // 高亮显示
            highlightedContent = highlight.highlight(languageHint, content).value;
        } catch (err) {
        }
    }
    // 当无法检测语言时，使用自动模式
    if (!highlightedContent) {
        try {
            highlightedContent = highlight.highlightAuto(content).value;
        } catch (err) {
        }
    }

    // 把代码中的{}转
    highlightedContent = highlightedContent.replace(/[\{\}]/g, (match) = > `{'${match}'}`
)
    ;

    // 加上 hljs 根据code标签上的class识别语言
    highlightedContent = highlightedContent.replace('<code class="', '<code class="hljs ').replace('<code>', '<code class="hljs">')

    return highlight.fixMarkup(highlightedContent);
}
```
**至此，解析Markdown部分基本完成。**

### 3. 执行代码
我们需要将中的jsx语言代码块执行并渲染，需要用到`markdown-it-container`。
#### 安装
`npm install markdown-it-container --save
`
#### [使用](https://github.com/markdown-it/markdown-it-container)
```
let md = require('markdown-it')()
            .use(require('markdown-it-container'), name [, options]);
```
参数如下：

* **name**- 包裹的名称
* options：
	* **validate**- 验证函数
	* **render**- 渲染函数
	* **marker**- 分隔符（`:`）的使用

根据使用方法，我们代码如下：
```
//引用
const mdContainer = require('markdown-it-container');
let moduleJS = [];
let flag = '';

md.use(mdContainer, 'demo', {
    validate: function (params) {
        return params.trim().match(/^demo\s*(.*)$/);
    },
    render: function (tokens, idx) {
        // container 从开头到结尾把之间的token跑一遍，其中idx定位到具体的位置

        // 获取描述
        const m = tokens[idx].info.trim().match(/^demo\s*(.*)$/);

        // 有此标记代表 ::: 开始
        if (tokens[idx].nesting === 1) {
            flag = idx;

            let jsx = '', i = 1;

            // 从 ::: 下一个token开始
            let token = tokens[idx + i];

            // 如果没有到结尾
            while (token.markup !== ':::') {
                // 只认```，其他忽略
                if (token.markup === '```') {
                    if (token.info === 'js') {
                        // 插入到import后，component前
                        moduleJS.push(token.content);
                    } else if (token.info === 'jsx') {
                        // 插入render内
                        jsx = token.content;
                    }
                }
                i++;
                token = tokens[idx + i]
            }

            // 描述也执行md
            return formatOpening(jsx, md.render(m[1]), flag);
        }
        return formatClosing(flag);
    }
});
```

代码具体含义可参照注释。至于`formatOpening`、`formatClosing
`只是简单的使用其他HTML标签简单的包裹了一下，可忽略次函数，直接`return m[1]` 或 `flag`。

**至此，对代码的处理也基本完成。**
可加入一些其他的美化代码后，放入`module.exports
`函数中输出。

### 4. 组装完成
此部分不做详细描述，可直接移步至[github](https://github.com/AdamantG/react-markdown-it-loader)查看。

