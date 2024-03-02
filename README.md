# JS逆向知识梳理

1.如何处理无限debbuger?

无限debbuger原因：分析网络请求、查看元素的事件监听器、跟踪 js 等需求第一步就要打开浏览器的开发者工具，只要打开开发者工具就可能会碰到无限 debugger 死循环，或者在调试过程中也可能会出现无限 debugger 的死循环。

解决方法：
（1）HOOK注入，通过HOOK注入方式实现，替换原方法将原方法的函数变为空；通过HOOK注入方式可以对加密进行断点（诸如cookie内加密）也就是进行跟栈找加密位置，可以通过直接调用如下代码在浏览器调试中进行。

      AAA = Function.prototype.constructor;
      Function.prototype.constructor = function(a)
          {if (a=='debugger'){
          return function(){};
          }
                  return AAA(a);
          };

（2）重写关键函数，具体将debugger设置为空函数。重写关键函数可以指定方法名，或者使用 Function.prototype.constructor = function() {} ，这种方法只有在 (function(){}).constructor === Function 时才会生效。
      
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

（3）断点调试，通过将debugger打上断点，将其设置为False.
      ![image](https://github.com/zhihuishou/Javascript_Reverse/assets/161868456/b68cfa3b-6a52-4680-8368-fc9866abdbc1)
      
2.如何处理webpack打包?

webpack是一个基于模块化的打包（构建）工具, 它把一切都视作模块，万变不离其中的就是它的自执行函数，只要在逆向中看到类似如下代码，基本可以确定就是webpack函数，从而需要找到如何运行加密模块的自执行函数：类似这个格式 !function('形参'){'加载器'}(['模块'])。

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
