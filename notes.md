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
#### 解析 v-for

通过 **processFor** 方法来处理 ***v-for*** 然后通过 **processFor** 里的 **getAndRemoteAttr** 方法 得到 ***v-for*** 属性的值 例如：`item in list` 然后通过循环得到 ***v-for*** 在 **attrList** 的位置 然后进行删除
```
function getAndRemoveAttr (
  el,
  name,
  removeFromMap
) {
  var val
  if ((val = el.attrsMap[name]) != null) {
    var list = el.attrsList
    for (var i = 0, l = list.length; i < l; i++) {
      if (list[i].name === name) {
        list.splice(i, 1)
        break
      }
    }
  }
  if (removeFromMap) {
    delete el.attrsMap[name]
  }
  return val
}
```
然后将得到的值 `item in list` 传给 **parseFor** 这个方法 里面通过正则匹配 得到 **inMatch** ：`["item in list", "item", "list"]` 最终得到一个 ***res*** 
```
{
  alias: "item",
  for: "list"
}
```
然后通过 **extend** 方法 将这个 ***res*** 合并到当前的 **element** 的 ***AST*** 上 最终得到合并好的 ***AST***
```
{
  alias: "item",
  ...
  for: "list",
  parent,
  rawAttrsMap: {
    v-for: {
      end: 87,
      name: "v-for",
      start: 67,
      value: "item in list"
    }
  }
  ...
  type: 1
}
```


#### 解析文本

从解析的当前位置到下一个 `<` 如果 `indexOf('<')` 大于0 说明之间是文本内容
##### 解析{{}}

通过 **parseText** 方法最终得到 这样的 res 其中有个 **parseFilters** 解析过滤器
```
{
  expression: "_s(txt)",
  tokens: [
    {
      @binding: "txt"
    }
  ]
}
```
然后推入此时 ***currentParent*** 的 ***children***
```
{
  end: 51,
  expression: "_s(txt)",
  start: 44,
  text: "{{txt}}",
  tokens: [
    {
      @binding: "txt"
    }
  ],
  type: 2
}
```
#### 解析End Tag

如匹配到闭合标签 随后将执行 **parseEndTag** 方法 从后向前循环 然后调用 **parseHTML** 传进来的 **end** 方法 然后通过 ``stack.length -= 1`` pop掉 **parse** 方法里的最新解析到的 ***AST*** 元素 最后调用 **closeElement** 方法 里面通过调用 **processElement** 方法 然后通过其里面的 **processKey**, **processRef**, **processSlotContent**, **processSlotOutlet**, **processComponent**, **processAttrs** 方法分别对元素进行 ***key*** , ***Ref***, ***slot***, ***componnet(:is)***, ***Attr*** 进行处理 然后将处理好的 ***element*** 进行
```
currentParent.children.push(element)
element.parent = currentParent
```
然后再将 **parseHTML** 的 ***stack*** 通过 `stack.length = pos` pop掉最新的解析标签

# AST生成render函数

调用 **generate** 方法 传入解析好的 ***AST*** 方法内部调用 **genElement** 方法 传入 ***AST*** 和 ***option*** 
```
  { attrs: {"id": "app"} }

  [
    {on: {'click': handleClick}, }
  ]

```
## genElement函数

通过各种判断来决定生成什么
```
function genElement (el, state) {
  if (el.parent) {
    el.pre = el.pre || el.parent.pre;
  }

  if (el.staticRoot && !el.staticProcessed) {
    return genStatic(el, state)
  } else if (el.once && !el.onceProcessed) {
    return genOnce(el, state)
  } else if (el.for && !el.forProcessed) {
    return genFor(el, state)
  } else if (el.if && !el.ifProcessed) {
    return genIf(el, state)
  } else if (el.tag === 'template' && !el.slotTarget && !state.pre) {
    return genChildren(el, state) || 'void 0'
  } else if (el.tag === 'slot') {
    return genSlot(el, state)
  } else {
    // component or element
    var code;
    if (el.component) {
      code = genComponent(el.component, el, state);
    } else {
      var data;
      if (!el.plain || (el.pre && state.maybeComponent(el))) {
        data = genData$2(el, state);
      }

      var children = el.inlineTemplate ? null : genChildren(el, state, true);
      code = "_c('" + (el.tag) + "'" + (data ? ("," + data) : '') + (children ? ("," + children) : '') + ")";
    }
    // module transforms
    for (var i = 0; i < state.transforms.length; i++) {
      code = state.transforms[i](el, code);
    }
    return code
  }
}
```
在 **component or element** 调用 **genData$2**方法 处理标签上的属性 得到 ***data***
```
"{
  attrs: {
    "id": "app"
  }
}"
```
然后调用 **genChildren** 生成子元素 这里面有个 **genNode** 方法 genNode方法针对不同的元素去调用生成不同元素的方法
```
function genChildren (
  el,
  state,
  checkSkip,
  altGenElement,
  altGenNode
) {
  var children = el.children;
  if (children.length) {
    var el$1 = children[0];
    // optimize single v-for
    if (children.length === 1 &&
      el$1.for &&
      el$1.tag !== 'template' &&
      el$1.tag !== 'slot'
    ) {
      var normalizationType = checkSkip
        ? state.maybeComponent(el$1) ? ",1" : ",0"
        : "";
      return ("" + ((altGenElement || genElement)(el$1, state)) + normalizationType)
    }
    var normalizationType$1 = checkSkip
      ? getNormalizationType(children, state.maybeComponent)
      : 0;
    var gen = altGenNode || genNode;
    return ("[" + (children.map(function (c) { return gen(c, state); }).join(',')) + "]" + (normalizationType$1 ? ("," + normalizationType$1) : ''))
  }
}
```
```
function genNode (node, state) {
  if (node.type === 1) {
    return genElement(node, state)
  } else if (node.type === 3 && node.isComment) {
    return genComment(node)
  } else {
    return genText(node)
  }
}
```
这里基本逻辑是通过不断的循环元素 父元素 -> 子元素 -> 子子元素 -> 直到子元素循环完毕 结束 得到一个 ***code***
```
code = "_c(
  'div', 
  {attrs: {"id": "app"}},
  [
    _c(
      'h1',
      {on: {"click": "handleClick"}},
      [
        _v(_s(txt))
      ]
    ),
    _v(""),
    _l(
      (list), 
      function (item) {return _c('div', [_v(_s(item.name))])}
    )
  ],
  2
)"
```

