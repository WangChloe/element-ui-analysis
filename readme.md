# element ui 源码学习

[element](https://github.com/ElemeFE/element)
版本:2.3.4

# 项目结构
src为核心代码和打包入口.
package为组件.
package/theme-chalk里为css样式.
examples为跑ui组件的用例,同时也是文档展示,用一个vue-markdown-loader的东西读取examples/docs里的md文件.


# mixins
## emitter
vue2.0废弃了$dispatch()和$dispatch(),
element里加回了类似功能的mixins.

broadcast是调用/触发指定子孙组件的事件,并非广义上的'广播'的概念.
dispatch是调用/触发指定父组件的事件.
```
function broadcast(componentName, eventName, params) {
  //遍历所有子组件
  this.$children.forEach(child => {
    var name = child.$options.componentName;
    //寻找符合指定名称的子组件
    if (name === componentName) {
      //在符合的子组件上触发的事件，但是不会再继续寻找符合名称的组件的子集
      child.$emit.apply(child, [eventName].concat(params));
    } else {
      //不符合继续寻找他的子集（即孙子组件）
      broadcast.apply(child, [componentName, eventName].concat([params]));
    }
  });
}
export default {
  methods: {
    dispatch(componentName, eventName, params) {
      var parent = this.$parent || this.$root;
      var name = parent.$options.componentName;
      //寻找父级，如果父级不是符合的组件名，则循环向上查找
      while (parent && (!name || name !== componentName)) {
        parent = parent.$parent;

        if (parent) {
          name = parent.$options.componentName;
        }
      }
      //找到符合组件名称的父级后，触发其事件。整体流程类似jQuery的closest方法
      if (parent) {
        parent.$emit.apply(parent, [eventName].concat(params));
      }
    },
    broadcast(componentName, eventName, params) {
      broadcast.call(this, componentName, eventName, params);
    }
  }
};
```
$option里的componentName也是为了配合这个mixins用的.

## migrating
似乎是为了用户使用老(废弃)API时做提示.

## focus
触发focus
```
export default function(ref) {
  return {
    methods: {
      focus() {
        this.$refs[ref].focus();
      }
    }
  };
};
```

# utils
## merge
可以理解为是Object.assign的一个polyfill
```
export default function(target) {
  for (let i = 1, j = arguments.length; i < j; i++) {
    let source = arguments[i] || {};
    for (let prop in source) {
      if (source.hasOwnProperty(prop)) {
        let value = source[prop];
        if (value !== undefined) {
          target[prop] = value;
        }
      }
    }
  }

  return target;
};
```

## dom
trim()的pollyfill
`\uFEFF`是对BOM的特殊处理.
```
const trim = function(string) {
  return (string || '').replace(/^[\s\uFEFF]+|[\s\uFEFF]+$/g, '');
};
```

-_等变成驼峰式，如foo-bar变成fooBar
```
const MOZ_HACK_REGEXP = /^moz([A-Z])/; //对firefox进行特殊判断

const camelCase = function(name) {
  return name.replace(SPECIAL_CHARS_REGEXP, function(_, separator, letter, offset) {
    // 开头的不大写，其余的大写
    return offset ? letter.toUpperCase() : letter;
  }).replace(MOZ_HACK_REGEXP, 'Moz$1');
};
```


addEventListener IE9+
```
export const on = (function() {
  if (!isServer && document.addEventListener) {// 浏览器环境，且非IE9以下版本
    return function(element, event, handler) {
      if (element && event && handler) {
        element.addEventListener(event, handler, false);
      }
    };
  } else {
    return function(element, event, handler) {
      if (element && event && handler) {
        element.attachEvent('on' + event, handler);
      }
    };
  }
})();
```

绑定一次性事件监听
```
export const once = function(el, event, fn) {
  var listener = function() {
    if (fn) { // 如果有回调，执行一次
      fn.apply(this, arguments);
    }
    off(el, event, listener);   // 移除监听
  };
  on(el, event, listener);
};
```

判断是否包含某className
```
export function hasClass(el, cls) {
  if (!el || !cls) return false; // 参数不足
  if (cls.indexOf(' ') !== -1) throw new Error('className should not contain space.');
  if (el.classList) { // 先看看支不支持原生的方法
    return el.classList.contains(cls);
  } else {
    return (' ' + el.className + ' ').indexOf(' ' + cls + ' ') > -1;  // 前后加空格来保证是完整匹配
  }
};
```

添加className
classList IE10+
```
export function addClass(el, cls) {
  if (!el) return;
  var curClass = el.className;
  var classes = (cls || '').split(' '); // 将多个类名以空格拆分

  for (var i = 0, j = classes.length; i < j; i++) {
    var clsName = classes[i];
    if (!clsName) continue; // 空格等则跳过

    if (el.classList) {
      el.classList.add(clsName);
    } else if (!hasClass(el, clsName)) {  // 保证不重复添加
      curClass += ' ' + clsName;
    }
  }
  if (!el.classList) {  // 更新类
    el.className = curClass;
  }
};
```

移除className
```
export function removeClass(el, cls) {
  if (!el || !cls) return;
  var classes = cls.split(' ');
  var curClass = ' ' + el.className + ' ';

  for (var i = 0, j = classes.length; i < j; i++) {
    var clsName = classes[i];
    if (!clsName) continue;

    if (el.classList) {
      el.classList.remove(clsName);
    } else if (hasClass(el, clsName)) {  // 使用空格替换掉相应的类
      curClass = curClass.replace(' ' + clsName + ' ', ' ');
    }
  }
  if (!el.classList) {
    el.className = trim(curClass);  // 移除首尾的空格
  }
};
```


取dom某样式的值
```
export const getStyle = ieVersion < 9 ? function(element, styleName) {
  if (isServer) return; // 如果是服务器端渲染直接返回
  if (!element || !styleName) return null; // 参数不足
  styleName = camelCase(styleName); // 将样式名转成驼峰式
  if (styleName === 'float') { // float特殊处理
    styleName = 'styleFloat';
  }
  try {
    switch (styleName) {
      case 'opacity':  // opacity特殊处理
        try {
          return element.filters.item('alpha').opacity / 100;
        } catch (e) {
          return 1.0;
        }
      default:
        return (element.style[styleName] || element.currentStyle ? element.currentStyle[styleName] : null);
    }
  } catch (e) {
    return element.style[styleName];
  }
} : function(element, styleName) {
  if (isServer) return;
  if (!element || !styleName) return null;
  styleName = camelCase(styleName);
  if (styleName === 'float') {
    styleName = 'cssFloat';
  }
  try {
    var computed = document.defaultView.getComputedStyle(element, '');
    return element.style[styleName] || computed ? computed[styleName] : null;
  } catch (e) {
    return element.style[styleName];
  }
};
```

设置样式
```
export function setStyle(element, styleName, value) {
  if (!element || !styleName) return;

  if (typeof styleName === 'object') { // 如果是对象则拆分后依次设置
    for (var prop in styleName) {
      if (styleName.hasOwnProperty(prop)) {  // 过滤掉非法的键值对??
        setStyle(element, prop, styleName[prop]);
      }
    }
  } else {
    styleName = camelCase(styleName);
    if (styleName === 'opacity' && ieVersion < 9) {
      element.style.filter = isNaN(value) ? '' : 'alpha(opacity=' + value * 100 + ')';
    } else {
      element.style[styleName] = value;
    }
  }
};
```

## popup
popup-manager 似乎是js生成一个悬浮框,但是具体实现没看明白.
index是相关操作popup-manager的方法之类的.
这里有逻辑如果有自定义滚动条,对于body的滚动条使用隐藏的方式处理.

## vue-popper
似乎是对popup的进一步封装.

## scrollbar-width
似乎是获取自定义滚动条的宽度.

## resize-event
通过scroll来实现模拟resize的事件监听.
大概就是检测滚动条的变化和距离这些来判定,具体实现没太看明白...

里面有addResizeListener, removeResizeListener这些方法.

## util
浅复制
```
function extend(to, _from) {
  for (let key in _from) {
    to[key] = _from[key];
  }
  return to;
};
```

转为对象
```
export function toObject(arr) {
  var res = {};
  for (let i = 0; i < arr.length; i++) {
    if (arr[i]) {
      extend(res, arr[i]);
    }
  }
  return res;
};
```

生成随机数
```
export const generateId = function() {
  return Math.floor(Math.random() * 10000);
};
```

## date.js
基本是[fecha](https://github.com/taylorhakes/fecha)库的源代码.

## popper.js
[popper.js](https://github.com/FezVrasta/popper.js)小幅修改版.

## sync.js
同步组件的prop和它上下文中的变量
```
const SYNC_HOOK_PROP = '$v-sync';

/**
 * v-sync directive
 *
 * Usage:
 *  v-sync:component-prop="context prop name"
 *
 * If your want to sync component's prop "visible" to context prop "myVisible", use like this:
 *  v-sync:visible="myVisible"
 */
export default {
  bind(el, binding, vnode) {
    // 获取上下文，即组件渲染的作用域中
    const context = vnode.context;
    const component = vnode.child;// 返回组件实例
    const expression = binding.expression;// 返回表达式
    const prop = binding.arg; // 返回传参

    if (!expression || !prop) {// 如果没有表达式或者传参
      console.warn('v-sync should specify arg & expression, for example: v-sync:visible="myVisible"');
      return;
    }

    if (!component || !component.$watch) {// 判断是否包含 watch 来确保是 Vue 的组件
      console.warn('v-sync is only available on Vue Component');
      return;
    }

    // 将组件的 prop 与上下文中的信息同步，返回取消 watch 的函数
    const unwatchContext = context.$watch(expression, (val) => {
      component[prop] = val;
    });

    // 将上下文中的信息和组件的 prop，返回取消 watch 的函数
    const unwatchComponent = component.$watch(prop, (val) => {
      context[expression] = val;
    });

    // 保存到组件实例中
    Object.defineProperty(component, SYNC_HOOK_PROP, {
      value: {
        unwatchContext,
        unwatchComponent
      },
      enumerable: false
    });
  },

  unbind(el, binding, vnode) {
    const component = vnode.child;
    if (component && component[SYNC_HOOK_PROP]) {// 取消监听
      const { unwatchContext, unwatchComponent } = component[SYNC_HOOK_PROP];
      unwatchContext && unwatchContext();
      unwatchComponent && unwatchComponent();
    }
  }
};

```

# directives
## clickoutside
一个点击元素外隐藏.
实现没看明白...

这个比较好理解
```
export default {
  bind (el, binding, vnode) {
    el.event = function (event) {
      if (!(el === event.target || el.contains(event.target))) {
        // call method provided in attribute value
        vnode.context[binding.expression](event)
      }
    }
    document.body.addEventListener('click', el.event)
  },
  unbind (el) {
    document.body.removeEventListener('click', el.event)
  }
}
```

## mousewheel
滚轮事件的兼容性处理
[normalize-wheel](https://github.com/basilfx/normalize-wheel)
```
import normalizeWheel from 'normalize-wheel';

const isFirefox = typeof navigator !== 'undefined' && navigator.userAgent.toLowerCase().indexOf('firefox') > -1;

const mousewheel = function(element, callback) {
  if (element && element.addEventListener) {
    element.addEventListener(isFirefox ? 'DOMMouseScroll' : 'mousewheel', function(event) {
      const normalized = normalizeWheel(event);
      callback && callback.apply(this, [event, normalized]);
    });
  }
};

export default {
  bind(el, binding) {
    mousewheel(el, binding.value);
  }
};
```

## repeat-click
控制鼠标按下时不断触发事件
```
import { once, on } from 'element-ui/src/utils/dom';

export default {
  bind(el, binding, vnode) {
    let interval = null;
    let startTime;
    // 获取表达式的内容
    const handler = () => vnode.context[binding.expression].apply();
    const clear = () => {
      // 如果当前时间距离开始时间少于 100ms，执行 handler
      if (new Date() - startTime < 100) {
        handler();
      }
      clearInterval(interval);
      interval = null;
    };

    // 绑定鼠标点击下的事件
    on(el, 'mousedown', (e) => {
      if (e.button !== 0) return;
      startTime = new Date();// 更新当前时间
      once(document, 'mouseup', clear);// 给鼠标抬起绑定一次性事件 clear
      clearInterval(interval);
      interval = setInterval(handler, 100);// 开始定时器
    });
  }
};
```


# theme-chalk
用的sass,用gulp单独打包,估计开发时是一边开watch一边开发的,变量的命名方式有点像css变量.

BEM命名风格
```
$namespace: 'el';
$element-separator: '__';
$modifier-separator: '--';
$state-prefix: 'is-';
```
```
@mixin b($block) {
  $B: $namespace+'-'+$block !global;

  .#{$B} {
    @content;
  }
}

@mixin e($element) {
  $E: $element !global;
  $selector: &;
  $currentSelector: "";
  @each $unit in $element {
    $currentSelector: #{$currentSelector + "." + $B + $element-separator + $unit + ","};
  }
}

@mixin m($modifier) {
  $selector: &;
  $currentSelector: "";
  @each $unit in $modifier {
    $currentSelector: #{$currentSelector + & + $modifier-separator + $unit + ","};
  }

  @at-root {
    #{$currentSelector} {
      @content;
    }
  }
}

@mixin when($state) {
  @at-root {
    &.#{$state-prefix + $state} {
      @content;
    }
  }
}
```

```
@include b(alert) {//.el-alert
  @include when(center) {}//.el-alert.is-center
  @include m(success) {}//.el-alert--success
  @include e(content) {}//.el-alert__content
```

# 动效(<transition>)
渐隐
```
.el-alert-fade-enter,
.el-alert-fade-leave-active {
  opacity: 0;
}
```

从上到下加渐隐
```
.dialog-fade-enter-active {
  animation: dialog-fade-in 0.3s;
}

.dialog-fade-leave-active {
  animation: dialog-fade-out 0.3s;
}

@keyframes dialog-fade-in {
  0% {
    transform: translate3d(0, -20px, 0);
    opacity: 0;
  }
  100% {
    transform: translate3d(0, 0, 0);
    opacity: 1;
  }
}

@keyframes dialog-fade-out {
  0% {
    transform: translate3d(0, 0, 0);
    opacity: 1;
  }
  100% {
    transform: translate3d(0, -20px, 0);
    opacity: 0;
  }
}
```

横向渐现变大
```
.el-zoom-in-center-enter-active,
.el-zoom-in-center-leave-active {
  transition: all .3s cubic-bezier(.55,0,.1,1);
}
.el-zoom-in-center-enter,
.el-zoom-in-center-leave-active {
  opacity: 0;
  transform: scaleX(0);
}
```

从左到右渐现
```
.carousel-arrow-left-enter,
.carousel-arrow-left-leave-active {
  transform: translateY(-50%) translateX(-10px);
  opacity: 0;
}
```

从右到左渐现
```
.carousel-arrow-right-enter,
.carousel-arrow-right-leave-active {
  transform: translateY(-50%) translateX(10px);
  opacity: 0;
}
```

# Alert
这个不是Alert方法,而只是一个摆放的可以隐藏的bar而已.
不同的type通过`el-alert el-alert--success`覆写样式.

# Container , Aside
组件里直接用了flex,那我为啥不直接用flex...

# Input
`inheritAttrs: false`关闭继承父组件非props的属性.

Form相关组件provide,broadcast,
被校验组件inject继承,dispatch触发.

隐藏ie10+自带的X
```
/** 文本输入框的 X  **/input::-ms-clear{display: none;}
/** 密码输入框的 X  **/input::-ms-reveal{display: none;}
```

input里有dispatch,也有对$parent的相关逻辑,用这种方法使form.item可进行交互验证.这种方式我暂时说不上好坏,直觉上觉得这种互相操作的方式很奇怪...

## calcTextareaHeight
计算内容textarea里撑开的高度，然后动态的设置textarea的height.
input里对value进行watch然后调用calcTextareaHeight.

# Autocomplete
数据请求加了debounce限制.

# Scrollbar
自定义滚动条组件.

# Badge
用的`<sup></sup>`标签实现的小红点.

$slots.default可用于判断.

小红点位置是这样算的:
```
position: absolute;
top: 0;
right: #{1 + $--badge-size / 2};
transform: translateY(-50%) translateX(100%);
```

# Breadcrumb
内部调用的是vue-router的跳转方法.

# Button
用了button-group设置多个button组合的样式.

# Card
box-shadow效果为
`$--box-shadow-light: 0 2px 12px 0 rgba(0, 0, 0, 0.1) !default;`

# Carousel
使用了throttle控制点击,防止动画没结束就滚到下一帧.
使用了resize动态变化.
无限轮播有点不同平常的子元素放在一个整体的父元素中滚动父元素,而是滚动全部子元素.

# 其他
## 未知
## 槽点
对自定义属性和dom操作依赖过重,没具体开发不知复杂后是否变得难以控制.
form校验不好用,实现的也感觉比较绕.
绑定事件非常多,不知道是不是通病.