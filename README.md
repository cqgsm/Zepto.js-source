 尽管当下前端开发的技术栈中框架运用已经十分普遍，jQuery类的库正在衰落，不过jQuery中大量的DOM操作技巧并非变得无意义相反而是融入到了各个框架之中。Zepto.js是jQuery.js在移动浏览器端的最知名的替代，有着和jQuery一样的API和大部分相同的表现，同时它有少了很多用来兼容不同浏览器的代码，这使得它变得相当轻量，所以在现在阅读Zepto.js的代码相较于jQuery或许更有价值。本篇文章将详细解释Zepto.js的源码工作细节——从整体结构到某个函数的实现。之所以细致地去追求理解它的源码，目的有二：一、学习JavaScript代码的编写与组织技巧，了解一个库的开发中该有哪些考量与设计；二、学习DOM操作技巧，加深对DOM接口s的理解。 本篇文章讲解的是Zepto.js 1.2.0默认代码，不包含可选模块。另外，博客地址是http://codeduan.com/posts/zepto.html，同时也欢迎大家通过Issues来探讨。
# Zepto.js源码概况
## 代码逻辑结构
Zepto.js的最外层是个典型的IIFE。

``` JavaScript
(function(global,factory){
	if(typeof define==='function'&& define.amd)
		define(function(){
			return factory(global)
		})
	else factory(global)
}(this,function(window){
}))
```

在这里，需要注意AMD模块规范与this关键词的指向。
* AMD模块是JavaScript模块解决方案中的一种，在浏览器端最知名的实现是[RequireJS](http://requirejs.org/ "RequireJS")。一个标准的AMD模块写法如下:

```JavaScript
//foo.js

define(function(){
	return {
		foo:'foo',
		bar:'bar'
	}
})
```

使用时如下：

```JavaScript
//main.js
require(['foo'], function (foo) {
　　　　console.log(foo.foo)//'foo'
　　　　console.log(foo.bar)//'bar'
})
```

所以，在Zepto.js的最外层代码中，先检测当前环境是否支持AMD，如果支持则调用defined函数来生成AMD模块。也就是说，factory(global)将会返回一个对象，而这个对象就是Zepto.js入口函数对象。不支持AMD则直接调用，返回的入口对象将不会被捕获，因为factory函数内部将直接将入口函数添加在全局对象上，所以并不会影响访问。
* 在最外层代码中，非严格模式下的this关键词将指向window，这个是接下来代码运行的关键。所以，最终情况是factory函数以window对象为参数，在内部生成了Zepto.js入口对象并返回。

## 链式调用的实现
 链式调用是一种API风格，它在jQuery和Zepto.js中得到了很好的展现。它实现的原理十分简单，即在原型对象中返回this。一个简单实现的如下：
``` JavaScript
//JavaScript数组方法中有些不是返回被操作了的数组，所以需要链式调用的话可以这样实现
function ArrayChain(arr) {
    this.arr = arr;
  }

  ArrayChain.prototype = {
    shift: function() {
      this.arr.shift();
      return this;
    },
    push: function() {
      var arg = [].slice.call(arguments,0);
      arg.forEach(function(item) {
        this.arr.push(item);
      }, this)
      return this;
    },
    get: function() {
      return this.arr;
    }
  }

  var a=new ArrayChain([1,2,3]);
  a.shift().push(7,8,9).get()//[2,3,7,8,9]
  ```

## 编码风格
* 几乎不使用分号，除非必要
* 大量使用三元运算符

## API风格
* 驼峰命名法
* 很多方法在提供参数时是对Zepto.js集合中的所以元素进行遍历设置，空参数时获取集合中第一个元素的属性值

# Zepto.js的模块
## 模块总览
上面说到factory函数以window为参数执行，而factory函数结构如下：
```JavaScript
//factory函数，包含生成Zepto.js入口函数及各个模块的的完整逻辑
function (window){
//核心模块
var Zepto=(function(){
}());

//挂载Zepto.js入口函数到window上，并尝试添加别名$
window.Zepto=Zepto
window.$===undefined && (window.$=Zepto)

//事件模块
;
(function($){
})(Zepto)

//ajax模块
;
(function($){
})(Zepto)

//工具函数模块
;
(function($){
})(Zepto)

//重写window.getComputedStyle,IE浏览器上getComputedStyle被传入undefined时会报错
(function(){
try {
	getComputedStyle(undefined)
}catch(e){
	var nativeGetComputedStyle = getComputedStyle
      window.getComputedStyle = function(element, pseudoElement) {
        try {
          return nativeGetComputedStyle(element, pseudoElement)
        } catch (e) {
          return null
        }
      }
}
})()

//返回Zepto.js入口对象，只在使用AMD加载器的情况下有用
retrun Zepto.js
}
```
具体点说，在核心模块执行完成后Zepto.js的构造函数、原型对象、DOM除事件外的操作就已经完成，之后的模块是在扩充以上的函数对象，并最终返回Zepto.js入口函数，不过这只在使用AMD加载器的情况下有意义。
这里要注意：
* 最后一个IIFE目的是重写getComputeStyle函数。原因是在IE浏览器里当第一个参数不是DOM元素时会抛出错误，而核心模块的CSS部分会使用这个函数，Zepto.js希望即使参数并非预期也不抛出错误被浏览器捕获进而导致运行中断。
  接下来我将逐个讲解核心模块、事件模块、Ajax模块、工具函数模块。

## 核心模块
核心模块是Zepto.js主体逻辑结构和DOM除事件外操作的定义模块。和其他模块一样，核心模块也是一个IIFE，它返回Zepto.js入口函数。另外里面也有几个帮助实现Zepto.js逻辑的对象与函数，这里把主要的列出来：
* $函数：最终返回用户使用的Zepto.js入口函数。
  * $.Zepto对象(注意这是这个模块里的局部变量)：其中的函数与对象会被$入口函数使用
    * $.Zepto.init：用来形成Zepto.js对象的主体逻辑
    * $.Zepto.isZ：用来判断某个对象是否是Zepto.js对象
    * $.Zepto.Z：给数组或NodeList挂载原型对象，也就是各个Zepto.js对象有的方法，并且添加selector属性
      * $.Zepto.Z.prototype：指向$.fn，但$.Zepto.js.Z不是Zepto.js对象的构造函数
    * $.Zepto.fragment：用来创建DOM片段
    * $.Zepto.qsa：选择器函数
    * $.fn：原型对象，包含Zepto对象的原型
* Z构造函数：$.Zepto.Z的执行主体，用来形成每个包含元素的数值并添加selector属性，之后挂载$.fn对象为原型
* Z.prototype：指向$.fn，Z函数才是真正的Zepto对象的构造函数
  
除以上外，还有多个函数与对象被添加在$函数或$.Zepto对象上，不过它们都算辅助函数，所以暂时省略。现在我以一个例子来演绎以上函数与对象间的关系。
假设页面中有很多p元素，这样你会调用$('p')，这样会按顺序调用一下函数:
* 调用核心模块的$方法，以字符串'p'为参数
* 在$函数内部调用Zepto.init函数，同样以字符串'p'为参数
* 在Zepto.init函数内调用Zepto.qsa函数，得到由每个p元素组成的NodeList对象，再调用Zepto.Z函数
* 在Zepto.Z函数中执行new Z，这样，NodeList对象中的每个对象被依次放到一个对象中，并将该对象的原型指向$.fn对象
* 依次返回，这样使用者将得到一个以$.fn对象为原型并包含每个p元素的类数组，这就是最终使用的Zepto集合

接下来，我会详细说明Zepto.js的核心模块。

```JavaScript
var undefined, key, $, classList,emptyArray = [],
       
      //获得数组concat、fliter、slice的引用，数组方法是可以在类数组上调用的，因为这些方法只涉及数字下标和length属性
      concat = emptyArray.concat,
      filter = emptyArray.filter,
      slice = emptyArray.slice,
      document = window.document,
      //用来保存元素的默认display属性值
      elementDisplay = {},
      classCache = {},
      //保存可以是数值的CSS属性
      cssNumber = {
        'column-count': 1,
        'columns': 1,
        'font-weight': 1,
        'line-height': 1,
        'opacity': 1,
        'z-index': 1,
        'zoom': 1
      },

      //用来匹配文档碎片的正则表达式，形如<div>、<div></div>的都可以
      fragmentRE = /^\s*<(\w+|!)[^>]*>/,
      //匹配非嵌套标签，如<div><p></p></div>就不会被匹配，注意?:用来关闭捕获
      singleTagRE = /^<(\w+)\s*\/?>(?:<\/\1>|)$/,
      //用来将<div/>替换成<div></div>
      tagExpanderRE = /<(?!area|br|col|embed|hr|img|input|link|meta|param)(([\w:]+)[^>]*)\/>/ig,
      rootNodeRE = /^(?:body|html)$/i,
      //大写字母匹配
      capitalRE = /([A-Z])/g,
      
      //被封装成setter和getter的属性列表，用在使用入口函数创建文档片段时设置属性
      methodAttributes = ['val', 'css', 'html', 'text', 'data', 'width', 'height', 'offset'],
      
      //相对于某个元素的偏移操作的函数的属性名称列表
      adjacencyOperators = ['after', 'prepend', 'before', 'append'],

      table = document.createElement('table'),
      tableRow = document.createElement('tr'),
      containers = {
        'tr': document.createElement('tbody'),
        'tbody': table,
        'thead': table,
        'tfoot': table,
        'td': tableRow,
        'th': tableRow,
        '*': document.createElement('div')
      },
      //匹配文档可以交互的3种readyState值
      readyRE = /complete|loaded|interactive/,
      //检测是否是“简单选择器”
      simpleSelectorRE = /^[\w-]*$/,
      class2type = {},
      //获取Object.prototype.toString的引用，用来识别对象类型，如Object.prototype.toString.call([])将返回[object Array],表明这是一个数组
      toString = class2type.toString,
      Zepto.js = {},
      camelize, uniq,
      tempParent = document.createElement('div'),
      propMap = {
        'tabindex': 'tabIndex',
        'readonly': 'readOnly',
        'for': 'htmlFor',
        'class': 'className',
        'maxlength': 'maxLength',
        'cellspacing': 'cellSpacing',
        'cellpadding': 'cellPadding',
        'rowspan': 'rowSpan',
        'colspan': 'colSpan',
        'usemap': 'useMap',
        'frameborder': 'frameBorder',
        'contenteditable': 'contentEditable'
      },
      //简单封装的isArray函数，在没有Array.isArray的情况下使用instanceof运算符
      isArray = Array.isArray ||
      function(object) {
        return object instanceof Array
      }
```
接下来是Zepto.js.matches函数，用来检测元素是否与某个选择器匹配。
```JavaScript
Zepto.js.matches = function(element, selector) {
      if (!selector || !element || element.nodeType !== 1) return false
      var matchesSelector = element.matches || element.webkitMatchesSelector ||
        element.mozMatchesSelector || element.oMatchesSelector ||
        element.matchesSelector
      if (matchesSelector) return matchesSelector.call(element, selector)
      var match, parent = element.parentNode,
        temp = !parent
      if (temp)(parent = tempParent).appendChild(element)
      match = ~Zepto.js.qsa(parent, selector).indexOf(element)
      temp && tempParent.removeChild(element)
      return match
    }
```
它的实现方式是先判断元素是否实现了matchesSelector函数,没有的话就寻找父节点，在父节点上测试，如果未有父节点，则将元素插入tempParent元素（也就是一个空的div元素）中，生成父节点再测试，最后返回结果。这里使用了下面定义的Zepto.qsa函数，它是一个相当简洁的选择器。另外，为操作符用来将~-1计算为0，作为假值。
接下来是一系列工具函数的定义。
``` JavaScript
function type(obj) {
      return obj == null ? String(obj) :
        class2type[toString.call(obj)] || "object"
    }
```
type函数用来获取对象类型，null与undefined直接显示转化成null与undefined，之后便是调用Object.prototype函数的toString方法。其他的全是object。

``` JavaScript
function isFunction(value) {
      return type(value) == "function"
    }
```
判断对象是否为函数。

``` JavaScript
 function isWindow(obj) {
      return obj != null && obj == obj.window
 }

 function isDocument(obj) {
      return obj != null && obj.nodeType == obj.DOCUMENT_NODE
 }

 function isObject(obj) {
      return type(obj) == "object"
 }
```
前两个函数中必须先判断是否是null和undefined，因为null与undefined上的属性同样是undefined。isObject函数用来判断对象是否是广义的对象，比如各种元素对象也会返回true。

``` JavaScript

function isPlainObject(obj) {
      return isObject(obj) && !isWindow(obj) && Object.getPrototypeOf(obj)==Object.prototype
}
```
plainObject可以说是纯粹的对象，对象字面量就是干净的对象。它的原型必须是Object.prototype。

``` JavaScript
function likeArray(obj) {
      var length = !!obj && 'length' in obj && obj.length,
        type = $.type(obj)

      return 'function' != type && !isWindow(obj) && (
        'array' == type || length === 0 ||
        (typeof length == 'number' && length > 0 && (length - 1) in obj)
      )
    }
```
一个用判断对象是类数组或者数组的函数，为真必须要满足以下条件：
* 必须有length属性
* 不能是函数或者window对象，它们也有length属性
* 可以是数组
* length值必须合理，比如在length不为0时length-1必须存在

``` JavaScript
function compact(array) {
      return filter.call(array, function(item) {
        return item != null
      })
    }
```
compact函数用来去除数组中的null和undefined。

``` JavaScript
 function flatten(array) {
      return array.length > 0 ? $.fn.concat.apply([], array) : array
    }
```
简单封装的数值压平方法。

```JavaScript
camelize = function(str) {
      return str.replace(/-+(.)?/g, function(match, chr) {
        return chr ? chr.toUpperCase() : ''
      })
    }
```
字符串转驼峰形式。正则/-+(.)?/匹配“-”及后面的字符串。

``` JavaScript
function dasherize(str) {
      return str.replace(/::/g, '/')
        .replace(/([A-Z]+)([A-Z][a-z])/g, '$1_$2')
        .replace(/([a-z\d])([A-Z])/g, '$1_$2')
        .replace(/_/g, '-')
        .toLowerCase()
    }
```
将字符串转变为以“-”相隔的形式。形如“JavaScript”变为“Java-Script”，用于Zepto.js原型中的css函数实现。
``` JavaScript
 uniq = function(array) {
      return filter.call(array, function(item, idx) {
        return array.indexOf(item) == idx
     })
 }
```
数组去重，利用数组的indexOf方法返回某个元素在数组中的第一个索引实现，当遍历数组时某个元素的下标与第一个索引不一致时说明之前已经出现同样的值了。
``` JavaScript
function classRE(name) {
      return name in classCache ?
        classCache[name] : (classCache[name] = new RegExp('(^|\\s)' + name + '(\\s|$)'))
}
```
生成用来判定某个类是否在className中的正则表达式。它可能位于开通、中间、结尾。
``` JavaScript
function maybeAddPx(name, value) {
      return (typeof value == "number" && !cssNumber[dasherize(name)]) ? value + "px" :value
}
```
根据css属性名称name来判断value是否应该添加“px”单位。
```  JavaScript
function defaultDisplay(nodeName) {
      var element, display
      if (!elementDisplay[nodeName]) {
        element = document.createElement(nodeName)
        document.body.appendChild(element)
        display = getComputedStyle(element, '').getPropertyValue("display")
        element.parentNode.removeChild(element)
        display == "none" && (display = "block")
        elementDisplay[nodeName] = display
      }
      return elementDisplay[nodeName]
}
```

获取某种元素的默认显示方式。实现方式是创建一个临时节点，插入body元素中，之后使用getComputedStyle获取display属性值。不过，其实这不能真正地获取默认display值，因为这会被样式表影响。这个函数用在原型对象中的show方法上，因为要考虑显示出来时该有的display属性值。
``` JavaScript
 function children(element) {
      return 'children' in element ?
        slice.call(element.children) :
        $.map(element.childNodes, function(node) {
          if (node.nodeType == 1) return node
        })
 }
```
获取某个元素的子元素。先判断元素是否有children属性，否者获取元素的所有子节点，然后通过nodeType筛选出元素。

``` JavaScript
 function Z(dom, selector) {
      var i, len = dom ? dom.length : 0
      for (i = 0; i < len; i++) this[i] = dom[i]
      this.length = len
      this.selector = selector || ''
}
```
Z函数是生成Zepto.js对象的构造函数。下面会讲到的$.fn是Z.prototype属性，也就是原型对象。

``` JavaScript
 Zepto.fragment = function(html, name, properties) {
      var dom, nodes, container
      if (singleTagRE.test(html)) dom = $(document.createElement(RegExp.$1))

      if (!dom) {
        if (html.replace) html = html.replace(tagExpanderRE, "<$1></$2>")
        if (name === undefined) name = fragmentRE.test(html) && RegExp.$1
        if (!(name in containers)) name = '*'

        container = containers[name]
        container.innerHTML = '' + html
        dom = $.each(slice.call(container.childNodes), function() {
          container.removeChild(this)
        })
      }

      if (isPlainObject(properties)) {
        nodes = $(dom)	
        $.each(properties, function(key, value) {
          if (methodAttributes.indexOf(key) > -1) nodes[key](value)
          else nodes.attr(key, value)
        })
      }

      return dom
 }
```
Zepto.fragment是Zepto.js逻辑中的关键部分，当使用$('<div></div>')之类的语句时就是调用这个函数来实现创建DOM片段的。首先如果是简单标签，也就是非嵌套标签，则立即创建对应元素并执行$函数，否则，先获取最外层的标签名，之后使用innerHTML注入字符串到合适的容器对象中，最后分别取出形成数组。另外，如果有properties对象的，将刚刚形成的DOM节点列表生成Zepto集合，再调用attr方法设置。最后返回。这里要注意：
* 表格内部的标签只能存在与table及内部的如tbody标签内，否者只会留下空格换行节点，所以需要获取合适的父容器

``` JavaScript
Zepto.Z=function(dom,selector) {
	return new Z(dom,selector)
}
```
调用Z构造函数来生成Zepto集合。

``` JavaScript
Zepto.isZ = function(object) {
  return object instanceof Zepto.Z
}
```
通过instanceof操作符来判断对象是否是Zepto集合。

``` JavaScript
 Zepto.init = function(selector, context) {
      var dom
      if (!selector) return Zepto.Z()
      else if (typeof selector == 'string') {
        selector = selector.trim()
        if (selector[0] == '<' && fragmentRE.test(selector))
          dom = Zepto.fragment(selector, RegExp.$1, context), selector = null
        else if (context !== undefined) return $(context).find(selector)
        else dom = Zepto.qsa(document, selector)
      }
      else if (isFunction(selector)) return $(document).ready(selector)
      else if (Zepto.isZ(selector)) return selector
      else {
        if (isArray(selector)) dom = compact(selector)
        else if (isObject(selector))
          dom = [selector], selector = null
        else if (fragmentRE.test(selector))//冗余
          dom = Zepto.fragment(selector.trim(), RegExp.$1, context), selector = null//冗余
        else if (context !== undefined) return $(context).find(selector)

        else dom = Zepto.qsa(document, selector)
      }
      return Zepto.Z(dom, selector)
    }
```
Zepto的入口函数，也就是我们实际使用的$函数或者Zepto函数。它一共处理了下面几种情况：
* 完全没有参数，返回一个空的Zepto对象。其实没有人会这样使用，不过对于这样暴露给用户的API，它应该有良好的参数检测，防止异常
* 是字符串的情况可能有有两中，DOM片段和选择器，如果是DOM片段，调用之前的Zepto.fragment函数，这时，context应该是个对象。否则在不同的上下文里进行筛选。
* 如果selector是函数，则调用$(document).ready(selector)注册DOM可交互时的事件，并返回。
* 如果是已经是Zepto对象，返回它即可
* 如果是类数组，形成去除null和undefined的数组，保存在dom变量中，之后调用Zepto.Z(dom,selector)形成Zepto集合
* 如果是对象，也将它保存在dom变量的中，之后调用Zepto.Z生成Zepto集合

代码内部使用了接下来会介绍的Zepto.qsa函数，也就是选择器函数。另外，在判断selector是否为数组和对象的逻辑后面代码是冗余，上面已经有对字符串的处理了。

``` JavaScript
 $ = function(selector, context) {
      return Zepto.init(selector, context)
 }
```
入口函数，可以看到它是Zepto.init的简单封装。

``` JavaScript
function extend(target, source, deep) {
      for (key in source)
        if (deep && (isPlainObject(source[key]) || isArray(source[key]))) {
          if (isPlainObject(source[key]) && !isPlainObject(target[key]))
            target[key] = {}
          if (isArray(source[key]) && !isArray(target[key]))
            target[key] = []
          extend(target[key], source[key], deep)
        } else if (source[key] !== undefined) target[key] = source[key]
}
```
对象拷贝函数。deep为真则采用递归深拷贝。具体实现是检测source属性是原始值还是数组还是对象，如果是后两者，则再次调用extend。其实这里处理深拷贝并不严谨，可能会形成循环引用，不过Zepto.js目标就是轻量兼容，所以某些代码不严谨也很正常。

``` JavaScript
$.extend = function(target) {
      var deep, args = slice.call(arguments, 1)
      if (typeof target == 'boolean') {
        deep = target
        target = args.shift()
      }
      args.forEach(function(arg) { extend(target, arg, deep) })
      return target
}
```
将之前实现的extend函数进行封装，并暴露在$对象上。与extend函数相比，它利用数组的forEach方法来实现对多个对象拷贝的支持。

``` JavaScript
Zepto.qsa = function(element, selector) {
      var found,
        maybeID = selector[0] == '#',
        maybeClass = !maybeID && selector[0] == '.',
        nameOnly = maybeID || maybeClass ? selector.slice(1) : selector,
        isSimple = simpleSelectorRE.test(nameOnly)
      return (element.getElementById && isSimple && maybeID) ?//文档片段不支持getElementById
        ((found = element.getElementById(nameOnly)) ? [found] : []) :
        (element.nodeType !== 1 && element.nodeType !== 9 && element.nodeType !== 11) ? [] :
        slice.call(
          isSimple && !maybeID && element.getElementsByClassName ?
          maybeClass ? element.getElementsByClassName(nameOnly) : 
          element.getElementsByTagName(selector) :
          element.querySelectorAll(selector) 
        )
   }
```
这是Zepto的选择器函数，但使用$(selector)时调用的就是它。首先，Zepto.qsa检查了选择器的几个特征，包括是否是id选择器，或者类选择器，是否是简单的非复合选择器（这里指单个的id、class、tag并且命名常规的选择器）。这样，如果是简单的id选择器并且上下文支持getElementByID，则调用getElementById函数。之后排除掉不合理的上下文环境后，先是判断是否是简单的class选择器，再判断是否是简单的tag属性，最后仍未有结果就使用兼容性最强的querySelectorAll函数了。这里，要注意：
* 文档片段不支持getElementByID函数，有些浏览器文档片段不支持getElementsByClassName函数，所以要进行判断
* 使用id、class、tag选择器是出于性能的考虑。另外，Zepto.qsa函数只是原生API的简单封装，相比于jQuery的Sizzle选择器，它不支持特殊的选择方法，例如$('tr:odd')，它用来选择奇数索引的tr元素。

``` JavaScript
function filtered(nodes, selector) {
      return selector == null ? $(nodes) : $(nodes).filter(selector)
}
```
通过选择器来筛选Zepto对象。

``` JavaScript
$.contains = document.documentElement.contains ?
      function(parent, node) {
        return parent !== node && parent.contains(node)
      } :
      function(parent, node) {
        while (node && (node = node.parentNode))
          if (node === parent) return true
        return false
}
```
$.contains函数用来判断node参数是否是parent节点的子节点。这里，先尝试contains函数，否则不断获取node的parentNode属性，如果node.parentNode就是parent返回true，否者返回false。

``` JavaScript
function funcArg(context, arg, idx, payload) {
      return isFunction(arg) ? arg.call(context, idx, payload) : arg
}
```
用来对arg参数进行封装，在arg参数是函数时，funcArg函数返回的就是arg函数以context参数作为上下文，idx和payload作为参数的执行结果。在Zepto原型里，很多可以接受函数作为参数，实现追加的方法就是使用这个函数实现的。例如：
```
//html
<p id="para">hello</p>

// JavaScript
$("#para").html(function(i,origin){
	return origin+' world!'
})

$("#box").html()//hello world!

```

``` JavaScript
function setAttribute(node, name, value) {
      value == null ? node.removeAttribute(name) : node.setAttribute(name, value)
}
```
设置或移除节点的属性。使用了原生的DOM操作方法removeAttribute和setAttribute。

``` JavaScript
 function className(node, value) {
      var klass = node.className || '',
        svg = klass && klass.baseVal !== undefined

      if (value === undefined) return svg ? klass.baseVal : klass
      svg ? (klass.baseVal = value) : (node.className = value)
}
```
获取和设置className，特别的是，svg元素的className属性是个对象，不为undefined。

``` JavaScript
// "true"  => true
// "false" => false
// "null"  => null
// "42"    => 42
// "42.5"  => 42.5
// "08"    => "08"
// JSON    => parse if valid
// String  => self

function deserializeValue(value) {
      try {
        return value ?
          value == "true" ||
          (value == "false" ? false :
            value == "null" ? null :
            +value + "" == value ? +value :
            /^[\[\{]/.test(value) ? $.parseJSON(value) :
            value) : value
      } catch (e) {
        return value
      }
}
```
一个类型转换函数，用来将某些字符串转化为“更合理”的值。规则请看注释。

``` JavaScript
$.type = type
$.isFunction = isFunction
$.isWindow = isWindow
$.isArray = isArray
$.isPlainObject = isPlainObject
```
暴露几个之前定义的API。

``` JavaScript
$.isEmptyObject = function(obj) {
      var name
      for (name in obj) return false
      return true
}
```
检测对象是否为空。

``` JavaScript
$.isNumeric = function(val) {
      var num = Number(val),
        type = typeof val
      return val != null && type != 'boolean' &&
        (type != 'string' || val.length) &&
        !isNaN(num) && isFinite(num) || false
}
```
判断某个值是否可以把它当成数字或者就是数字，这里去除了几个不常规的数字。

``` JavaScript
$.inArray = function(elem, array, i) {
      return emptyArray.indexOf.call(array, elem, i)
}
```

``` JavaScript
$.inArray = function(elem, array, i) {
      return emptyArray.indexOf.call(array, elem, i)
}
```
判断某个元素是否在某个数组中并且是特定索引。

``` JavaScript
$.camelCase = camelize
```
暴露之前定义的驼峰化函数.

``` JavaScript
$.trim = function(str) {
      return str == null ? "" : String.prototype.trim.call(str)
} 
```
简单封装的去除字符串左右空白的函数，不过对str参数是null或undefined函数做了处理。

``` JavaScript
$.uuid = 0
$.support = {}//包含浏览器对某些API支持情况的信息，不过这里为空
$.expr = {}
$.noop = function() {}//一个什么都不做的函数
```
暴露一些变量，提高对jQuery的兼容，因为一些jQuery插件使用了这些变量

``` JavaScript
$.map = function(elements, callback) {
      var value, values = [],
        i, key
      if (likeArray(elements))
        for (i = 0; i < elements.length; i++) {
          value = callback(elements[i], i)
          if (value != null) values.push(value)
        }
      else
        for (key in elements) {
          value = callback(elements[key], key)
          if (value != null) values.push(value)
        }
      return flatten(values)
}
```
$.map函数通过遍历对象和数组调用callback函数来计算新的对象或数组，压平后返回。注意，$.map函数跳过null和undefined。

``` JavaScript
$.each = function(elements, callback) {
      var i, key
      if (likeArray(elements)) {
        for (i = 0; i < elements.length; i++)
          if (callback.call(elements[i], i, elements[i]) === false) return elements
      } else {
        for (key in elements)
          if (callback.call(elements[key], key, elements[key]) === false) return elements
      }
    return elements
}
```
$.each函数用来遍历一个对象或数组，并把它们作为上下文执行callback，不过，如果callback返回false时立即终止遍历并返回调用它的对象或数组。

``` JavaScript
$.grep = function(elements, callback) {
      return filter.call(elements, callback)
    }
```
利用数组filter函数封装的过滤工具函数。

``` JavaScript
if (window.JSON) $.parseJSON = JSON.parse
```
在$对象上引用JSON.parse函数。

``` JavaScript
$.each("Boolean Number String Function Array Date RegExp Object Error".split(" "), function(i, name) {
      class2type["[object " + name + "]"] = name.toLowerCase()
    })
```
在前面的type函数里，在对象上调用Object.prototype.toString函数后返回类似“[object Number]”的字符串，之后在class2type寻找对应的键的属性，即是该对象的类型。这里就填入了常见的类型。所以在这里，JavaScript中的原生对象的类型都将得到一个“更精确”的结果，而浏览器宿主对象中的各类对象则都会统一返回object。

现在，核心模块的结构已经完整了，完成Zepto.js逻辑的对象已经构建完成，工具函数也添加了不少。不过Zepto.js原型对象还空空如也。接下来，我开始详解$.fn对象，它定义了除事件外的所有DOM操作。
``` JavaScript $.fn对象
constructor: Zepto.js.Z
```
constructor属性引用Zepto.js.Z对象，表示一个Zepto.js对象是由Zepto.js.Z函数“构造出来的”。注意，constructor只是个普通的属性，它可以指向其他函数或对象，只是起一个简单标识的作用。

``` JavaScript
length: 0
```
集合长度，之后会被重写为正确的length。

``` JavaScript
 forEach: emptyArray.forEach
 reduce: emptyArray.reduce
 push: emptyArray.push
 sort: emptyArray.sort
 splice: emptyArray.splice
 indexOf: emptyArray.indexOf
```
引用一些数组的方法。之后某些原型方法中会用到。

``` JavaScript
concat: function() {
        var i, value, args = []
        for (i = 0; i < arguments.length; i++) {
          value = arguments[i]
          args[i] = Zepto.js.isZ(value) ? value.toArray() : value
        }
        return concat.apply(Zepto.js.isZ(this) ? this.toArray() : this, args)
}
```
concat函数将Zepto对象与函数参数使用数组的concat方法拼接在一起，注意，内部在Zepto对象上调用toArray方法变成数组，这样最终返回得结果就是一个数组而不是Zepto集合了。

``` JavaScript
map: function(fn) {
        return $($.map(this, function(el, i) {
          return fn.call(el, i, el)
        }))
}
```
对之前说明的$.map函数进行包装使之成为原型上的方法。通过回调函数执行得到的数组会通过$函数转为一个新的Zepto集合。


``` JavaScript
 slice: function() {
        return $(slice.apply(this, arguments))
 }
```
获取Zepto集合的切片，结果仍为Zepto集合。

``` JavaScript
ready: function(callback) {
        if (readyRE.test(document.readyState) && document.body) callback($)
        else document.addEventListener('DOMContentLoaded', function() { callback($) }, false)
        return this
},
```
ready函数接受一个函数为参数，它在readyState状态为complete或者loaded或者interactive时立即执行，否者说明文档还在加载，因此，注册DOMContentLoaded事件。

``` JavaScript
 get: function(idx) {
        return idx === undefined ? slice.call(this) : this[idx >= 0 ? idx : idx + this.length]
}
```
get函数用来提取Zepto集合里的元素。当不使用idx参数时返回包含所有元素的数组。当idx是负数是，idx+length获取实际下标。

``` JavaScript
toArray: function() {
   return this.get()
}
```
get函数参数为零时的简单封装。返回一个数组，包含所有元素。

``` JavaScript
size: function() {
  return this.length
}
```
获取集合中的元素个数。

``` JavaScript
remove: function() {
        return this.each(function() {
          if (this.parentNode != null)
            this.parentNode.removeChild(this)
   })
}
```
在DOM树中移除这个Zepto集合中的所有元素。

``` JavaScript
each: function(callback) {
        emptyArray.every.call(this, function(el, idx) {
          return callback.call(el, idx, el) !== false
        })
        return this
}
```
each函数用来遍历Zepto集合，直到callback函数返回false为止。each函数是Zepto.js原型中涉及集合操作的函数的基础，所有涉及批量设置的方法内部都使用了each方法。另外，其实这里可以不用数组的every函数实现,而是使用之前定义的$.each函数。如下：
``` JavaScript
each:function(callback){
	$.each(this,callback)
	return this	
}
```

``` JavaScript
 filter: function(selector) {
        if (isFunction(selector)) return this.not(this.not(selector))
        return $(filter.call(this, function(element) {
          return Zepto.js.matches(element, selector)
        }))
  }
```
filter函数用来过滤Zepto集合，可以接受字符串或者函数。在函数情况下，通过not函数过滤出selector函数返回值不为真的元素集合，再对这个集合过滤一下，得到selector函数为真的情况下的集合。在字符串情况下，使用之前说明过的Zepto.matches函数来判断。

``` JavaScript
add: function(selector, context) {
   return $(uniq(this.concat($(selector, context))))
}
```
扩展Zepto集合。注意，uniq用来去除重复的元素引用。

``` JavaScript
is: function(selector) {
        return this.length > 0 && Zepto.js.matches(this[0], selector)
}
```
用来判断集合的第一个元素是否与某个选择器匹配。

``` JavaScript
not: function(selector) {
        var nodes = []
        if (isFunction(selector) && selector.call !== undefined)
          this.each(function(idx) {
            if (!selector.call(this, idx)) nodes.push(this)
          })
        else {
          var excludes = typeof selector == 'string' ?this.filter(selector) :
            (likeArray(selector) && isFaunction(selector.item)) ? slice.call(selector) : $(selector)
          this.forEach(function(el) {
            if (excludes.indexOf(el) < 0) nodes.push(el)
          })
        }
        return $(nodes)
}
```
not函数用于在Zepto集合里选出与selector不匹配的元素。selector是函数时对函数返回值取反，获取不匹配的元素并放在数组中。字符串情况下，先获取符合该选取器的元素，之后用indexOf函数取反。类数组与NodeList的情况下，前者返回一个数组，后者返回一个Zepto集合，之后再用indexOf函数取反。

``` JavaScript
has: function(selector) {
        return this.filter(function() {
          return isObject(selector) ?
            $.contains(this, selector) :
            $(this).find(selector).size()
    })
 }
```
判断当前对象集合的子元素是否有符合选择器的元素，或者是否包含指定的DOM节点，如果有，则返回新的对象集合，函数内部$.contains方法过滤掉不含有选择器匹配元素或者不含有指定DOM节点的对象。在上文的isObject函数实现里，元素节点作为参数将返回true。

``` JavaScript
 eq: function(idx) {
        return idx === -1 ? this.slice(idx) : this.slice(idx, +idx + 1)
 }
```
返回新Zepto集合，包含通过索引获取Zepto集合元素。

``` JavaScript
first: function() {
        var el = this[0]
        return el && !isObject(el) ? el : $(el)
      },
last: function() {
        var el = this[this.length - 1]
        return el && !isObject(el) ? el : $(el)
 }
```
返回新的Zepto.js集合，包括第一个或最后一个元素。

``` JavaScript
find: function(selector) {
        var result, $this = this
        if (!selector) result = $()
        else if (typeof selector == 'object')
          result = $(selector).filter(function() {
            var node = this
            return emptyArray.some.call($this, function(parent) {
              return $.contains(parent, node)
            })
          })
        else if (this.length == 1) result = $(Zepto.js.qsa(this[0], selector))
        else result = this.map(function() {
          return Zepto.js.qsa(this, selector)
        })
    return result
}
```
find函数相当常用，用来在当前Zepto集合里筛选出新的Zepto集合。在selector是对象的情况下（指元素节点）先获取selector匹配的Zepto集合，之后对这个集合进行filter操作，将每个元素和调用find函数的Zepto集合进行匹配，只要这个集合中的元素能在调用find方法中的Zepto集合中找到，则过滤成功，并过滤下一个。selector是选择器时，通过map函数和Zepto.qsa搜寻。

``` JavaScript
 closest: function(selector, context) {
        var nodes = [],
          collection = typeof selector == 'object' && $(selector)
        this.each(function(_, node) {
          while (node && !(collection ? collection.indexOf(node) >= 0 : Zepto.js.matches(node, selector)))
            node = node !== context && !isDocument(node) && node.parentNode
          if (node && nodes.indexOf(node) < 0) nodes.push(node)
        })
        return $(nodes)
 }
```
closet函数用来寻找Zepto集合中符合selector和context的与当前Zepto集合元素最近的元素。这里使用了while循环来不断往祖先方向移动。另外还有排除掉重复的引用，因为一个集合中的元素有可能有相同的祖先元素。

``` JavaScript
 parents: function(selector) {
        var ancestors = [],
          nodes = this
        while (nodes.length > 0)
          nodes = $.map(nodes, function(node) {
            if ((node = node.parentNode) && !isDocument(node) && ancestors.indexOf(node) < 0) {
              ancestors.push(node)
              return node
            }
          })
        return filtered(ancestors, selector)
 }
```
获取Zepto集合内每个元素的所有祖先元素。函数内部维护了一个nodes数组用来保存所有结果，另外和closest函数一样同样使用while函数来层层递进。

``` JavaScript
parent: function(selector) {
        return filtered(uniq(this.pluck('parentNode')), selector)
}
```
parent函数用来返回由Zepto元素的父元素组成的Zepto集合。它内部使用的pluck方法，这个方法用来获取一个Zepto集合中每个元素的某个属性值组成的Zepto集合。另外，parent函数还进行了去重与过滤。

``` JavaScript
children: function(selector) {
        return filtered(this.map(function() {
          return children(this)
     }), selector)
 }
```
返回由Zepto集合中每个元素的后代元素组成的Zepto集合，可以筛选。函数内部使用之前定义的children函数来获取元素的后代元素。

``` JavaScript
contents: function() {
        return this.map(function() {
          return this.contentDocument || slice.call(this.childNodes)
    })
 }
```
返回Zepto集合中的元素的后代节点。对于frame，则获取它的contentDocument属性，contentDocument返回这个窗体的文档节点。

``` JavaScript
siblings: function(selector) {
        return filtered(this.map(function(i, el) {
          return filter.call(children(el.parentNode), function(child) {
            return child !== el
          })
        }), selector)
 }
```
获取Zepto集合的同辈元素。实现逻辑是获取元素的父元素再取出父元素的子元素，这样得到的集合就由父元素的所有子元素组成，之后使用“child!==el”来去除先前的元素。另外，形成新的Zepto集合时使用了filtered函数来过滤集合。

``` JavaScript
 empty: function() {
        return this.each(function() { this.innerHTML = '' })
 }
```
删除Zepto集合中每个元素的后代节点。这里直接使用更高效的innerHTML方法而非removeChild方法。

``` JavaScript
pluck: function(property) {
        return $.map(this, function(el) {
          return el[property]
     })
 }
```
pluck方法返回集合中每个元素的某个属性。上面的parent函数就使用它来获取节点的parent属性。

``` JavaScript
 show: function() {
        return this.each(function() {
          this.style.display == "none" && (this.style.display = '')
          if (getComputedStyle(this, '').getPropertyValue("display") == "none")
            this.style.display = defaultDisplay(this.nodeName)
        })
 }
```
show函数用来让元素显示成“默认样式”。实现逻辑是如果内联display属性为none，则使用“style.display=""”去除内联的值为none的display属性。实际上，使用style.propertyName=""或style.propertyName=""可以让对应内联样式失效。接下来对计算样式进行判断，如果仍为none的话则将样式还原为“默认样式”。

``` JavaScript
 replaceWith: function(newContent) {
        return this.before(newContent).remove()
 }
```
将Zepto集合中的元素替换成newContent。这里通过在元素前插入newContent后再移除元素完成。


``` JavaScript
 wrapAll: function(structure) {
        if (this[0]) {
          $(this[0]).before(structure = $(structure))
          var children
            // 获取结构中最里层的元素
          while ((children = structure.children()).length) structure = children.first()
          $(structure).append(this)
        }
        return this
 }
```
wrapAll函数用来将Zepto集合中的元素包裹在一个html片段或者DOM元素中。实现方式是先在集合中的第一个元素前插入structure生成的元素，之后遍历这个元素获取它的最里层元素，之后使用Zepto原型中的append方法将Zepto集合中的元素移动到刚刚获取的最里层元素中。

``` JavaScript
wrap: function(structure) {
        var func = isFunction(structure)
        if (this[0] && !func)
          var dom = $(structure).get(0),
            clone = dom.parentNode || this.length > 1

        return this.each(function(index) {
          $(this).wrapAll(
            func ? structure.call(this, index) :
            clone ? dom.cloneNode(true) : dom
          )
     })
  }
```
wrap方法用来包裹每个Zepto集合中的元素，内部使用了wrapAll方法和Zepto集合的遍历操作来包裹每个元素。注意， “clone = dom.parentNode || this.length > 1”用来判断structure是否要克隆，dom.parentNode存在意味着structure已经在文档之中了，这时应该将它克隆，不然会插入到已经存在structure之中，另外，Zepto集合中的元素多于一个时，也就是将有多个元素被包裹时也要进行克隆。

``` JavaScript
wrapInner: function(structure) {
        var func = isFunction(structure)
        return this.each(function(index) {
          var self = $(this),
            contents = self.contents(),
            dom = func ? structure.call(this, index) : structure
          contents.length ? contents.wrapAll(dom) : self.append(dom)
        })
}
```
wrapInner用来将Zepto集合中的每个元素的后代节点包裹起来。内部先使用contents方法获取Zepto集合中元素的后代节点，之后如果后代节点存在则使用wrapAll方法将它包裹，否则直接插入structure。

``` JavaScript
unwrap: function() {
        this.parent().each(function() {
          $(this).replaceWith($(this).children())
        })
     return this
 }
```
移除包裹结构。实现原理是遍历Zepto集合中的父元素，将它们替换成它们的后代节点。

``` JavaScript
 clone: function() {
        return this.map(function() {
          return this.cloneNode(true)
     })
  }
```
将Zepto集合中的元素克隆一份。这里利用了cloneNode函数，当它的参数为true是表示事件监听也会被克隆。

``` JavaScript
 hide: function() {
        return this.css("display", "none")
  }
```
通过设置“display属性为none”让集合元素不显示。这里用到了原型上的css方法。

``` JavaScript
toggle: function(setting) {
        return this.each(function() {
          var el = $(this);
          (setting === undefined ? el.css("display") == "none" : setting) ? el.show(): el.hide()
        })
 }
```
toggle函数接受一个参数，但参数为真时将Zepto集合显示，反之隐藏。这里通过show方法与hide方法实现显示与隐藏。另外，“(setting === undefined ? el.css("display") == "none" : setting)”存在冗余，所以toggle函数应该这样实现：
``` JavaScript
toggle: function(setting) {
        return this.each(function() {
          var el = $(this)
          setting? el.show(): el.hide()
        })
 }
```

``` JavaScript
 prev: function(selector) {
        return $(this.pluck('previousElementSibling')).filter(selector || '*')
      },
 next: function(selector) {
        return $(this.pluck('nextElementSibling')).filter(selector || '*')
  }
```
prev与next方法通过分别获取集合中每个元素的上一个元素与下一个元素来放回新的Zepto集合。这里使用pluck函数来获取新集合，使用filter函数进行筛选。

``` JavaScript
 html: function(html) {
        return 0 in arguments ?
          this.each(function(idx) {
            var originHtml = this.innerHTML
            $(this).empty().append(funcArg(this, html, idx, originHtml))
          }) :
          (0 in this ? this[0].innerHTML : null)
  }
```
html函数用来设置与获取集合中元素的innerHTML。当html参数存在时，则对集合中的元素进行遍历设置。这种情况下如果html参数是函数的话，它将在funcArg函数内执行并返回结果，并且原有的innerHTML将会作为参数，这样，就可以做到追加内容而不是完全重写。如果不传入html参数，则返回集合中第一个元素的innerHTML。

``` JavaScript
text: function(text) {
        return 0 in arguments ?
          this.each(function(idx) {
            var newText = funcArg(this, text, idx, this.textContent)
            this.textContent = newText == null ? '' : '' + newText
          }) :
          (0 in this ? this.pluck('textContent').join("") : null)
   }
```
text函数的使用与内部实现与html函数类似，不同点在于当获取textContent时采用了join方法来拼接字符串,这意味着它会将整个Zepto集合中各个元素的textContent连接起来一并返回。

``` JavaScript
 attr: function(name, value) {
        var result
        return (typeof name == 'string' && !(1 in arguments)) ?
          (0 in this && this[0].nodeType == 1 && (result = this[0].getAttribute(name)) != null ? result : undefined) :
          this.each(function(idx) {
            if (this.nodeType !== 1) return
            if (isObject(name))
              for (key in name) setAttribute(this, key, name[key])
            else setAttribute(this, name, funcArg(this, value, idx, this.getAttribute(name)))
          })
  }
```
attr方法用来设置和获取集合中元素的属性。在获取属性的情况下通过getAttribute函数返回属性值。如果用来设置则对集合进行遍历，这时，name可能是对象，或者value属性是函数，它们分别通过对象遍历与funcArg函数处理。

``` JavaScript
removeAttr: function(name) {
        return this.each(function() {
          this.nodeType === 1 && name.split(' ').forEach(function(attribute) {
            setAttribute(this, attribute)
          }, this)
        })
  }
```
使用之前定义的setAttribute函数来移除属性。“name.split（" "）”来把name参数分割成数组来进行遍历。split方法使用如下：
``` JavaScript
"foo bar".split(" ")//[foo,bar]
"foo".split(" ")//[foo]
"foo".split("")//[f,o,o]
```

``` JavaScript
prop: function(name, value) {
        name = propMap[name] || name
        return (1 in arguments) ?
          this.each(function(idx) {
            this[name] = funcArg(this, value, idx, this[name])
          }) :
          (this[0] && this[0][name])
 }
```
获取节点属性。内部实现与上面几个函数逻辑类似。propMap用来将一些特殊的属性进行正确转化，比如将“for”转为“htmlFor”。

```  JavaScript
removeProp: function(name) {
        name = propMap[name] || name
        return this.each(function() { delete this[name] })
 }
```
删除属性。这里直接使用delete操作符删除节点对象的属性。

``` JavaScript
 data: function(name, value) {
        var attrName = 'data-' + name.replace(capitalRE, '-$1').toLowerCase()

        var data = (1 in arguments) ?
          this.attr(attrName, value) :
          this.attr(attrName)

        return data !== null ? deserializeValue(data) : undefined
 }
```
data函数用来获取和添加元素的“数据”。它的内部使用以data为前缀的属性来实现，具体来说，它先把保存数据的键名中大写字母部分重写为已“-”分割的形式，接着进行DOM属性操作。在获取属性时，如果属性值不为null，则使用之前定义的deserializeValue函数处理后再返回。仅仅使用标准的data属性意味着它与jQuery中的data函数在功能上有很大的区别，后者可以使用各种数据结构。

``` JavaScript
val: function(value) {
        if (0 in arguments) {
          if (value == null) value = ""
          return this.each(function(idx) {
            this.value = funcArg(this, value, idx, this.value)
          })
        } else {
          return this[0] && (this[0].multiple ?
            $(this[0]).find('option').filter(function() {
              return this.selected
            }).pluck('value') :
            this[0].value)
        }
 }
```
val函数用于设置和获取元素的value属性。在查询的情况下，如果是select元素并且带有multiple属性，则将它被选中的option子元素筛选出来并通过pluck方法一并获取它们的value属性。

``` JavaScript
offset: function(coordinates) {
        if (coordinates) return this.each(function(index) {
          var $this = $(this),
            coords = funcArg(this, coordinates, index, $this.offset()),
            parentOffset = $this.offsetParent().offset(),
            props = {
              top: coords.top - parentOffset.top,
              left: coords.left - parentOffset.left
            }

          if ($this.css('position') == 'static') props['position'] = 'relative'
          $this.css(props)
        })
        if (!this.length) return null
        if (document.documentElement !== this[0] && !$.contains(document.documentElement, this[0]))
          return { top: 0, left: 0 }
        var obj = this[0].getBoundingClientRect()
        return {
          left: obj.left + window.pageXOffset,
          top: obj.top + window.pageYOffset,
          width: Math.round(obj.width),
          height: Math.round(obj.height)
        }
  }
```
offset方法用来获取元素的位置与大小信息。在获取操作中，如果元素尚未在DOM中则返回“{top:0,left:0}”,否则先是通过getBoundingClientRect方法获取元素相对于视口的位置属性，之后加上页面滚动的偏移量就得到元素相对于文档的位置了。对于设置的情况，遍历每个元素，获取它的父元素的offset属性，之后将要设置的偏移位置减去父元素的偏移位置就能得到实际应该设置的left与top属性。

``` JavaScript
 css: function(property, value) {
        if (arguments.length < 2) {
          var element = this[0]
          if (typeof property == 'string') {
            if (!element) return
            return element.style[camelize(property)] || getComputedStyle(element, '').getPropertyValue(property)
          } else if (isArray(property)) {
            if (!element) return
            var props = {}
            var computedStyle = getComputedStyle(element, '')
            $.each(property, function(_, prop) {
              props[prop] = (element.style[camelize(prop)] || computedStyle.getPropertyValue(prop))
            })
            return props
          }
        }

        var css = ''
        if (type(property) == 'string') {
          if (!value && value !== 0)
            this.each(function() { this.style.removeProperty(dasherize(property)) })
          else
            css = dasherize(property) + ":" + maybeAddPx(property, value)
        } else {
          for (key in property)
            if (!property[key] && property[key] !== 0)
              this.each(function() { this.style.removeProperty(dasherize(key)) })
            else
              css += dasherize(key) + ':' + maybeAddPx(key, property[key]) + ';'
        }

        return this.each(function() { this.style.cssText += ';' + css })
   }
```
css方法用来获取和设置样式的属性。在获取的情况下，显示通过style对象获取通过style属性设置的样式，否则使用window对象上的getComputedStyle方法获取计算样式。而设置功能则通过拼接元素的cssText属性实现，元素的的cssText属性返回该元素的style属性值字符串，对于某些属性，maybeAddPx函数用来添加“px”后缀’。设置时传入的属性值为假值且不为零时则使用style对象上的removeProperty方法来移除style属性内嵌的对应样式。另外，css方法支持批量操作。其实css函数可以做点改进，将查询情况下的return 语句改为：
``` JavaScript
 return element.style[camelize(property)] || getComputedStyle(element, '').getPropertyValue(dasherize(property))//dasherize函数将驼峰形式转化为横杠相连的形式
```
这样能支持驼峰形式的属性查询。

``` JavaScript
index: function(element) {
        return element ? this.indexOf($(element)[0]) : this.parent().children().indexOf(this[0])
 }
```
获取某个元素在Zepto集合中的索引，或者获取集合中第一个元素在相邻元素中的索引。

``` JavaScript
 hasClass: function(name) {
        if (!name) return false
        return emptyArray.some.call(this, function(el) {
          return this.test(className(el))
        }, classRE(name))
 }
```
判断集合中是否有元素满足某个类名。

``` JavaScript
addClass: function(name) {
        if (!name) return this
        return this.each(function(idx) {
          if (!('className' in this)) return
          classList = []
          var cls = className(this),
            newName = funcArg(this, name, idx, cls)
          newName.split(/\s+/g).forEach(function(klass) {
            if (!$(this).hasClass(klass)) classList.push(klass)
          }, this)
          classList.length && className(this, cls + (cls ? " " : "") + classList.join(" "))
        })
 }
```
addClass方法用来为集合中每个元素添加类名。具体实现是先通过className属性获取原有类名，接着将新类名放在数组之中，最后，将旧类名与即将要添加的新类拼接在一起再通过className函数重新赋值给className属性。具体的拼接规则是，原有的className属性加上一个合适的分隔字符串（如果原有className存在的话是" "否则是空字符串），最后再加上新类名数组调用join方法得到的以空格隔开的字符串，这样就能得到了全新的className属性值。另外函数还做了些检查工作，比如防止重复的类名被加入。

``` JavaScript
 removeClass: function(name) {
        return this.each(function(idx) {
          if (!('className' in this)) return
          if (name === undefined) return className(this, '')
          classList = className(this)
          funcArg(this, name, idx, classList).split(/\s+/g).forEach(function(klass) {
            classList = classList.replace(classRE(klass), " ")
          })
          className(this, classList.trim())
        })
 }
```
removeClass函数先为name参数，也就是即将被移除的类名生成能匹配元素className属性的正则表达式，比如当name为"foo"时，正则表达式为/(^|\s"")foo(\s|$)/，于是这个表达式将可以测试出元素的className是否包含这个类。在这里通过调用字符串的replacr方法直接将前文提到的正则表达式匹配到的类名替换成空字符串，这样就达到了移除某个类的目的。

``` JavaScript
toggleClass: function(name, when) {
        if (!name) return this
        return this.each(function(idx) {
          var $this = $(this),
            names = funcArg(this, name, idx, className(this))
          names.split(/\s+/g).forEach(function(klass) {
            (when === undefined ? !$this.hasClass(klass) : when) ?
            $this.addClass(klass): $this.removeClass(klass)
          })
        })
 }
```
toggleClass函数通过元素是否包含某个类来做出相反的添加或移除类的操作，或者根据when的真假性来添加或移除类。内部实现主要是通过addClass或者removeClass函数以及一些参数判断来完成。

``` JavaScript
 scrollTop: function(value) {
        if (!this.length) return
        var hasScrollTop = 'scrollTop' in this[0]
        if (value === undefined) return hasScrollTop ? this[0].scrollTop : this[0].pageYOffset
        return this.each(hasScrollTop ?
          function() { this.scrollTop = value } :
          function() { this.scrollTo(this.scrollX, value) })
  },
   scrollLeft: function(value) {
        if (!this.length) return
        var hasScrollLeft = 'scrollLeft' in this[0]
        if (value === undefined) return hasScrollLeft ? this[0].scrollLeft : this[0].pageXOffset
        return this.each(hasScrollLeft ?
          function() { this.scrollLeft = value } :
          function() { this.scrollTo(value, this.scrollY) })
  }
```
scrollTop与scrollLeft函数分别用来设置元素或者页面的滚动位置。在获取情况下先尝试获取元素的scrollTop与scrollLeft属性，否则获取pageYOffset与pageXOffset属性，后两个只在window对象上存在，这意味着$(document).scrollTop或者$(document).scrollLeft将不能获取文档相对于视口的偏移。在设置时，先尝试设置元素的scrollTop或者scrollLeft，否则在调用window上的scrollTo函数设置。

``` javaScript
position: function() {
        if (!this.length) return

        var elem = this[0],
          offsetParent = this.offsetParent(),
          offset = this.offset(),
          parentOffset = rootNodeRE.test(offsetParent[0].nodeName) ? { top: 0, left: 0 } : offsetParent.offset()

        // 减去元素的margin，因为offset函数返回的left与top是从边框开始计算的
        offset.top -= parseFloat($(elem).css('margin-top')) || 0
        offset.left -= parseFloat($(elem).css('margin-left')) || 0
       
       //加上border，不然接下来相减会多出border宽度的量
        parentOffset.top += parseFloat($(offsetParent[0]).css('border-top-width')) || 0
        parentOffset.left += parseFloat($(offsetParent[0]).css('border-left-width')) || 0

        return {
          top: offset.top - parentOffset.top,
          left: offset.left - parentOffset.left
        }
  }
```
position函数返回集合中第一个元素的相对于定位元素的偏移。实现原理是找出定位元素，这样就能得到定位元素的offset，也就是相对于文档的偏移。之后再获取第一个元素的相对于文档的偏移。然后，因为offset属性是从border开始计算偏移的，所以应该将第一个元素的offset.top与offset.left减去margin，这样得到的才是包括margin的偏移。另外，注意定位元素的border，因为最终返回的top和left是指集合中第一个元素到定位元素的content区域边缘的偏移。

``` JavaScript
offsetParent: function() {
        return this.map(function() {
          var parent = this.offsetParent || document.body
          while (parent && !rootNodeRE.test(parent.nodeName) && $(parent).css("position") == "static")
            parent = parent.offsetParent
          return parent
        })
 }
 
}//$.fn对象的属性与方法的定义暂时结束，接下来会用$.fn[method]来定义原型方法
```
获取元素定位时的参照元素。使用了offsetParent属性。

``` JavaScript
 $.fn.detach = $.fn.remove
```
给remove方法添加一个别名。

``` JavaScript
;
    ['width', 'height'].forEach(function(dimension) {
    //形成Width与Height两个首字母大写的字符串，方便形成innerWidth之类的属性名
      var dimensionProperty =
        dimension.replace(/./, function(m) {
          return m[0].toUpperCase()
        })

      $.fn[dimension] = function(value) {
        var offset, el = this[0]
        if (value === undefined) return isWindow(el) ? el['inner' + dimensionProperty] :
          isDocument(el) ? el.documentElement['scroll' + dimensionProperty] :
          (offset = this.offset()) && offset[dimension]
        else return this.each(function(idx) {
          el = $(this)
          el.css(dimension, funcArg(this, value, idx, el[dimension]()))
        })
      }
 })
```
在原型上添加width和height方法。在查询情况下需要对window对象，document对象和元素进行不同区分：window对象返回window.innerWidth（innerHeight）属性，document对象返回document.scrollWidth（scrollHeight）属性，如果是元素先查询offset，之后再返回对应属性。设置时，直接通过css函数设置。

``` JavaScript
 function traverseNode(node, fun) {
      fun(node)
      for (var i = 0, len = node.childNodes.length; i < len; i++)
        traverseNode(node.childNodes[i], fun)
}
```
递归遍历一个节点及其子节点。每遍历一个node执行一次fun(node)。

``` JavaScript
  adjacencyOperators.forEach(function(operator, operatorIndex) {
      var inside = operatorIndex % 2 //区分出“内部操作”与“外部操作”

      $.fn[operator] = function() {
        var argType, nodes = $.map(arguments, function(arg) {
            var arr = []
            argType = type(arg)
            if (argType == "array") {
              arg.forEach(function(el) {
                if (el.nodeType !== undefined) return arr.push(el)
                else if ($.Zepto.js.isZ(el)) return arr = arr.concat(el.get())
                arr = arr.concat(Zepto.js.fragment(el))
              })
              return arr
            }
            return argType == "object" || arg == null ?
              arg : Zepto.js.fragment(arg)
          }),
          parent, copyByClone = this.length > 1
        if (nodes.length < 1) return this

        return this.each(function(_, target) {
          parent = inside ? target : target.parentNode

          target = operatorIndex == 0 ? target.nextSibling :
            operatorIndex == 1 ? target.firstChild :
            operatorIndex == 2 ? target :
            null

          var parentInDocument = $.contains(document.documentElement, parent)

          nodes.forEach(function(node) {
            if (copyByClone) node = node.cloneNode(true)
            else if (!parent) return $(node).remove()

            parent.insertBefore(node, target)
            if (parentInDocument) traverseNode(node, function(el) {
              if (el.nodeName != null && el.nodeName.toUpperCase() === 'SCRIPT' &&
                (!el.type || el.type === 'text/javascript') && !el.src) {
                var target = el.ownerDocument ? el.ownerDocument.defaultView : window
                target['eval'].call(target, el.innerHTML)
              }
            })
          })
        })
      }
      
      //处理相反操作
      // after    => insertAfter
      // prepend  => prependTo
      // before   => insertBefore
      // append   => appendTo
      $.fn[inside ? operator + 'To' : 'insert' + (operatorIndex ? 'Before' : 'After')] = function(html) {
        $(html)[operator](this)
        return this
      }
  })

```

  这段代码用来为原型生成after，prepend等八个与在不同位置插入元素相关的方法。在DOM标准中，存在insertBefore方法来进行插入操作。所以，这八个方法都是改变insertBefore的调用元素和后一个参数来做到的。具体来说：
  * adjacencyOperator变量为['after', 'prepend', 'before', 'append']，这样，通过 var inside = operatorIndex % 2 就可以区分出这四个操作是“内部操作”还是“外部操作”，所谓“外部操作”指的是对发起这个操作的元素的相邻方向进行的操作，就是after与before，而“内部操作”指的是对发起操作的元素的子元素集合进行的操作，就是append与prepend。比如，after在数组中的索引是0，这样inside就是0，这表示after操作是外部操作。接下来，先获取参数返回一个包含即将要被插入的元素的集合，之后便是运用原生的insertBefore方法进行操作了。
  * 如何操作？第一，获取不同的parent元素，比如，当进行after插入时，parent元素的就是进行after操作的元素的父元素。第二，获取一个合适的元素作为insertBefore的第二个参数。于是，比如执行$('<p id="foo"></p').after('<p id="bar"></p>')，先是获取#foo的父元素作为insertBefore的调用上下文，获取#foo的nextSibling作为insertBefore的第二个参数，这样就可以在#foo之后插入#bar了。
  * 特殊地，当你插入script元素时，script插入后会被立即执行，此外，要把“</script>”写为“<\/script>”，因为前者会被解析错误。其实这里对脚本的处理只是一种当用户非要这样使用时的一种兼容，插入脚本的良好实现应该是在head标签内插入。
  * 这里说完了after，prepend，before，append的实现，对于另外四个方向的操作，实现的内部将它们进行相反调用即可。

``` JavaScript
 Zepto.Z.prototype = Z.prototype = $.fn
    Zepto.uniq = uniq
    Zepto.deserializeValue = deserializeValue
    $.Zepto = Zepto

    return $
  })()
  
 //这里的Zepto就是刚刚返回的$
 window.Zepto = Zepto
 window.$ === undefined && (window.$ = Zepto)
```
将形成的原型对象$.fn为挂载在Z.prototype与Zepto.Z.prototype上，将两个方法挂载在Zepto对象上，最后把Zepto挂载在$入口函数上，并返回$，最终把入口函数挂载在window对象上。至此，Zepto.js库的核心部分——主体结构与DOM除事件外的操作已经完成。

## 事件模块
事件模块是jQuery，Zepto之类的库广受欢迎的另一个原因，它为开发者提供了简洁的API来注册事件，虽然Zepto的使用主要分布在现代浏览器，尤其是移动浏览器上，它们往往遵循标准API，但对它们进行进一步封装仍有必要。对于Zepto的事件封装逻辑，可以先从以下几个函数了解：
* $.fn.on 它挂载在Zepto.js原型对象上，是我们注册事件的入口函数
* add函数 它在$.fn.on方法中被调用，内部调用了标准的addEventListener函数来监听事件
* $.fn.off 它挂载在Zepto.js原型对象上，是移除事件的入口函数
* remove函数 他在$.fn.off中使用，内部调用了removeEventListener函数来解除事件监听
* handlers对象，它用来保存所有使用Zepto库来监听事件的元素及事件的引用
另外，真正使用addEventListener函数注册的事件处理函数并不是使用on函数时传入的回调函数，在事件模块的内部，真正的回调函数在一个代理函数中执行，这也是实现通过返回false来禁用默认事件与停止事件传播功能的关键。
 现在举个简单的例子：
``` JavaScript
$('body').on('click',function(){
	console.log('body clicked')
})
```
这里$函数获取了包含body元素的Zepto集合，之后调用原型上的on方法，传入了“click”与事件处理函数两个参数，接着这两个参数再次作为参数传入add函数中，add函数接着被调用进而完成事件监听。这里只是大概介绍逻辑，事件模块其实会做相当多的处理的,远不止前面几个函数，逻辑比较复杂。接下里，我会完整解释事件模块源码。

``` JavaScript
;
(function($){
}(Zepto))
```
这是事件模块的模块形式，之前核心模块返回的入口函数$被赋值给Zepto，现在Zepto变量作为事件模块参数传入，这样在事件模块中就可以直接使用$来调用Zepto入口函数而不用考虑挂载在window对象上的是$还是Zepto。

``` JavaScript
 var _zid = 1,
      undefined,
      slice = Array.prototype.slice,
      isFunction = $.isFunction,
      isString = function(obj) {
        return typeof obj == 'string'
      },
      handlers = {},
      specialEvents = {},
      focusinSupported = 'onfocusin' in window,
      focus = { focus: 'focusin', blur: 'focusout' },
      hover = { mouseenter: 'mouseover', mouseleave: 'mouseout' }
```
这部分代码声明了很多变量。其中,_zid用来为注册事件的元素添加一个属性来标识唯一性，这里先初始化为1，handlers对象保存通过事件模块注册的事件与元素的应用，便于维护，比如你为body元素注册了三个事件，这样handlers对象就可能为：
``` JavaScript
{
1:[{},{},{}]//每个事件对应一个对象，对象里包含回调函数，选择器等属性
}
```
focusinSupported标示浏览器是否支持focusin事件。这在之后会说到，简单来说需要用focusin来替代focus事件。

``` JavaScript
specialEvents.click = specialEvents.mousedown = specialEvents.mouseup = specialEvents.mousemove = 'MouseEvents'
```
之后使用document.createEvent函数创建事件对象时使用。对于click，mousedown，mouseup，mousemove事件使用“MouseEvents”来标示。

``` JavaScript
 function zid(element) {
      return element._zid || (element._zid = _zid++)
 }
```
传入元素，如果有_zid属性则返回，否则添加_zid属性并更新它的值。

``` JavaScript
 function findHandlers(element, event, fn, selector) {
      event = parse(event)
      if (event.ns) var matcher = matcherFor(event.ns)
      return (handlers[zid(element)] || []).filter(function(handler) {
        return handler && (!event.e || handler.e == event.e) && (!event.ns || matcher.test(handler.ns)) && (!fn || zid(handler.fn) === zid(fn)) && (!selector || handler.sel == selector)
      })
}
```
findHandlers函数用来在handlers对象中寻找某个元素对应的包含事件信息的数组，并使用event，fn，selector来筛选，最终返回筛选后的数组。内部的parse函数用来解析注册事件时使用的字符串，返回事件名称与命名空间，因为我们注册事件时可能会像下面一样使用事件命名空间：
``` JavaScript
$('body').on('click.name',fn)
```
其中，name就是命名空间。

``` JavaScript
function parse(event) {
      var parts = ('' + event).split('.')
      return { e: parts[0], ns: parts.slice(1).sort().join(' ') }
}
```
parse函数用来解析注册事件时的字符串，把事件名与命名空间分离出来。比如parse('click.message.modal')将返回{e: "click", ns: "message modal"}，其中ns属性还经过排序处理。

``` JavaScript
 function matcherFor(ns) {
      return new RegExp('(?:^| )' + ns.replace(' ', ' .* ?') + '(?: |$)')
    }
```
生成某个命名空间对应的正则表达式，它可以用来检测某个事件是否属于某个命名空间。

``` JavaScript
 function eventCapture(handler, captureSetting) {
      return handler.del &&
        (!focusinSupported && (handler.e in focus)) ||
        !!captureSetting
    }
```
eventCapture函数返回一个布尔值，在注册事件的过程中标识是否使用在事件捕获的过程中触发事件。可以看到，只有在使用事件代理并且浏览器不支持focusin与focusout并且注册的事件为foucs或者blur时使用事件捕获，其他时候都是在冒泡阶段注册，除非强制将captureSetting设为true。这里详细解释下为什么要对focus与blur事件进行特殊处理。你可能知道，事件代理的实现原理是通过在实际要触发事件的元素的祖先元素上注册相应事件。比如：
``` html
<body>
	<div id="box"></div>
</body 
```

``` JavaScript
$('body').on('click','#box',function(ev){
	console.log('body clicked');
})
``
这样，当body上触发click时会判断事件对象的target属性是否指向#box元素，如果是则触发事件，这样就将#box上的click事件托管在了body元素上。但是，对于focus事件，它并不会冒泡，于是，如果事件代理还是通过在冒泡阶段注册那祖先对象将接收不到事件，因此无法完成事件代理。另外，focusin与focusout事件会冒泡，于是如果浏览器支持它们的话就仍在冒泡阶段处理代理，之后的逻辑中会把focus与blur事件替换成focusin与focusout。

``` JavaScript
function realEvent(type) {
      return hover[type] || (focusinSupported && focus[type]) || type
}
```
revlEvent函数返回真正要注册的事件。比如前面提到的将focus与blur替换成focusin与focusout。另外Zepto也把mouseenter与mouseleave事件替换成mouseover与mouseout事件，原因同样是前两者不支持冒泡。其他事件类型直接返回。

``` JavaScript
function add(element, events, fn, data, selector, delegator, capture) {
      var id = zid(element),
        set = (handlers[id] || (handlers[id] = []))
      events.split(/\s/).forEach(function(event) {
        if (event == 'ready') return $(document).ready(fn)
        var handler = parse(event)
        handler.fn = fn
        handler.sel = selector
          // 模仿mouseenter与mouseleave
        if (handler.e in hover) fn = function(e) {
          var related = e.relatedTarget
          if (!related || (related !== this && !$.contains(this, related)))
            return handler.fn.apply(this, arguments)
        }
        handler.del = delegator
        var callback = delegator || fn
        handler.proxy = function(e) {
          e = compatible(e)
          if (e.isImmediatePropagationStopped()) return
          e.data = data
          var result = callback.apply(element, e._args == undefined ? [e] : [e].concat(e._args))
          if (result === false) e.preventDefault(), e.stopPropagation()
          return result
        }
        handler.i = set.length
        set.push(handler)
        if ('addEventListener' in element)
          element.addEventListener(realEvent(handler.e), handler.proxy, eventCapture(handler, capture))
      })
 }
```
add函数是注册事件的关键逻辑，内部通过调用addEventListener函数来注册事件。它接受七个参数，分别是：要监听的元素，要注册的事件列表，事件处理程序，附加的数据对象，进行事件委托时实际要监听的元素的选择器，事件委托函数，一个标识事件捕获的布尔值。函数的实现逻辑如下：
* 设置或返回要注册事件的元素的_zid属性，它是handlers对象的键
* 创建或返回一个保存该元素事件信息对象的数组set
* 如果是ready事件，则在document上注册ready事件
* 解析event字符串，返回一个包含事件信息的对象handler
* 将回调函数与委托时的选择器挂载在handler上
* 如果要注册的事件是mouseenter或者mouseleave，则用mouseover或者mouseout替代。如何替代？这里需要对回调函数进行包装，也就是设计一个逻辑判断只留下符合mouseenter与mouseleave的情况留下。首先获取事件对象的relatedTarget属性，之后如果related为假值，这个情况意味着鼠标是从浏览器外移动到监听元素里的，所以执行实际的回调函数。当relatedTarget存在时，要满足relatedTarget不为监听元素（因为在mouseover的情况下从注册元素移动到内嵌的另外一个元素内或者mouseout情况下从内嵌的一个元素移动到注册元素时relatedTarget指向注册元素）并且注册元素不包含relatedTarget元素（原因与前一个语句类似）
* 将处理事件委托的函数挂载在handler对象的del属性上
* 生成最终被注册的proxy函数，它的通过函数的apply方法实际地调用了真正的回调函数，当函数实际的返回值为false时调用e.perventDefault和e.stopPropagation来禁用默认行为与停止事件传播，另外，它还对事件对象进行了扩展，下面会介绍到
* 在标识一个事件的handler对象上添加标识i，代表这个事件时这个元素通过Zepto.js注册的第几个事件，之后把handler放入set数组里
* 通过标准API addEventListener来监听事件

``` JavaScript
function remove(element, events, fn, selector, capture) {
      var id = zid(element);
      (events || '').split(/\s/).forEach(function(event) {
        findHandlers(element, event, fn, selector).forEach(function(handler) {
          delete handlers[id][handler.i]
          if ('removeEventListener' in element)
            element.removeEventListener(realEvent(handler.e), handler.proxy, eventCapture(handler, capture))
        })
      })
 }
```
remove函数与add函数相反，它通过removeEventListener来移除事件。首先获取元素的_zid属性，之后通过findHandlers函数和一些参数对handlers对象进行筛选，并返回一个包含handler对象的数组。接着，在维护所有事件的handlers对象中删除将要移除的handler并使用removeEventListener来移除事件。

``` JavaScript
$.event = { add: add, remove: remove }
```
将add与remove暴露给开发者，这意味着开发者可以直接使用，不过这意义不大。

``` JavaScript
$.proxy = function(fn, context) {
      var args = (2 in arguments) && slice.call(arguments, 2)
      if (isFunction(fn)) {
        var proxyFn = function() {
          return fn.apply(context, args ? args.concat(slice.call(arguments)) : arguments)
        }
        proxyFn._zid = zid(fn)
        return proxyFn
      } else if (isString(context)) {
        if (args) {
          args.unshift(fn[context], fn)
          return $.proxy.apply(null, args)
        } else {
          return $.proxy(fn[context], fn)
        }
      } else {
        throw new TypeError("expected function")
      }
    }
```
$.proxy是一个工具函数与Zepto的事件系统没有关系，功能是让函数的的上下文固定。函数通过传入函数fn和上下文对象context返回一个proxyFn函数，接下来直接调用proxyFn，这样fn就会在context的上下文下执行。函数内部通过Function.prototype上的apply函数实现。另外$.proxy函数支持对象与对象中的方法名作为参数调用。

``` JavaScript
$.fn.bind = function(event, data, callback) {
      return this.on(event, data, callback)
    }
$.fn.unbind = function(event, callback) {
      return this.off(event, callback)
    }
 $.fn.one = function(event, selector, data, callback) {
      return this.on(event, selector, data, callback, 1)
 }
```
对on方法进行简易封装，生成更特殊的bind，unbind和one方法。它们接受的参数个数少些，功能也就少些。

``` JavaScript
var returnTrue = function() {
        return true
      },
      returnFalse = function() {
        return false
      },
      ignoreProperties = /^([A-Z]|returnValue$|layer[XY]$|webkitMovement[XY]$)/,
      eventMethods = {
        preventDefault: 'isDefaultPrevented',
        stopImmediatePropagation: 'isImmediatePropagationStopped',
        stopPropagation: 'isPropagationStopped'
      }
 ```
声明两个函数用来返回布尔值，这在下面的compatible函数里会用到。ignoreProperties用来对原生的event对象进行过滤、eventMethods对象在下面的compatible函数中用到。

 ``` JavaScript
 function compatible(event, source) {
      if (source || !event.isDefaultPrevented) {
        source || (source = event)

        $.each(eventMethods, function(name, predicate) {
          var sourceMethod = source[name]
          event[name] = function() {
            this[predicate] = returnTrue
            return sourceMethod && sourceMethod.apply(source, arguments)
          }
          event[predicate] = returnFalse
        })

        event.timeStamp || (event.timeStamp = Date.now())

        if (source.defaultPrevented !== undefined ? source.defaultPrevented :
          'returnValue' in source ? source.returnValue === false :
          source.getPreventDefault && source.getPreventDefault())
          event.isDefaultPrevented = returnTrue
      }
      return event
    }

 ```
 
compatible函数用来对事件对象进行兼容性扩展。先是遍历eventMethods对象，获取类似preventDefault和isDefaultPrevented这样的键值对，这样就可以对原有的preventDefault、stopImmediatePropagation、stopPropagation三个方法进行重写，当这三个方法调用时，先把以is为前缀的判断方法是否执行过的方法重写为返回true的函数，之后才调用原来的方法。这样就让preventDefault、stopImmediatePropagation、stopPropagation与isPreventDefault、isImmediatePropagation和isPropagationStopped关联起来。
接着，compatible函数还对不支持event.timeStamp的浏览器进行了修复，不过标准中的event.timeStamp与Date.now()完全不同。compatible函数的最后检查了原事件是否处于禁用默认行为的状态，如果是则将isDefaultPrevented重写为returnTrue。

``` JavaScript
function createProxy(event) {
      var key, proxy = { originalEvent: event }
      for (key in event)
        if (!ignoreProperties.test(key) && event[key] !== undefined) proxy[key] = event[key]

      return compatible(proxy, event)
  }
```
在使用事件委托时，将原先事件对象传入createProxy函数返回一个新的事件对象，在新事件对象中使用originalEvent属性保存对原先对象的引用并且拷贝了原有事件对象的属性与方法。另外，函数内部过滤掉了一些没有用的属性。

```JavaScript
$.fn.delegate = function(selector, event, callback) {
      return this.on(event, selector, callback)
  }
$.fn.undelegate = function(selector, event, callback) {
      return this.off(event, selector, callback)
  }

 $.fn.live = function(event, callback) {
      $(document.body).delegate(this.selector, event, callback)
      return this
  }
 $.fn.die = function(event, callback) {
      $(document.body).undelegate(this.selector, event, callback)
      return this
  }
```
封装四个我们常用的事件行为，分别是事件委托与事件委托的解除，在body元素上委托事件和它的解除。

``` JavaScript
$.fn.on = function(event, selector, data, callback, one) {
      var autoRemove, delegator, $this = this
      if (event && !isString(event)) {
        $.each(event, function(type, fn) {
          $this.on(type, selector, data, fn, one)
        })
        return $this
      }

      if (!isString(selector) && !isFunction(callback) && callback !== false)
        callback = data, data = selector, selector = undefined
      if (callback === undefined || data === false)
        callback = data, data = undefined

      if (callback === false) callback = returnFalse

      return $this.each(function(_, element) {
        if (one) autoRemove = function(e) {
          remove(element, e.type, callback)
          return callback.apply(this, arguments)
        }

        if (selector) delegator = function(e) {
          var evt, match = $(e.target).closest(selector, element).get(0)
          if (match && match !== element) {
            evt = $.extend(createProxy(et), { currentTarge: match, liveFired: element })
            return (autoRemove || callback).apply(match, [evt].concat(slice.call(arguments, 1)))
          }
        }

        add(element, event, callback, data, selector, delegator || autoRemove)
      })
 }
```
on函数是注册事件的常用函数。前面提到的add函数就是在on函数处理好各种情况后调用，on函数的作用有以下几个：
* 处理第一个参数为对象的情况，当event为对象时on函数会使用它的键值对来注册事件
* 处理参数个数变动时的位置
* 当callback===false时，直接将回调函数设为returnFalse。这样每次执行事件时，因为callback的返回值为false所以在事件实际执行函数proxy内部检查这个返回值并停止冒泡并禁用默认事件
* 如果one参数为真，表示事件只执行一次，因此它的内部直接调用remove函数来移除事件并执行一下回调函数
* 当有selector时，表示采用事件委托，这样当事件触发时获取当前的event.target，通过closest获取符合selector的元素，如果成功匹配就说明事件委托成功，触发事件。注意delegator函数内还扩展了event对象。

``` JavaScript
$.fn.off = function(event, selector, callback) {
      var $this = this
      if (event && !isString(event)) {
        $.each(event, function(type, fn) {
          $this.off(type, selector, fn)
        })
        return $this
      }

      if (!isString(selector) && !isFunction(callback) && callback !== false)
        callback = selector, selector = undefined

      if (callback === false) callback = returnFalse

      return $this.each(function() {
        remove(this, event, callback, selector)
      })
  }
```
off函数与on函数相反，内部调用remove函数来移除事件。

``` JavaScript
$.fn.trigger = function(event, args) {
      event = (isString(event) || $.isPlainObject(event)) ? $.Event(event) : compatible(event)
      event._args = args
      return this.each(function() {
        if (event.type in focus && typeof this[event.type] == "function") this[event.type]()
        else if ('dispatchEvent' in this) this.dispatchEvent(event)
        else $(this).triggerHandler(event, args)
      })
 }
```
手动触发事件。内部通过$.Event函数返回一个DOM事件对象，接下来调用元素上的dispatchEvent函数触发事件。对于非DOM元素，通过调用tirggerHandler函数在handlers对象中过滤出合适的回调函数直接执行。另外，args参数可作为附加的参数填入event对象中。

```  JavaScript
$.fn.triggerHandler = function(event, args) {
      var e, result
      this.each(function(i, element) {
        e = createProxy(isString(event) ? $.Event(event) : event)
        e._args = args
        e.target = element
        $.each(findHandlers(element, event.type || event), function(i, handler) {
          result = handler.proxy(e)
          if (e.isImmediatePropagationStopped()) return false
        })
      })
      return result
 }
```
与trigger函数不同，triggerHandler函数并未在元素上通过dispatchEvent来触发事件，它直接使用findHandlers函数在维护所有Zepto.js事件的handlers对象上寻找出符合对应元素与事件的handler对象，并遍历执行handler.proxy（实际的回调函数被包装在里面）。

``` JavaScript
 ;
    ('focusin focusout focus blur load resize scroll unload click dblclick ' +
      'mousedown mouseup mousemove mouseover mouseout mouseenter mouseleave ' +
      'change select keydown keypress keyup error').split(' ').forEach(function(event) {
      $.fn[event] = function(callback) {
        return (0 in arguments) ?
          this.bind(event, callback) :
          this.trigger(event)
      }
   })
```
在$.fn原型对象上暴露常见事件，这样开发者就可以使用类似$('body').click(fn)这样的形式来注册事件。当不提供参数时使用trigger函数触发对应事件。

``` JavaScript
$.Event = function(type, props) {
      if (!isString(type)) props = type, type = props.type
      var event = document.createEvent(specialEvents[type] || 'Events'),
        bubbles = true
      if (props)
        for (var name in props)(name == 'bubbles') ? (bubbles = !!props[name]) : (event[name] = props[name])
      event.initEvent(type, bubbles, true)
      return compatible(event)
}
```
$.Event函数通过封装document.createEvent和event.initEvent函数提供了一个方便创建模拟事件对象的API。它在$.fn.trigger函数中就被用到。其中要注意的是document.createEvent函数接受事件名称来创建事件对象，这里应该提供更明确的事件名，比如“MouseEvents”。

在前面的代码中我们可以看到，Zepto内部维护一个对象来保存所以通过Zepto注册的事件，实际上就是一个发布订阅模式的实现。在后面的Ajax模块中，Zepto的事件功能还会被用到。


## Ajax模块
 Ajax模块通过对浏览器原生的XMLHttpRequest和JSONP技术进行封装从而提供一个更易用且兼容性更强的API，具体表现在：
* 易用的API
* 根据类型设置对返回的数据进行二次处理
* 和jQuery一样，对JSONP技术有着良好的支持
ajax的模块原理大致来说就是根据开发者提供的配置对象获取类似URL之类的信息，从而发起请求，并注册一个回调函数众多的readystatechange事件，将配置对象中的各类回调函数放在不同的条件分支下执行。另外，为了兼容性，Ajax模块内部并未封装responseType属性，这导致了它处理二进制文件的不便。Zepto.js提供了Deferred模块，它提供了类似promise的接口，不过接下来我将不做介绍。

``` JavaScript
;
(function($){
})(Zepto)
```
与其他模块类似，Ajax模块仍然以IIFE的形式存在。

``` JavaScipt
var jsonpID = +new Date(),
      document = window.document,
      key,
      name,
      rscript = /<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi,
      scriptTypeRE = /^(?:text|application)\/javascript/i,
      xmlTypeRE = /^(?:text|application)\/xml/i,
      jsonType = 'application/json',
      htmlType = 'text/html',
      blankRE = /^\s*$/,
      originAnchor = document.createElement('a')
```
一系列的变量声明。作用是：
* jsonpID：生成一个时间戳，用来当发起jsonp请求但未指定回调函数名称时形成一个唯一的回调函数名称，使用时间戳可以保证唯一。在发起多次jsonp请求时，后续只要在时间戳上不断自增就行
* rscript用来匹配script标签及其内容。在$.load函数里，需要对请求返回的script过滤掉，只留下正常内容

接下的几个是几种MIME类型，之后的originAnchor指向一个a元素，之后会被用来检测是否跨域。

``` JavaScript
originAnchor.href = window.location.href
```
将当前url保存，用于后面检测是否跨域。

``` JavaScript
 function triggerAndReturn(context, eventName, data) {
      var event = $.Event(eventName)
      $(context).trigger(event, data)
      return !event.isDefaultPrevented()
 }
```
用于在指定上下文触发指定事件。注意，在Ajax模块中，一些逻辑会使用前面讲解过的事件模块来发布一些事件。比如请求开始和结束阶段。

``` JavaScript
  function triggerGlobal(settings, context, eventName, data) {
      if (settings.global) return triggerAndReturn(context || document, eventName, data)
   }
```
在Ajax请求设定中，如果global为true则会通过triggerAndReturn函数触发事件。

``` JavaScript
 $.active = 0
```
标识当前正在进行的并且为“全局的”Ajax请求数量。

``` JavaScript
function ajaxStart(settings) {
      if (settings.global && $.active++ === 0) triggerGlobal(settings, null, 'ajaxStart')
    }

 function ajaxStop(settings) {
      if (settings.global && !(--$.active)) triggerGlobal(settings, null, 'ajaxStop')
 }
```
用于触发ajaxStart和ajaxStop事件。在Ajax请求和结束时段将调用它们。注意，因为内部让$.active自增或自减，因此只有当前没有其他正在进行的请求时才会触发两个事件。

``` JavaScript
 function ajaxBeforeSend(xhr, settings) {
      var context = settings.context
      if (settings.beforeSend.call(context, xhr, settings) === false ||
        triggerGlobal(settings, context, 'ajaxBeforeSend', [xhr, settings]) === false)
        return false

      triggerGlobal(settings, context, 'ajaxSend', [xhr, settings])
 }
```
调用Ajax配置中的beforeSend函数，之后尝试触发ajaxBeforeSend和ajaxSend事件，因为开发者对Ajax请求的设定可能不同，可能将不会触发它们。Ajax模块在Ajax发送请求前一刻将调用它。

``` JavaScript
function ajaxSuccess(data, xhr, settings, deferred) {
      var context = settings.context,
        status = 'success'
      settings.success.call(context, data, status, xhr)
      if (deferred) deferred.resolveWith(context, [data, status, xhr])
      triggerGlobal(settings, context, 'ajaxSuccess', [xhr, settings, data])
      ajaxComplete(status, xhr, settings)
 }
```
ajaxSuccess函数将在Ajax调用成功后调用，用来执行success回调函数并调用ajaxComplete函数。ajaxComplete函数功能类似，这意味着complete回调函数将在success回调函数后执行。

``` JavaScript
 // 错误类型: "超时t", "错误", "强制终止", "解析错误"
    function ajaxError(error, type, xhr, settings, deferred) {
      var context = settings.context
      settings.error.call(context, xhr, type, error)
      if (deferred) deferred.rejectWith(context, [xhr, type, error])
      triggerGlobal(settings, context, 'ajaxError', [xhr, settings, error || type])
      ajaxComplete(type, xhr, settings)
    }
```
ajaxError函数将在ajax请求超时，错误，强制终止和解析错误的情况下执行，它将调用Ajax配置中的error回调函数，接着触发全局的ajaxError事件并调用ajaxComplete函数。

``` JavaScript
// 完成类型: "成功", "内容未变动", "错误", "超时", "强制终止", "解析错误"
    function ajaxComplete(status, xhr, settings) {
      var context = settings.context
      settings.complete.call(context, xhr, status)
      triggerGlobal(settings, context, 'ajaxComplete', [xhr, settings])
      ajaxStop(settings)
 }
```
执行complete函数，触发ajaxComplete事件，最终调用ajaxStop触发ajaxStop事件。

``` JavaScript
 function ajaxDataFilter(data, type, settings) {
      if (settings.dataFilter == empty) return data
      var context = settings.context
      return settings.dataFilter.call(context, data, type)
 }
```
ajaxDataFilter函数用来对请求返回的数据进行过滤并返回。

``` JavaScript
function empty() {}
```
一个空函数作为Ajax配置中诸如success，complete等回调函数的默认值。

``` JavaScript
$.ajaxJSONP = function(options, deferred) {
      if (!('type' in options)) return $.ajax(options)

      var _callbackName = options.jsonpCallback,
        callbackName = ($.isFunction(_callbackName) ?
          _callbackName() : _callbackName) || ('Zepto.js' + (jsonpID++)),
        script = document.createElement('script'),
        originalCallback = window[callbackName],
        responseData,
        abort = function(errorType) {
          $(script).triggerHandler('error', errorType || 'abort')
        },
        xhr = { abort: abort },
        abortTimeout
      if (deferred) deferred.promise(xhr)

      $(script).on('load error', function(e, errorType) {
        clearTimeout(abortTimeout)
        $(script).off().remove()

        if (e.type == 'error' || !responseData) {
          ajaxError(null, errorType || 'error', xhr, options, deferred)
        } else {
          ajaxSuccess(responseData[0], xhr, options, deferred)
        }

        window[callbackName] = originalCallback
        if (responseData && $.isFunction(originalCallback))
          originalCallback(responseData[0])

        originalCallback = responseData = undefined
      })

      if (ajaxBeforeSend(xhr, options) === false) {
        abort('abort')
        return xhr
      }

      window[callbackName] = function() {
        responseData = arguments
      }
      
      script.src = options.url.replace(/\?(.+)=\?/, '?$1=' + callbackName)
      document.head.appendChild(script)

      if (options.timeout > 0) abortTimeout = setTimeout(function() {
        abort('timeout')
      }, options.timeout)

      return xhr
  }
```
$.ajaxJSONP函数用来完成JSONP请求。一个常见JSONP URL如coolshell.cn/t.php?n=10&callback=print，当发出如上请求时，将返回文本print(result)。所以将URL作为一个script元素的src属性并插入DOM中就会返回文本print(result)，它将作为js代码调用，最终浏览器将执行print(result)，这样就完成了一个JSONP的请求。而$.ajaxJSONP函数就对这些过程进行了封装。主要逻辑是：
* 如果请求对象过于简略，进入$.ajax函数，并在$.ajax函数内部做些必要的配置后重新进入$.ajaxJSONP逻辑。
* 如果没有指定回调函数名称jsonpCallback，使用之前定义的jsonp变量作为回调函数名并自增为下一次使用做准备
* 获取与回调函数名称相同的全局函数，之后会被重写
* 定义函数abort，这样当请求事件超时后将调用它进而触发error事件
* 定义一个xhr对象，在发出请求会返回，引用了abort函数，注意这里只是为了API风格一致，JSONP和XMLHttpRequest没有关系
* 监听刚刚创建的scrip标签的load和error事件，请求成功后将调用ajaxSuccess函数
* 重写与回调函数同名的全局函数，将它获取的参数对象赋值给responseData，假定请求返回的脚本会执行print(result)，result为数据，这样就会将result保存至responseData变量。
* 将URL带上回调函数，比如上面提到的URL使用时作为URL参数是coolshell.cn/t.php?n=10&callback=？，在之后的实际调用时将?替换回调函数名
* 将script元素插入head
* 使用timeout参数设置超时时间

``` JavaScript
$.ajaxSettings = {
      type: 'GET',
      beforeSend: empty,
      success: empty,
      error: empty,
      complete: empty,
      context: null,
      global: true,
      xhr: function() {
        return new window.XMLHttpRequest()
      },
       accepts: {
        script: 'text/javascript, application/javascript, application/x-javascript',
        json: jsonType,
        xml: 'application/xml, text/xml',
        html: htmlType,
        text: 'text/plain'
      },
   
      crossDomain: false,
      timeout: 0,
      processData: true,
      cache: true,
      dataFilter: empty
 }
```
保存默认的Ajax配置。HTTP请求方式默认为GET，之后的四个回调函数将会在请求的不同阶段调用、context指定前面的几个回调函数在执行时的上下文，默认为null、global默认为true，表示是否触发“全局事件”，“全局事件”意味着开发者可以以监听的方式对Ajax请求的不同阶段并作出反应，xhr引用实际的XMLHttpRequest对象，有时十分有用。accepts提供了几种MIME类型，在请求时会被写入请求中，以告知服务器期望的文件类型、crossDomain表示请求是否跨域、timeout用来设置可接受的请求时间，超时后停止请求，设置为0表示没有超时时间、processData表明是否对发起GET请求时的data属性进行进行编码以查询字符串的形式附加在URL后、cache表示是否缓存，浏览器自身会对资源缓存，而禁用缓存的原理是使用时间戳附加在URL生成唯一的URL从而禁用缓存。最后的dataFilter函数用来对请求返回的数据进行处理。

``` JavaScript
 function mimeToDataType(mime) {
      if (mime) mime = mime.split(';', 2)[0]
      return mime && (mime == htmlType ? 'html' :
        mime == jsonType ? 'json' :
        scriptTypeRE.test(mime) ? 'script' :
        xmlTypeRE.test(mime) && 'xml') || 'text'
 }
```
有时我们会将Ajax请求配置对象中的mimeType设置为如"text/plain; charset=x-user-defined"这样的形式，这样，Ajax模块会调用xhr.overrideMimeType函数来强制设置mimeType。mimeToDataType将通过mimeType返回对应的类型用于得到数据后的处理。函数的内部现将分号前的字符提前出来，之后先后判断是否为html、json、script、xml、text。

```  avaScript
function appendQuery(url, query) {
    if (query == '') return url
    return (url + '&' + query).replace(/[&?]{1,2}/, '?')
}
```
appendQuery函数用来为URL附加查询字符串。例如：
``` JavaScript
appendQuery('example.com','foo=1&bar=2')//example.com?foo=1&bar=2
```
一个合法的带有查询字符串的URL类似example.com?foo=1&bar=2，所以函数内部将直接拼接的字符串中第一个&字符替换成？后返回。

``` JavaScript
function serializeData(options) {
    if (options.processData && options.data && $.type(options.data) != "string")
      options.data = $.param(options.data, options.traditional)
    if (options.data && (!options.type || options.type.toUpperCase() == 'GET' || 'jsonp' == options.dataType))
      options.url = appendQuery(options.url, options.data), options.data = undefined
}
```
serializeData在$.ajax内部调用，作用是当使用GET请求时将option.data处理成字符串并调用刚刚说到的appendQuery函数追加到URL后。当使用POST时另有处理，因为POST请求数据得走send方法。

``` JavaScript
$.ajax = function(options){
   //扩展options对象，因为传入的options可能非常简略
    var settings = $.extend({}, options || {}),
        deferred = $.Deferred && $.Deferred(),
        urlAnchor, hashIndex
    for (key in $.ajaxSettings) if (settings[key] === undefined) settings[key] = $.ajaxSettings[key]
    
    //发布Ajax调用开始的事件
    ajaxStart(settings)
    
    //通过对比url检测是否真的跨域，这里针对IE浏览器
    if (!settings.crossDomain) {
      urlAnchor = document.createElement('a')
      urlAnchor.href = settings.url
      urlAnchor.href = urlAnchor.href
      settings.crossDomain = (originAnchor.protocol + '//' + originAnchor.host) !== (urlAnchor.protocol + '//' + urlAnchor.host)
    }

    if (!settings.url) settings.url = window.location.toString()
    if ((hashIndex = settings.url.indexOf('#')) > -1) settings.url = settings.url.slice(0, hashIndex)
    serializeData(settings)

    var dataType = settings.dataType, hasPlaceholder = /\?.+=\?/.test(settings.url)
    if (hasPlaceholder) dataType = 'jsonp'

    if (settings.cache === false || (
         (!options || options.cache !== true) &&
         ('script' == dataType || 'jsonp' == dataType)
        ))
      settings.url = appendQuery(settings.url, '_=' + Date.now())

    if ('jsonp' == dataType) {
      //有时dataType被设置为jsonp，但url并不符合jsonp要求，于是进行检测并自动添加
      if (!hasPlaceholder)
        settings.url = appendQuery(settings.url,
          settings.jsonp ? (settings.jsonp + '=?') : settings.jsonp === false ? '' : 'callback=?')
      return $.ajaxJSONP(settings, deferred)
    }

    var mime = settings.accepts[dataType],
        headers = { },
        setHeader = function(name, value) { headers[name.toLowerCase()] = [name, value] },
        protocol = /^([\w-]+:)\/\//.test(settings.url) ? RegExp.$1 : window.location.protocol,
        xhr = settings.xhr(),
        nativeSetHeader = xhr.setRequestHeader,
        abortTimeout

    if (deferred) deferred.promise(xhr)

    if (!settings.crossDomain) setHeader('X-Requested-With', 'XMLHttpRequest')
    setHeader('Accept', mime || '*/*')
    if (mime = settings.mimeType || mime) {
      if (mime.indexOf(',') > -1) mime = mime.split(',', 2)[0]
      xhr.overrideMimeType && xhr.overrideMimeType(mime)
    }
    //处理POST请求
    if (settings.contentType || (settings.contentType !== false && settings.data && settings.type.toUpperCase() != 'GET'))
      setHeader('Content-Type', settings.contentType || 'application/x-www-form-urlencoded')

    if (settings.headers) for (name in settings.headers) setHeader(name, settings.headers[name])
    xhr.setRequestHeader = setHeader
    
    //options对象中的回调函数就是存在于onreadystatechange回调函数中的各个分支中
    xhr.onreadystatechange = function(){
      if (xhr.readyState == 4) {
        xhr.onreadystatechange = empty
        //Zepto模块中的超时设计并未直接使用标准API中timeout，而是使用setTimeout函数，这里的意思是请求成功之后清除一下之前设定的计时器
        clearTimeout(abortTimeout)
        var result, error = false
        if ((xhr.status >= 200 && xhr.status < 300) || xhr.status == 304 || (xhr.status == 0 && protocol == 'file:')) {
          dataType = dataType || mimeToDataType(settings.mimeType || xhr.getResponseHeader('content-type'))
          //要使用XHR2级API去读取二进制文件意味着要在发出Ajax请求后手动配置responseType属性
          if (xhr.responseType == 'arraybuffer' || xhr.responseType == 'blob')
            result = xhr.response
          else {
            result = xhr.responseText

            try {
              result = ajaxDataFilter(result, dataType, settings)
              if (dataType == 'script')    (1,eval)(result)
              else if (dataType == 'xml')  result = xhr.responseXML
              else if (dataType == 'json') result = blankRE.test(result) ? null : $.parseJSON(result)
            } catch (e) { error = e }

            if (error) return ajaxError(error, 'parsererror', xhr, settings, deferred)
          }

          ajaxSuccess(result, xhr, settings, deferred)
        } else {
          ajaxError(xhr.statusText || null, xhr.status ? 'error' : 'abort', xhr, settings, deferred)
        }
      }
    }

    if (ajaxBeforeSend(xhr, settings) === false) {
      xhr.abort()
      ajaxError(null, 'abort', xhr, settings, deferred)
      return xhr
    }

    var async = 'async' in settings ? settings.async : true
    xhr.open(settings.type, settings.url, async, settings.username, settings.password)

    if (settings.xhrFields) for (name in settings.xhrFields) xhr[name] = settings.xhrFields[name]

    for (name in headers) nativeSetHeader.apply(xhr, headers[name])

    if (settings.timeout > 0) abortTimeout = setTimeout(function(){
        xhr.onreadystatechange = empty
        xhr.abort()
        ajaxError(null, 'timeout', xhr, settings, deferred)
      }, settings.timeout)

    // 为保证兼容，使用null替代空字符串
    xhr.send(settings.data ? settings.data : null)
    return xhr
  }
```
$.ajax是发起Ajax请求的入口函数，它对传入$.ajax函数的options配置对象进行解析，从而设置MIME类型，设置请求头，监听readystatechange事件进而调用各种回调函数。由于代码较长，我会将一部分讲解放在注释中。具体处理逻辑如下：
* 将配置对象options和$.ajaxSettings默认配置对象合并，形成最终使用的settings对象
* 调用ajaxStart函数，如果settings.global为true，触发全局的ajaxStart事件
* 调用serializeData函数序列化settings.data，如果是非POST请求的情况，则以查询字符串的形式附加在URL后
* 获取dataType，它表示开发者期望返回的数据类型
* 使用正则表达式判断URl是否是JSONP请求，如果是将dataType设为jsonp
* 处理缓存选项，如果cache为false，表示禁用缓存，这样会在URL后附加一个事件戳来使URL始终唯一从而实现禁用缓存。如果未明确cache选项，进一步检测请求类型，如果是script或者jsonp，则禁用缓存。cache为ture表示使用缓存。
* 对于JSONP请求，逻辑进入$.ajaxJSONP函数
* 通过dataType属性设置合适的MIMEType类型并使用xhr.overrideMimeType函数进行强制设置浏览器对返回数据的类型识别
* 当使用POST请求方式时，使用contentType属性代表的编码方式对数据编码，默认是application/x-www-form-urlencoded
* 注册onreadystatehange事件，options中的如success之类的回调函数会在不同的阶段触发。前面提到了Ajax模块内部并未使用XHR 2级 API，所以导致三个不同，它们分别是：如果想要对二进制数据方便地处理，应该使用$.ajax函数返回的实际xhr对象上的responseType属性（一个2级API）来获取二进制文件，所以这里产生了针对xhr.responseType的条件分支、对其他出二进制文件以外的数据直接responseText属性，这意味着对json文件需要调用parseJSON函数来解析获取的文本、超时终止的处理方法是通过setTimeout函数设置一个调用xhr.abort的回调函数来实现，而非xhr对象的timeout属性
* 在使用xhr.send方法前调用ajaxBeforeSend函数
* 设置是否为异步调用，其实浏览器默认的是同步调用
* 调用xhr.send函数发送请求
这里要注意一点：
  * Zepto的Ajax模块因为没有封装responseType属性，所以当你想处理二进制文件是最好像这样：
``` JavaScript
var xhr=$.ajax({
      url: '1.jpg',
      success:function (data) {
        console.log(data)
      }
    })    
xhr.responseType='blob'
```
也就是对发起Ajax请求后返回的xhr对象添加responseType属性，因为Ajax模块内部在请求成功后做了检测。这样，最终传给回调函数的data是response属性而非responseText属性。

``` JavaScript
function parseArguments(url, data, success, dataType) {
    if ($.isFunction(data)) dataType = success, success = data, data = undefined
    if (!$.isFunction(success)) dataType = success, success = undefined
    return {
      url: url
    , data: data
    , success: success
    , dataType: dataType
    }
  }
```
parseArguments函数用来解析四个参数，并返回一个可以传入$.ajax函数的对象，用在$.get等简写方法内。代码前面的if语句用来处理不同参数个数的情况。

``` JavaScript
$.get = function(/* url, data, success, dataType */){
    return $.ajax(parseArguments.apply(null, arguments))
  }

  $.post = function(/* url, data, success, dataType */){
    var options = parseArguments.apply(null, arguments)
    options.type = 'POST'
    return $.ajax(options)
  }

  $.getJSON = function(/* url, data, success */){
    var options = parseArguments.apply(null, arguments)
    options.dataType = 'json'
    return $.ajax(options)
  }
```
三个常用的简写方法，使用了前面介绍的parseArguments函数，注意这里因为这三个参数的个数会有不同情况，所以使用了函数对象的apply方法调用。

``` JavaScript
$.fn.load = function(url, data, success){
    if (!this.length) return this
    var self = this, parts = url.split(/\s/), selector,
        options = parseArguments(url, data, success),
        callback = options.success
    if (parts.length > 1) options.url = parts[0], selector = parts[1]
    options.success = function(response){
      self.html(selector ?
        $('<div>').html(response.replace(rscript, "")).find(selector)
        : response)
      callback && callback.apply(self, arguments)
    }
    $.ajax(options)
    return this
}
```
load函数用来加载html片段，它和普通的Ajax请求没有区别，只是通过innerHTML方法将返回的文本文件的字符填入一个DOM片段中，形成一个新的DOM片段，这样就可以在文档中使用了。有时候只需要将返回的DOM片段中符合某个选择器的片段提取出来，只需要在url后加个空格之后再跟上一个选择器就行，所以，函数中将url分成两部分，之后再进行选取。

``` JavaScript
var escape = encodeURIComponent
```
之后会使用encodeURIComponent函数进行对字符串编码最后拼接到URL。

``` JavaScript
function serialize(params, obj, traditional, scope){
    var type, array = $.isArray(obj), hash = $.isPlainObject(obj)
    $.each(obj, function(key, value) {
      type = $.type(value)
      if (scope) key = traditional ? scope :
        scope + '[' + (hash || type == 'object' || type == 'array' ? key : '') + ']'
      if (!scope && array) params.add(value.name, value.value)
      else if (type == "array" || (!traditional && type == "object"))
        serialize(params, value, traditional, key)
      else params.add(key, value)
    })
 }
 $.param = function(obj, traditional){
    var params = []
    params.add = function(key, value) {
      if ($.isFunction(value)) value = value()
      if (value == null) value = ""
      this.push(escape(key) + '=' + escape(value))
    }
    serialize(params, obj, traditional)
    return params.join('&').replace(/%20/g, '+')
  }
})(Zepto.js)
```
$.param函数用来将对象序列化且编码为在URL中合法的字符串。它用在GET请求附加data时，data往往是个对象，这时需要序列化和编码。因为空格会被编码成%20，但URL中空格应该是+，所以这里使用了replace方法处理。$.param函数内部调用了serialize函数。

至此Zepto.js中的Ajax模块已经完成。接下来仍有一个处理表单的模块，规模很小，不过封装了几个常用的工具函数。

## 表单模块
表单模块提供了三个方法，前两个用来序列化表单，最后一个用来提交表单。

``` JavaScript
;(function($){
  $.fn.serializeArray = function() {
    var name, type, result = [],
      add = function(value) {
        if (value.forEach) return value.forEach(add)
        result.push({ name: name, value: value })
      }
    if (this[0]) $.each(this[0].elements, function(_, field){
      type = field.type, name = field.name
      if (name && field.nodeName.toLowerCase() != 'fieldset' &&
        !field.disabled && type != 'submit' && type != 'reset' && type != 'button' && type != 'file' &&
        ((type != 'radio' && type != 'checkbox') || field.checked))
          add($(field).val())
    })
    return result
  }
 ```
$.fn.serializeArray函数将表单元素的值序列化为一个数组，就像下面一样。
``` JavaScript
//<form><input type="text" value="foo" name='foo"><input type="checkbox" value="bar" checked name="bar"></form>
$('form').serializeArray()//[{name:'foo',value:'foo'},{name:'bar',value:'bar'}]
```
函数内部需要注意的实现有两点：
* HTMLFormElement.elements属性返回一个HTMLFormControlsCollection对象，里面包含了一个表单元素下的所有除type='image'的表单控件元素。
* add函数中如果参数是数组，意味着则遍历数组在内部调用多次add函数。比如带有multiple属性的多选select就会被这样处理

``` JavaScript
$.fn.serialize = function(){
    var result = []
    this.serializeArray().forEach(function(elm){
      result.push(encodeURIComponent(elm.name) + '=' + encodeURIComponent(elm.value))
    })
    return result.join('&')
  }
```
另外一种形式的序列化，返回一个可以作为URL查询的字符串。之前的数组序列化中，序列化的每个结果作为对象存在。现在，将对象逐个解构。

``` JavaScript

  $.fn.submit = function(callback) {
    if (0 in arguments) this.bind('submit', callback)
    else if (this.length) {
      var event = $.Event('submit')
      this.eq(0).trigger(event)
      if (!event.isDefaultPrevented()) this.get(0).submit()
    }
    return this
  }
```
submit函数用来触发表单提交。

# 编码与程序开发感想
Zepto.js源码蕴含很多DOM操作技巧，不过这里我并不准备累述它们，我准备谈的是JavaScript程序语言的一些技巧和项目代码结构问题。
* JavaScript数组方法的应用。Array.prototype上的方法内部原理是使用数组的下标和length属性的，这意味着可以在类数组上调用
* JavaScript是弱类型语言。这种特性提供了很多便利，不过这导致了有时我们需要对传入函数的参数进行类型检查，对与强类型，参见[TypeScript](https://www.typescriptlang.org/ "TypeScript")
* 过多使用三元运算符和逻辑或、逻辑与运算符来进行分支控制会大大增加代码阅读难度，所以它们的大规模应用往往集中于生产环境中的代码压缩与混淆。不过有时还是很便利的，尤其是=需要返回值得时候。比如：
``` JavaScript
 // Generate the `after`, `prepend`, `before`, `append`,
 // `insertAfter`, `insertBefore`, `appendTo`, and `prependTo` methods.
 target = operatorIndex == 0 ? target.nextSibling :
                 operatorIndex == 1 ? target.firstChild :
                 operatorIndex == 2 ? target :
                 null
```
* 对分号使用的思考。JavaScript可以不使用分号，这得益于JavaScript中对语句是否结束的判断原则——大部分情况下如果一行的末尾不会与下一行的开头组合起来形成语句，那么不使用分号也会被认为语句已经结束了。但还是要注意特殊情况，这说明如果想省点心，稳健一点的话最好还是使用分号
* 思考哪些操作会是项目中的基本操作，将它们抽象出来。在Zepto中，大量的API都是对整个集合进行遍历的。这样一个编写良好的基础函数$.fn.each就很重要了。这会让程序更简洁而可靠
* 利用函数参数的个数的变化来实现不同的操作。这是一种API风格，他在jQuery和Zepto中广泛存在，比如$.fn.width方法在没有参数提供时表示获取值，而有参数时表示赋值。合理地使用这种风格可以让API更符合直觉和易用
* 对于不暴露给开发者的内部函数，没有必要对参数进行太多检测，毕竟参数的传入受控制较多。然而，暴露出来的API中应该对不同情形有着充分的考虑。
* 我们总是不喜欢使用try...catch语句，因为不喜欢“异常”，不过它在检测浏览器对某些特性否则支持时很有用
