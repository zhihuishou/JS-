# JS逆向知识梳理

1.如何处理无限debbuger?

（1）HOOK注入，通过HOOK注入方式实现，替换原方法将原方法的函数变为空；通过HOOK注入方式可以对加密进行断点（诸如cookie内加密）也就是进行跟栈找加密位置，可以通过直接调用如下代码在浏览器调试中进行。

      AAA = Function.prototype.constructor;
      Function.prototype.constructor = function(a)
          {if (a=='debugger'){
          return function(){};
          }
                  return AAA(a);
          };

（2）方式：
