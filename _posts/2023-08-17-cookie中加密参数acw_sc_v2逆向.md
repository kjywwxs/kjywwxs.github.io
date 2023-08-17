---
categories: 爬虫
tags: JS逆向 爬虫
---

# 起：为何要破解此参数

想要报名软考，但是每次都要点进网站看看是不是要报名了，属实过于麻烦，便想着每次启动电脑就可以自动看到最新可报名的软考项，于是简单写了个爬虫，复制curl转代码，获取可报名列表中最新的报名信息

```python
import requests

cookies = {
    'acw_tc': '2f6a1fd216922397931248074ead53308822501c0a0c63d6846ec6c8433bae',
    'acw_sc__v2': '64dd87b1b02c85aa09068d720ffae40dda957cad',
}

headers = {
    'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
    'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6',
    'Cache-Control': 'no-cache',
    'Connection': 'keep-alive',
    # Requests sorts cookies= alphabetically
    # 'Cookie': 'acw_tc=2f6a1fd216922397931248074ead53308822501c0a0c63d6846ec6c8433bae; acw_sc__v2=64dd87b1b02c85aa09068d720ffae40dda957cad',
    'Pragma': 'no-cache',
    'Referer': 'https://bm.ruankao.org.cn/sign/welcome',
    'Sec-Fetch-Dest': 'document',
    'Sec-Fetch-Mode': 'navigate',
    'Sec-Fetch-Site': 'same-origin',
    'Upgrade-Insecure-Requests': '1',
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/115.0.0.0 Safari/537.36 Edg/115.0.1901.203',
    'sec-ch-ua': '"Not/A)Brand";v="99", "Microsoft Edge";v="115", "Chromium";v="115"',
    'sec-ch-ua-mobile': '?0',
    'sec-ch-ua-platform': '"Windows"',
}

response = requests.get('https://bm.ruankao.org.cn/sign/welcome', cookies=cookies, headers=headers)
```

当时看到cookie中有两个参数acw_tc，和acw_sc__v2，就感觉不太妙，于是看了一眼过期时间：

![图 1](/images/a555fcf401ed9a5de05a86842dad575f2b85c3e5023d346964ba07b6009649e5.png)  


好家伙，半时就过期了，那试一下不带cookie可不可以正常返回数据，事实证明是我想得美了hh，不带cookies返回的是一段看不懂的js代码。

# 承：过程与困难

据我所知，cookie的生成主要有两种方式

1. 一种是服务端响应头中的set-cookie字段，浏览器会自动加如请求头中
2. js脚本中设置document.cookie

接下来看一下软考网的这个cookie是怎么生成  

打开开发者工具，清除cookie刷新网页，发现第一个请求就带了acw_tc，和acw_sc__v2两个cookie，可能是页面触发了reload,导致更早前的请求被刷掉了

打开抓包工具，清除cookie刷新，发现请求顺序如下：

![图 2](/images/a361b62dc926f0a342221aa93433e9a3c75ed9008e7faca6763251223ed6223e.png)  

果然有两个welcome请求，其中第一个请求响应头Set-Cookie设置了acw_tc，同时返回了一大串包裹在`<script>`标签中的js脚本，脚本中搜索cookie，找到了如下代码：

```js
function setCookie(name,value){
    var expiredate=new Date();
    expiredate.setTime(expiredate.getTime()+(3600*1000));
    document.cookie=name+"="+value+";expires="+expiredate.toGMTString()+";max-age=3600;path=/";
    }
function reload(x) {
    setCookie("acw_sc__v2", x);
    document.location.reload();
    }
```

acw_sc__v2参数生成的位置也找到了，参数值就是调用reload方法的参数x

>值得注意的是，调用的接口有一个safe.js，返回体如下，有一些反爬手段，粗略浏览一下，对我们影响不大

```js
layui.use(['jquery'], function () {
    var $ = layui.jquery;
    var div = document.createElement('div');
    var loop = setInterval(function () {
        console.log(div);
        console.clear();
    }, 200);

    Object.defineProperty(div, "id", {
        get: function () {
            clearInterval(loop);
            alert("禁止非法调试！请关闭开发者工具！")
            setInterval(breakDebugger, 100);//防止其他外部调试
        }
    });
    function checkDebugger() {
        var d = new Date();
        debugger;
        var dur = Date.now() - d;
        if (dur < 5) {
            return false;
        } else {
            alert("禁止非法调试！请关闭开发者工具！")
            return true;
        }
    }
    function breakDebugger() { if (checkDebugger()) { breakDebugger(); } }
    //其他扩展：
    //禁止右键
    $(document).bind("contextmenu", function () { return false; });
    var preventCtrl = function (e) {
        if (e.keyCode === 123) { //屏蔽F12 
            e.preventDefault();
            return false;
        } else if (e.keyCode === 17) { //ctrl
            console.log("prevent keycode s");
            document.onkeydown = preventS;
            return false;
        }
        return true;
    }
    var preventS = function (e) {
        if (e.keyCode === 123 || e.keyCode === 83) { //屏蔽F12 ctrl
            e.preventDefault();
            return false;
        }
        return true;
    }
    var nopreventS = function (e) {
        if (e.keyCode === 17) {
            console.log("no prevent keycode s");
            document.onkeydown = preventCtrl;
        }
    }
    //屏蔽f12, ctrl
    document.onkeydown = preventCtrl;
    document.onkeyup = nopreventS;
})
```

知道加密参数的位置后，使用本地覆盖的方法，断电、点到生成位置，往上跟栈

找到了reload(x)方法被异步调用的位置

```js
var _0x23a392 = arg1[_0x55f3('0x19', '\x50\x67\x35\x34')]();
arg2 = _0x23a392[_0x55f3('0x1b', '\x7a\x35\x4f\x26')](_0x5e8b26);
setTimeout('\x72\x65\x6c\x6f\x61\x64\x28\x61\x72\x67\x32\x29', 0x2);
```

\x编码的数据没法看懂，输入到控制台解码后，这段代码如下：

```js
var _0x23a392 = arg1[_0x55f3('0x19', '\x50\x67\x35\x34')]();
arg2 = _0x23a392[_0x55f3('0x1b', '\x7a\x35\x4f\x26')](_0x5e8b26);
setTimeout(reload(arg2), 2);
```

接下来就开始扣这些混淆的方法  
arg1是每次请求时都会返回的一个不同的字符串  
_0x55f3这个方法，貌似是返回一个用来获取arg某个方法的参数  
_0x23a392是执行获取的方法后的返回值，也是一个字符串  
_0x5e8b26参数多次试验，发现是个定值，为“3000176000856006061501533003690027800375”

所以我们扣_0x55f3这个方法

然后，噩梦来了，这个方法特别长，涉及的参数特别多，并且还调用很多其他方法，在扣的过程中，不知道内部做了什么处理，会导致堆栈溢出

>为什么正常浏览器用户浏览时不会堆栈溢出？有没有大神能为小弟解惑?

至此，局面陷入了僵局

# 转：柳暗花明

调试过程中突然发现，每次服务端返回的

```js
_0x55f3('0x19', '\x50\x67\x35\x34')
_0x55f3('0x1b', '\x7a\x35\x4f\x26')
```

其中参数都是固定的，会不会他们返回的数据也是固定的？

断点到指定位置，控制台执行对应方法，发现确实如此，一个固定为'unsbox'，一个固定为'hexXor'，所以代码变为如下形式

```js
var _0x23a392 = arg1['unsbox']();
arg2 = _0x23a392['hexXor'](_0x5e8b26);
```

接下来只要找到对应unsbox和hexXor方法就行，好巧不巧，这俩个方法竟然就在这两行代码上面一点点,半解码一下，代码如下：

```js
String['prototype']['hexXor'] = function(_0x4e08d8) {
    var _0x5a5d3b = '';
    for (var _0xe89588 = 0x0; _0xe89588 < this['length'] && _0xe89588 < _0x4e08d8['length']; _0xe89588 += 0x2) {
        var _0x401af1 = parseInt(this['slice'](_0xe89588, _0xe89588 + 0x2), 0x10);
        var _0x105f59 = parseInt(_0x4e08d8['slice'](_0xe89588, _0xe89588 + 0x2), 0x10);
        var _0x189e2c = (_0x401af1 ^ _0x105f59)['toString'](0x10);
        if (_0x189e2c['length'] == 0x1) {
            _0x189e2c = '0' + _0x189e2c;
        }
        _0x5a5d3b += _0x189e2c;
    }
    return _0x5a5d3b;
};

String['prototype']['unsbox'] = function() {
    var _0x4b082b = [0xf, 0x23, 0x1d, 0x18, 0x21, 0x10, 0x1, 0x26, 0xa, 0x9, 0x13, 0x1f, 0x28, 0x1b, 0x16, 0x17, 0x19, 0xd, 0x6, 0xb, 0x27, 0x12, 0x14, 0x8, 0xe, 0x15, 0x20, 0x1a, 0x2, 0x1e, 0x7, 0x4, 0x11, 0x5, 0x3, 0x1c, 0x22, 0x25, 0xc, 0x24];
    var _0x4da0dc = [];
    var _0x12605e = '';
    for (var _0x20a7bf = 0x0; _0x20a7bf < this['length']; _0x20a7bf++) {
        var _0x385ee3 = this[_0x20a7bf];
        for (var _0x217721 = 0x0; _0x217721 < _0x4b082b['length']; _0x217721++) {
            if (_0x4b082b[_0x217721] == _0x20a7bf + 0x1) {
                _0x4da0dc[_0x217721] = _0x385ee3;
            }
        }
    }
    _0x12605e = _0x4da0dc['join']('');
    return _0x12605e;
};
```

可以看到在字符串的原型上面增加了两个方法，并且方法内参数完全不依赖外部，至此，逆向完毕。

# 合：总结

1. 逆向经验+1
2. 了解acw_sc__v2参数生成方式
3. 死扣代码，补环境，反而陷入了堆栈溢出的问题（浏览器为啥可以正常允许访问呢？），需要静下来换个思路

最后，补一下完整的pyqt编写的软考提示代码：

```python
class MainWindow(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("自启动软考提示")
        self.resize(400, 300)

        self.label = QLabel(self)
        self.label.setAlignment(Qt.AlignCenter)
        self.label.setGeometry(50, 50, 300, 200)

        self.refresh_text()

    def refresh_text(self):
        # 发起接口请求获取文本内容
        text = self._get_infomation_from_url()
        # 更新标签的文本内容
        self.label.setText("最新可报名：" + text)

    def _get_infomation_from_url(self, url: str = 'https://bm.ruankao.org.cn/sign/welcome', cookies: dict = {}) -> str:
        '''返回最新软考报名信息：如 2023年上半年软考'''
        # cookies = {
        #     # 'acw_tc': '2f6a1fd116913336309368462e84d077d3b5ca67e70445988c8f307722b395',
        #     # 'acw_sc__v2': '64cfb3fe8a95b9779d0ffc75fe3ba8978387fd92',
        # }

        headers = {
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,image/apng,*/*;q=0.8,application/signed-exchange;v=b3;q=0.7',
            'Accept-Language': 'zh-CN,zh;q=0.9,en;q=0.8,en-GB;q=0.7,en-US;q=0.6',
            'Cache-Control': 'max-age=0',
            'Connection': 'keep-alive',
            # Requests sorts cookies= alphabetically
            # 'Cookie': 'acw_tc=2f6a1fd116913336309368462e84d077d3b5ca67e70445988c8f307722b395; acw_sc__v2=64cfb3fe8a95b9779d0ffc75fe3ba8978387fd92',
            'Referer': 'https://bm.ruankao.org.cn/sign/welcome',
            'Sec-Fetch-Dest': 'document',
            'Sec-Fetch-Mode': 'navigate',
            'Sec-Fetch-Site': 'same-origin',
            'Upgrade-Insecure-Requests': '1',
            'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/115.0.0.0 Safari/537.36 Edg/115.0.1901.188',
            'sec-ch-ua': '"Not/A)Brand";v="99", "Microsoft Edge";v="115", "Chromium";v="115"',
            'sec-ch-ua-mobile': '?0',
            'sec-ch-ua-platform': '"Windows"',
        }

        response = requests.get(url, cookies=cookies, headers=headers)
        if 'Set-Cookie' in response.headers:
            for k, v in response.cookies.items():
                # print(k + "=" + v)
                # 获得acw_tc作为cookie
                cookies[k] = v

        # print(response.text)
        html = etree.HTML(response.text)
        el_list = html.xpath("//*[@id='taskid']/option")
        if not el_list:
            # 如果列表为空明说明返回数据不完整，为第一次请求，缺乏cookies
            # 解析脚本并执行(执行的是手动逆向的脚本)，获得acw_sc__v2 作为cookie
            html_js_script = html.xpath("//script")[0].text
            # 获取arg1
            arg1 = re.findall(r"arg1='(.*?)';", html_js_script)[0]
            js_code = self._create_acw_sc__v2_js_code()
            # with open("软考网acw_sc__v2生成.js", 'r') as f:
            #     js_code = f.read()
            # 获得上下文对象
            context = compile(js_code)
            acw_sc__v2 = context.call('get_arg2', arg1)
            # 调用自己带完整cookies重新执行
            cookies['acw_sc__v2'] = acw_sc__v2
            return self._get_infomation_from_url(cookies=cookies)
        # print(el_list[0].text)  # 2023年上半年软考
        return el_list[0].text


    def _create_acw_sc__v2_js_code(self):
        js_code = """
        _0x5e8b26 = "3000176000856006061501533003690027800375"

        String['prototype']['hexXor'] = function(_0x4e08d8) {
            var _0x5a5d3b = '';
            for (var _0xe89588 = 0x0; _0xe89588 < this['length'] && _0xe89588 < _0x4e08d8['length']; _0xe89588 += 0x2) {
                var _0x401af1 = parseInt(this['slice'](_0xe89588, _0xe89588 + 0x2), 0x10);
                var _0x105f59 = parseInt(_0x4e08d8['slice'](_0xe89588, _0xe89588 + 0x2), 0x10);
                var _0x189e2c = (_0x401af1 ^ _0x105f59)['toString'](0x10);
                if (_0x189e2c['length'] == 0x1) {
                    _0x189e2c = '0' + _0x189e2c;
                }
                _0x5a5d3b += _0x189e2c;
            }
            return _0x5a5d3b;
        };

        String['prototype']['unsbox'] = function() {
            var _0x4b082b = [0xf, 0x23, 0x1d, 0x18, 0x21, 0x10, 0x1, 0x26, 0xa, 0x9, 0x13, 0x1f, 0x28, 0x1b, 0x16, 0x17, 0x19, 0xd, 0x6, 0xb, 0x27, 0x12, 0x14, 0x8, 0xe, 0x15, 0x20, 0x1a, 0x2, 0x1e, 0x7, 0x4, 0x11, 0x5, 0x3, 0x1c, 0x22, 0x25, 0xc, 0x24];
            var _0x4da0dc = [];
            var _0x12605e = '';
            for (var _0x20a7bf = 0x0; _0x20a7bf < this['length']; _0x20a7bf++) {
                var _0x385ee3 = this[_0x20a7bf];
                for (var _0x217721 = 0x0; _0x217721 < _0x4b082b['length']; _0x217721++) {
                    if (_0x4b082b[_0x217721] == _0x20a7bf + 0x1) {
                        _0x4da0dc[_0x217721] = _0x385ee3;
                    }
                }
            }
            _0x12605e = _0x4da0dc['join']('');
            return _0x12605e;
        };

         function get_arg2(arg1){
            var _0x23a392 = arg1['unsbox']();
            arg2 = _0x23a392['hexXor'](_0x5e8b26);
            return arg2
        }"""
        return js_code
```