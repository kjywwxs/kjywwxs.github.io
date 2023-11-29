
github pages 的markdown渲染默认不支持mermaid，在网页上只能看到原始文本，阅读体验较差，于是参照博文《[让github page支持mermaid语法](https://lingzihuan.icu/posts/2022-03-04-support-mermaid-on-github-page/)》中的方法

在post.html末尾，加上了如下代码

```js
文件：_layouts/post.html
<script src="https://cdn.jsdelivr.net/npm/mermaid/dist/mermaid.min.js"></script>
<script>
const config = {
    startOnLoad:true,
    theme: 'forest',
    flowchart: {
        useMaxWidth:false,
        htmlLabels:true
        }
};
mermaid.initialize(config);
window.mermaid.init(undefined, document.querySelectorAll('.language-mermaid'));
</script>
```

但是运行时发现，只有网页加载过程中**有一瞬间渲染正确**，然后就跳回空白的渲染代码，如下图

![图 0](/images/677cf7a1756870868b405aaef55c943e958724aebe5a63fac3c466ebc1b61ade.png)  

## 问题排查

找到渲染错误的元素，打上dom断点，选择子树修改类断点，刷新

发现断点在了如下的脚本中：

```js
<script>
// Init highlight js
document.addEventListener('DOMContentLoaded', function(event) {
  var els = document.querySelectorAll('pre code ')

  function addLangData(block) {
    var outer = block.parentElement.parentElement.parentElement;
    var lang = block.getAttribute('data-lang');
    for (var i = 0; i < outer.classList.length; i++) {
      var cls = outer.classList[i];
      if (cls.startsWith('language-')) {
        lang = cls;
        break;
      }
    }
    if (!lang) {
      cls = block.getAttribute('class');
      lang = cls ? cls.replace('hljs ', '') : '';
    }
    if (lang.startsWith('language-')) {
      lang = lang.substr(9);
    }
    block.setAttribute('class', 'hljs ' + lang);
    block.parentNode.setAttribute('data-lang', lang);
  }

  function addBadge(block) {
    var enabled = ('true' || 'true').toLowerCase();
    if (enabled == 'true') {
      var pre = block.parentElement;
      pre.classList.add('badge');
    }
  }

  function handle(block) {
    addLangData(block);
    addBadge(block)
    hljs.highlightBlock(block);

  }

  for (var i = 0; i < els.length; i++) {
    var el = els[i];
    handle(el);
  }
});
</script>
```

这段代码是初始化highlight js用的。
同时，注意到执行到此时mermaid区域渲染是正确的，继续运行程序，发现左边界面再次渲染错误

我们取消断点，在网络中屏蔽获取highlight.min.js的请求，刷新，页面渲染正常

![图 2](/images/b83ac7bfdac4f313dd36ab3e6d49de8e2d6dac8206ddb9ee2e86097866a20f3f.png)  

## 解决方案

找到项目中初始化highlight js 的代码，默认选择的高亮元素为CSS选择器`pre code`，我们单独将mermaid排除，不让其被highlight修改

```js
文件_includes\extensions\code-highlight.html

var els = document.querySelectorAll('pre code')

for (var i = 0; i < els.length; i++) {  
var el = els[i];  
// 检查元素是否有 'language-mermaid' 类  
if (el.classList.contains('language-mermaid')) {  
continue; // 跳过这次循环，不对此类进行处理  
}  
handle(el);

```
  