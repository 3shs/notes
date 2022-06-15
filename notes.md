# 模板编译
## pase函数

里面维护了一个 ***stack***  然后调用 ***parseHTMl*** 并且传入 **template**模板 和 **start** 、**end** 、**chars** 、**comment** 四个方法

### parseHTML函数

里面也维护了一个 ***stack***  以及游标 ***index*** 循环解析模板 用 `template.indexOf('<')` 来匹配 匹配到了 ***为0*** 先后判断分别是 是否为 ***comment*** 、***条件注释*** 、***Doctype*** 、***End Tag*** 、***Start Tag***

#### 解析Start Tag

调用 **parseStartTag** 这个方法主要匹配标签的开始到结束以及里面所有的属性 `<div id="app">` 顺序依次是 `<div` 、`id="app"` 、`>` 通过 **advance** 方法去记录游标 ***index*** 的位置并且通过 **subString** 来截取字符串 再通过 **while** 循环匹配 **attr** 直到属性全部匹配完匹配到 `>` 结束 最终得到一个 ***match*** 例如：
```
  {
    attrs: [
      ['id=app', 'id', '=', 'app', start: 4, end: 13, ... ]
    ],
    end: 14,
    start: 0,
    tagName: "div",
    unarySlash: ""
  }
```
然后通过 **handleStartTag** 方法处理一下 ***match*** 得到例如： `attr = [{name: 'id', value: 'app', start: 5, end: 13}]` 这样的属性数组
```
  var l = match.attrs.length
  var attrs = new Array(l)
  for (var i = 0; i < l; i++) {
    var args = match.attrs[i];
    var value = args[3] || args[4] || args[5] || '';
    var shouldDecodeNewlines = tagName === 'a' && args[1] === 'href'
      ? options.shouldDecodeNewlinesForHref
      : options.shouldDecodeNewlines
    attrs[i] = {
      name: args[1],
      value: decodeAttr(value, shouldDecodeNewlines)
    }
    if (options.outputSourceRange) {
      attrs[i].start = args.start + args[0].match(/^\s*/).length
      attrs[i].end = args.end
    }
  }
```
然后 **push** 到 **parseHTML** 的 ***stack*** 
```
  [
    {
      attrs: [
        {
          name: 'id', 
          value: 'app', 
          start: 5,
          end: 13
        }
      ]
      end: 14,
      lowerCasedTag: "div",
      start: 0,
      tag: "div"
    }
  ]
```
然后再调用 **parseHTML** 传进来的 **start** 的方法创建 ***AST*** element 同时，在这个方法里解析 ***v-pre*** 、***v-for*** 、***v-once*** 最终得到 ***AST*** element 例如：
```
  {
    attrsList: [
      {
        end: 13,
        name: "id",
        start: 5,
        value: "app"
      }
    ]
    attrsMap: {id: 'app'}
    children: []
    end: 14
    parent: undefined
    rawAttrsMap: {
      id: {
        end: 13,
        name: "id",
        start: 5,
        value: "app"
      }
    }
    start: 0
    tag: "div"
    type: 1
  }
```
然后做个判断
```
  // 是否有root的根节点
  if (!root) {
    root = element
    {
      checkRootConstraints(root)
    }
  }
  // 是否为自闭合标签
  if (!unary) {
    currentParent = element
    stack.push(element)
  } else {
    closeElement(element)
  }
```
然后将 ***AST*** element **push** **parse** 的 ***stack*** 内

```
  [
    {
      attrsList: [
        {
          end: 13,
          name: "id",
          start: 5,
          value: "app"
        }
      ]
      attrsMap: {id: 'app'}
      children: []
      end: 14
      parent: undefined
      rawAttrsMap: {
        id: {
          end: 13,
          name: "id",
          start: 5,
          value: "app"
        }
      }
      start: 0
      tag: "div"
      type: 1
    }
  ]
```