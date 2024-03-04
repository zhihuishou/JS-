JS逆向知识梳理
====

本内容为一些JS逆向知识的梳理，未来将会添加一些关于逆向案例在其中，也是对自己的知识总结与梳理，大家一起加油！

1.如何处理无限debbuger?
-------

> 无限debbuger原因：分析网络请求、查看元素的事件监听器、跟踪 js 等需求第一步就要打开浏览器的开发者工具，只要打开开发者工具就可能会碰到无限 debugger 死循环，或者在调试过程中也可能会出现无限 debugger 的死循环。

解决方法：

##### (1）HOOK注入，通过HOOK注入方式实现，替换原方法将原方法的函数变为空；通过HOOK注入方式可以对加密进行断点（诸如cookie内加密）也就是进行跟栈找加密位置，可以通过直接调用如下代码在浏览器调试中进行。

      AAA = Function.prototype.constructor;
      Function.prototype.constructor = function(a)
          {if (a=='debugger'){
          return function(){};
          }
                  return AAA(a);
          };

##### （2）重写关键函数，具体将debugger设置为空函数。重写关键函数可以指定方法名，或者使用 Function.prototype.constructor = function() {} ，这种方法只有在 (function(){}).constructor === Function 时才会生效。
      
      // 重写 eval 案例
       console.log(eval + '')
      // 'function eval() { [native code] }'
      
      // 重写 eval
      window._origin_eval = window.eval
      
      function $eval(src) {
        console.log(
          `==== eveal begin: length=${src.length}, caller=~${$eval.caller && $eval.caller.name} ====`
        )
        console.log(`injected ${document.location}`)
        console.log(src)
        console.log(`==== eval end ====`)
      
        return window._origin_eval(src)
      }
      
      Object.defineProperty(window, 'eval', { value: $eval })
      
      console.log(eval + '')
      // 'function $eval(src) {\n  console.log(\n    `==== eveal begin: length=${src.length}, caller=~${$eval.caller && $eval.caller.name} ====`\n  )\n  console.log(`injected ${document.location}`)\n  console.log(src)\n  console.log(`==== eval end ====`)\n\n  return window._origin_eval(src)\n}'
      
      $eval.toString = function () {
        return 'function eval() { [native code] }'
      }
      
      console.log(eval + '')
      // 'function eval() { [native code] }'

##### （3）断点调试，通过将debugger打上断点，将其设置为False.
      ![image](https://github.com/zhihuishou/Javascript_Reverse/assets/161868456/b68cfa3b-6a52-4680-8368-fc9866abdbc1)
      
2.如何处理webpack打包?
-------

> webpack是一个基于模块化的打包（构建）工具, 它把一切都视作模块，万变不离其中的就是它的自执行函数，只要在逆向中看到类似如下代码，基本可以确定就是webpack函数，从而需要找到如何运行加密模块的自执行函数：类似这个格式 !function('形参'){'加载器'}(['模块'])。

           !function(e) {
          var t = {};
      
          // 加载器  所有的模块都是从这个函数加载 执行
          function n(r) {
              if (t[r])
                  return t[r].exports;
              var o = t[r] = {
                  i: r,
                  l: !1,
                  exports: {}
              };
              return e[r].call(o.exports, o, o.exports, n),
                  o.l = !0,
                  o.exports
          }
      
          n(0)
      }
          ([
              function () {
                  console.log('123456')
              },
      
                    function () {
                  console.log('模块2')
              },
          ])
解决方法：
      扣 + 堆栈看，涉及到webpack，只要扣就完了。实现的方式，找到加密位置进行堆栈查看，找到加载器以及缺失的模块，构造如上的自执行函数，缺什么补什么。

3.JSPRC调用（RPC，英文 RangPaCong，中文让爬虫，旨在为爬虫开路，秒杀一切，让爬虫畅通无阻！ PS：玩笑话）
-------

> RPC和抓包工具逻辑是有点类似的（个人理解），都是通过中转站的逻辑将数据包回传，（RPC是通过websocket协议进行）。抓包工具是建立了通信证书的逻辑，从而可以抓到网页数据。而RPC在浏览器中将加密函数暴露出来，在本地直接调用浏览器中对应的加密函数，从而得到加密结果，不必去在意函数具体的执行逻辑，也省去了扣代码、补环境等操作，可以省去大量的逆向调试时间。


       
#### 原理：在网站的控制台新建一个WebScoket客户端链接到服务器通信，调用服务器的接口服务器会发送信息给客户端，客户端接收到要执行的方法执行完js代码后把获得想要的内容发回给服务器，服务器接收到后再显示出来。
       
参考大佬文章可以看这个：    [github:jxhczhl](https://github.com/jxhczhl/JsRpc?tab=readme-ov-file#%E5%AE%9E%E7%8E%B0) 

也可以看这个K哥的文章：     [k哥爬虫](https://baijiahao.baidu.com/s?id=1725536000710059774&wfr=spider&for=pc)

整体内容已经很丰富了，大概过一遍操作逻辑：
> 1.进入调试，将大佬的代码注入snippet

> 2.连接通信，这里就是相当于hook已经构建了一个function你要做的就是怎么让这个RPC连接和你之后要找的加密参数连起来，通过 new Hlclient("ws://127.0.0.1:12080/ws?group=zzz&name=hlg"); 建立一个http协议；

> 3.在你需要获取加密数据的地方打上你的断点，进行代码的调用。构造逻辑作者已经写好了，这里需要注意的是，在你的断点打上后，将构造的函数写入console中。

      demo.regAction("hello3", function (resolve,param) {
                //hello3 是你构建的函数，也就是在之后请求中你需要写进去的。
               // 这里的hlg就是你断点进行加密的函数
                res=hlg(param["user"],param["status"])
                resolve(res);
            })

![image](https://github.com/zhihuishou/Javascript_Reverse/assets/161868456/d8485aed-9a26-45a2-a97e-eab189f66b5c)

> 4.然后就是本地化调用，写最简单的request去请求就完了

