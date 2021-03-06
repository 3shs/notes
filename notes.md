# 1. 模板编译

## pase函数

里面维护了一个 ***stack***  然后调用 ***parseHTMl*** 并且传入 **template**模板 和 **start** 、**end** 、**chars** 、**comment** 四个方法

### parseHTML函数

里面也维护了一个 ***stack***  以及游标 ***index*** 循环解析模板 用 `template.indexOf('<')` 来匹配 匹配到了 ***为0*** 先后判断分别是 是否为 ***comment*** 、***条件注释*** 、***Doctype*** 、***End Tag*** 、***Start Tag***

#### 解析Start Tag

调用 **parseStartTag** 这个方法主要匹配标签的开始到结束以及里面所有的属性 `<div id="app">` 顺序依次是 `<div` 、`id="app"` 、`>` 通过 **advance** 方法去记录游标 ***index*** 的位置并且通过 **subString** 来截取字符串 再通过 **while** 循环匹配 **attr** 直到属性全部匹配完匹配到 `>` 结束 最终得到一个 ***match*** 例如：

```javascript
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

```javascript
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

```javascript
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

```javascript
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

```javascript
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

```javascript
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

```javascript
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

```javascript
{
  alias: "item",
  for: "list"
}
```
然后通过 **extend** 方法 将这个 ***res*** 合并到当前的 **element** 的 ***AST*** 上 最终得到合并好的 ***AST***

```javascript
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

```javascript
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
```javascript
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

如匹配到闭合标签 随后将执行 **parseEndTag** 方法 从后向前循环 然后调用 **parseHTML** 传进来的 **end** 方法 然后通过 `stack.length -= 1` pop掉 **parse** 方法里的最新解析到的 ***AST*** 元素 最后调用 **closeElement** 方法 里面通过调用 **processElement** 方法 然后通过其里面的 **processKey**, **processRef**, **processSlotContent**, **processSlotOutlet**, **processComponent**, **processAttrs** 方法分别对元素进行 ***key*** , ***Ref***, ***slot***, ***componnet(:is)***, ***Attr*** 进行处理 然后将处理好的 ***element*** 进行

```javascript
currentParent.children.push(element)
element.parent = currentParent
```
然后再将 **parseHTML** 的 ***stack*** 通过 `stack.length = pos` pop掉最新的解析标签

# 2. AST生成render函数

调用 **generate** 方法 传入解析好的 ***AST*** 方法内部调用 **genElement** 方法 传入 ***AST*** 和 ***option*** 

```javascript
  { attrs: {"id": "app"} }

  [
    {on: {'click': handleClick}, }
  ]

```
## genElement函数

通过各种判断来决定生成什么

```javascript
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

```javascript
"{
  attrs: {
    "id": "app"
  }
}"
```
然后调用 **genChildren** 生成子元素 这里面有个 **genNode** 方法 genNode方法针对不同的元素去调用生成不同元素的方法

```javascript
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
```javascript
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
这里基本逻辑是通过不断的循环元素 父元素 -> 子元素 -> 子子元素 -> 直到所有子元素循环完毕 结束 得到一个 ***code***

```javascript
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
然后利用code拼接一个 **with** 函数的字符串 再通过 `new Function(code)` 得到一个 ***render*** 函数

### 解析v-if

* 在前面模板解析的时候 对于标签上的属性都做了解析 当然也包括 **v-if** 解析出来基本格式：

```javascript
if: "isShow",
ifConditions: [
  {
    exp: 'isShow',
    block: {
      // 自身的所有AST属性
    }
  },
]
```
* 然后在 `genElement` 方法里对 **AST** 节点上 是否存在 **el.if** 属性进行判断 如果有 则调用 `genIf` 方法 该方法返回 `genIfConditions` 的结果

```javascript
function genIf (
  el,
  state,
  altGen,
  altEmpty
) {
  el.ifProcessed = true; // avoid recursion
  return genIfConditions(el.ifConditions.slice(), state, altGen, altEmpty)
}
```
* `genIfConditions` 内部主要
 * 首先判断 **AST** 上面的 **ifConditions** 是否为空 如果为空 直接返回 空节点 或者 '_e' 利用 `_e` 方法创建的空节点
 * 如果不为空则继续往下走 拿到 **condition** 头部的数据对象
 * 然后判断 该头部数据是有值 如果有值 则返回 一段经过处理好的 三元字符串 大致如下
 * 最后在调用生成好的 render 函数的时候 根据 **isShow** 来判断 是否生成对应的元素还是空节点

 ```javascript
 isShow ? _c('div') : _e()
 ```

```javascript
function genIfConditions (
  conditions,
  state,
  altGen,
  altEmpty
) {
  if (!conditions.length) {
    return altEmpty || '_e()'
  }

  var condition = conditions.shift();
  if (condition.exp) {
    // 这里 genTernaryExp 方法 目的就是返回 解析好的 元素 code
    return ("(" + (condition.exp) + ")?" + (genTernaryExp(condition.block)) + ":" + (genIfConditions(conditions, state, altGen, altEmpty)))
  } else {
    return ("" + (genTernaryExp(condition.block)))
  }

  // v-if with v-once should generate code like (a)?_m(0):_m(1)
  function genTernaryExp (el) {
    return altGen
      ? altGen(el, state)
      : el.once
        ? genOnce(el, state)
        : genElement(el, state)
  }
}
```

### 解析v-show

在解析模板的阶段会对 `v-show` 这个属性转化为 指令的形式

```javascript
// AST 比如这样子
{
  attrsList,
  attrsMap,
  children: [],
  directives: [
    {
      arg: null,
      end: 39,
      isDynamicArg: false,
      modifiers: undefined,
      name: "show",
      rawName: "v-show",
      start: 24,
      value: "isShow",
    }
  ],
  end: 46,
  hasBindings: true,
  parent: {type: 1, tag: 'div', attrsList: Array(1), attrsMap: {…}, rawAttrsMap: {…}, …}
  plain: false,
  rawAttrsMap,
  start: 19,
  tag: "div",
  type: 1,
}
```
然后在将 **AST** 转为 render 的 **code** 的时候 通过 `genData$2` 方法里的 `genDirectives` 方法 拼接出 Vue 想要的 字符串 放到 **data** 里的占位里

```javascript
// 解析成这样
"{directives:[{name:"show",rawName:"v-show",value:(isShow),expression:"isShow"}]}"

```
最后在创建真实Dom的时候 调用 `invokeCreateHooks` 方法 该方法里面通过 `updateDirectives` 来更新指令 就是 Vue 内部自定义的 `v-show` 指令 通过 `bind` 方法来处理 最后 通过之前的 **value** 来决定是否 `display:none`


# 3. mount阶段

在挂载阶段之前 先 **callHook** 一下 **beforeMount** 的函数 然后给 **updateComponent** 赋值

```javascript
updateComponent = function () {
  vm._update(vm._render(), hydrating)
}
```
```javascript
vnode = render.call(vm._renderProxy, vm.$createElement);
```
然后进入 **Watcher** 将 **updateComponent** 传给 **Watcher** Watcher 经过一系列初始化 然后调用 **updateComponent** 调用 **_render** 函数 然后调用 利用 ***code*** 生成好的 **render** 函数 并且传入 **createElement** 方法 通过 读取 ***_c*** (被 proxy)  调用 **createElement** 然后 **createElement** 调用 **_createElement** 生成完整的 ***vnode***

```javascript
function VNode(
  tag,
  data,
  children,
  text,
  elm,
  context,
  componentOptions,
  asyncFactory
  )
{
  this.tag = tag; //标签名
  this.data = data; // 标签上的 属性以及值
  this.children = children; // 子元素
  this.text = text; // 文本
  this.elm = elm; // 真实的Dom元素
  this.ns = undefined;
  this.context = context;
  this.fnContext = undefined;
  this.fnOptions = undefined;
  this.fnScopeId = undefined;
  this.key = data && data.key;
  this.componentOptions = componentOptions;
  this.componentInstance = undefined;
  this.parent = undefined;
  this.raw = false;
  this.isStatic = false;
  this.isRootInsert = true;
  this.isComment = false;
  this.isCloned = false;
  this.isOnce = false;
  this.asyncFactory = asyncFactory;
  this.asyncMeta = undefined;
  this.isAsyncPlaceholder = false;
}
```
然后传入 **vm._render** 生成好的 ***vnode*** 给 **vm._update** 这里分为两种情况 如果之前已经有 ***vm._vnode*** 那就进行 **updates** 否则就是属于第一次挂载

```javascript
Vue.prototype._update = function (vnode, hydrating) {
  var vm = this;
  var prevEl = vm.$el;
  var prevVnode = vm._vnode;
  var restoreActiveInstance = setActiveInstance(vm);
  vm._vnode = vnode;
  // Vue.prototype.__patch__ is injected in entry points
  // based on the rendering backend used.
  if (!prevVnode) {
    // initial render
    vm.$el = vm.__patch__(vm.$el, vnode, hydrating, false /* removeOnly */);
  } else {
    // updates
    vm.$el = vm.__patch__(prevVnode, vnode);
  }
  restoreActiveInstance();
  // update __vue__ reference
  if (prevEl) {
    prevEl.__vue__ = null;
  }
  if (vm.$el) {
    vm.$el.__vue__ = vm;
  }
  // if parent is an HOC, update its $el as well
  if (vm.$vnode && vm.$parent && vm.$vnode === vm.$parent._vnode) {
    vm.$parent.$el = vm.$el;
  }
  // updated hook is called by the scheduler to ensure that children are
  // updated in a parent's updated hook.
}
```
## 第一次挂载 （initial render）

第一次挂载 进行 **patch** 的时候 传递的 ***oldVnode*** 就是真实的Dom **vm.$el** ***vnode*** 就是通过 **_render** 方法生成的 ***vnode*** 

```javascript
function patch (oldVnode, vnode, hydrating, removeOnly) {
  if (isUndef(vnode)) {
    if (isDef(oldVnode)) { invokeDestroyHook(oldVnode); }
    return
  }

  var isInitialPatch = false;
  var insertedVnodeQueue = [];

  if (isUndef(oldVnode)) {
    // empty mount (likely as component), create new root element
    isInitialPatch = true;
    createElm(vnode, insertedVnodeQueue);
  } else {
    var isRealElement = isDef(oldVnode.nodeType);
    if (!isRealElement && sameVnode(oldVnode, vnode)) {
      // patch existing root node
      // 如果 oldvnode 不是真实的 dom 以及 oldVnode 和 vnode 相似 说明已经存在根节点 那就对这两份数据 进行 patchVnode
      patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly);
    } else {
      // oldVnode 是真实的 Dom
      if (isRealElement) {
        // mounting to a real element
        // check if this is server-rendered content and if we can perform
        // a successful hydration.
        if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
          oldVnode.removeAttribute(SSR_ATTR);
          hydrating = true;
        }
        if (isTrue(hydrating)) {
          if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
            invokeInsertHook(vnode, insertedVnodeQueue, true);
            return oldVnode
          } else {
            warn(
              'The client-side rendered virtual DOM tree is not matching ' +
              'server-rendered content. This is likely caused by incorrect ' +
              'HTML markup, for example nesting block-level elements inside ' +
              '<p>, or missing <tbody>. Bailing hydration and performing ' +
              'full client-side render.'
            );
          }
        }
        // either not server-rendered, or hydration failed.
        // create an empty node and replace it
        // 利用 emptyNodeAt 方法 将真实dom 生成一份 虚拟Dom 
        oldVnode = emptyNodeAt(oldVnode);
      }

      // replacing existing element
      // 这个真实的dom节点是要传给 createElm 方法的
      var oldElm = oldVnode.elm;
      // 获取 root 节点的 父节点 其实就是 #app 的父节点 body 第一次
      var parentElm = nodeOps.parentNode(oldElm);

      // create new node
      // 创建真实的Dom
      createElm(
        vnode,
        insertedVnodeQueue,
        // extremely rare edge case: do not insert if old element is in a
        // leaving transition. Only happens when combining transition +
        // keep-alive + HOCs. (#4590)
        oldElm._leaveCb ? null : parentElm,
        nodeOps.nextSibling(oldElm)
      );

      // update parent placeholder node element, recursively
      if (isDef(vnode.parent)) {
        var ancestor = vnode.parent;
        var patchable = isPatchable(vnode);
        while (ancestor) {
          for (var i = 0; i < cbs.destroy.length; ++i) {
            cbs.destroy[i](ancestor);
          }
          ancestor.elm = vnode.elm;
          if (patchable) {
            for (var i$1 = 0; i$1 < cbs.create.length; ++i$1) {
              cbs.create[i$1](emptyNode, ancestor);
            }
            // #6513
            // invoke insert hooks that may have been merged by create hooks.
            // e.g. for directives that uses the "inserted" hook.
            var insert = ancestor.data.hook.insert;
            if (insert.merged) {
              // start at index 1 to avoid re-invoking component mounted hook
              for (var i$2 = 1; i$2 < insert.fns.length; i$2++) {
                insert.fns[i$2]();
              }
            }
          } else {
            registerRef(ancestor);
          }
          ancestor = ancestor.parent;
        }
      }

      // destroy old node
      if (isDef(parentElm)) {
        removeVnodes(parentElm, [oldVnode], 0, 0);
      } else if (isDef(oldVnode.tag)) {
        invokeDestroyHook(oldVnode);
      }
    }
  }

  invokeInsertHook(vnode, insertedVnodeQueue, isInitialPatch);
  return vnode.elm
}
```
### createElm

这个方法主要是将 ***vnode*** 转为 真实的 Dom 结构<br>
createElm 主要接收 虚拟 Dom 对象 和 当前虚拟 Dom 父节点 以及 虚拟Dom 上的 Ele属性（真实的Dom） 就是形参中的 refElm<br>
取出 vnode 上面的 ***data*** 属性 和 ***children*** 属性 以及 ***tag***<br>
然后 利用 tag 创建 真实的 Dom 挂在 ***vnode*** 的 ***elm*** 属性上<br>
然后调用 **createChildren** 在 createChildren 里面继续调用 **createElm** 方法创建元素<br>
在 **createChildren** 的时候 会先调用 **checkDuplicateKeys** 方法 查看是否有相同的key值<br>
基本逻辑就是递归循环调用 将 ***vnode*** 一一对应的节点 转成 真实的 Dom 结构<br>
createElm 方法 主要就是创建 元素节点 注释节点 和 文本节点 判断依次顺序为<br>

``` javascript
if (isDef(tag)) { 

  //do something... 

} else if (isTrue(vnode.isComment)) { 

  //do someting... 
  //创建注释节点 插入对应位置
} else { 
  //文本节点
  //创建文本节点 插入对应位置

}
```
 
```javascript
function createElm (
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // This vnode was used in a previous render!
    // now it's used as a new node, overwriting its elm would cause
    // potential patch errors down the road when it's used as an insertion
    // reference node. Instead, we clone the node on-demand before creating
    // associated DOM element for it.
    vnode = ownerArray[index] = cloneVNode(vnode);
  }

  vnode.isRootInsert = !nested; // for transition enter check
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }

  var data = vnode.data;
  var children = vnode.children;
  var tag = vnode.tag;
  // 如果有标签 就走创建 元素节点
  if (isDef(tag)) {
    {
      if (data && data.pre) {
        creatingElmInVPre++;
      }
      if (isUnknownElement$$1(vnode, creatingElmInVPre)) {
        warn(
          'Unknown custom element: <' + tag + '> - did you ' +
          'register the component correctly? For recursive components, ' +
          'make sure to provide the "name" option.',
          vnode.context
        );
      }
    }

    vnode.elm = vnode.ns
      ? nodeOps.createElementNS(vnode.ns, tag)
      : nodeOps.createElement(tag, vnode);
    setScope(vnode);

    /* istanbul ignore if */
    {
      createChildren(vnode, children, insertedVnodeQueue);
      if (isDef(data)) {
        invokeCreateHooks(vnode, insertedVnodeQueue);
      }
      insert(parentElm, vnode.elm, refElm);
    }

    if (data && data.pre) {
      creatingElmInVPre--;
    }
  } else if (isTrue(vnode.isComment)) {
    // 如果 vnode isComment属性为 标记为 注释 则就创建注释节点
    vnode.elm = nodeOps.createComment(vnode.text);
    insert(parentElm, vnode.elm, refElm);
  } else {
    // 否则就是创建文本节点
    vnode.elm = nodeOps.createTextNode(vnode.text);
    insert(parentElm, vnode.elm, refElm);
  }
}



function createChildren (vnode, children, insertedVnodeQueue) {
  if (Array.isArray(children)) {
    {
      checkDuplicateKeys(children);
    }
    for (var i = 0; i < children.length; ++i) {
      createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i);
    }
  } else if (isPrimitive(vnode.text)) {
    nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)));
  }
}
```
#### invokeCreateHooks

这个方法主要用来处理 ***vnode*** 上面 ***data*** 属性 cbs.create 里面主要存放了 一些 update 标签上的属性的方法
同时，标签上写的 方法 也是在这里 通过 updateDOMListeners 来注册的

```javascript
cbs.create = [
  updateAttrs(oldVnode, vnode),
  updateClass(oldVnode, vnode),
  updateDOMListeners(oldVnode, vnode),
  updateDOMProps(oldVnode, vnode),
  updateStyle(oldVnode, vnode),
  _enter(_, vnode),
  create(_, vnode),
  updateDirectives(oldVnode, vnode)
]



function invokeCreateHooks (vnode, insertedVnodeQueue) {
  for (var i$1 = 0; i$1 < cbs.create.length; ++i$1) {
    cbs.create[i$1](emptyNode, vnode);
  }
  i = vnode.data.hook; // Reuse variable
  if (isDef(i)) {
    if (isDef(i.create)) { i.create(emptyNode, vnode); }
    if (isDef(i.insert)) { insertedVnodeQueue.push(vnode); }
  }
}
```
##### updateDOMListeners

基本原理还是利用 **addEventListener** 来注册事件<br>
在这个方法里面拿到 **oldVnode** 和 **vnode** 上面 data 上面的 on 属性 然后传入 **updateListeners** 这个方法<br>
在 updateListeners 循环遍历 on 属性上面的事件 拿到 新事件 和 旧事件<br> 
然后在 **invoker** 方法下 挂在一个静态属性 fns 并把 注册的事件赋给它<br>
通过 **add** 方法 注册事件<br>
在 **add** 方法里 通过 **addEventListener** 注册事件<br>
最后移除掉 oldVnode 上面的 事件<br>

# 4. update 阶段

## patchVnode

该方法主要是对node节点进行 **patch** 进行打补丁<br>
基本逻辑梳理<br>
如果 **oldVnode** 和 **vnode** 相等 则直接退出<br>
如果 **oldVnode** 和 **vnode** 是静态节点 则直接退出 ***静态节点指的的类似于 `<div>我是静态的</div>` 里面没有任何变量的值***<br>
* 如果 **vnode** 里面没有 **text** 属性的值 则说明是元素节点<br>
  * 判断 **oldVnode** 和 **vnode** 是否都有子节点 如果有
    * 判断 **oldVnode** 下的子节点 和 **vnode** 下的子节点 是否相同 如果不同 则通过 **updateChildren** 方法去更新子节点（***Diff***）
    * 判断 如果只有 **vnode** 下有子节点 如果 **oldVnode** 有文本 清空Dom中的文本 最后将 **vnode** 的子节点插入到Dom中
    * 判断 如果只有 **oldVnode** 下有子节点 则直接清空Dom中的子节点
    * 判断 如果 **oldVnode** 下有文本 则直接清空Dom中的文本
* 如果 **vnode** 里面 有 **text** 属性值 则说明是文本节点
  * 判断 **vnode** 和 **oldVnode** 的 **text** 是否一样 如果不一样  则用 **vnode** 里的文本值直接替换真实Dom节点中的文本内容

```javascript
function patchVnode (
  oldVnode,
  vnode,
  insertedVnodeQueue,
  ownerArray,
  index,
  removeOnly
) {
  if (oldVnode === vnode) {
    return
  }

  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // clone reused vnode
    vnode = ownerArray[index] = cloneVNode(vnode);
  }

  var elm = vnode.elm = oldVnode.elm;

  if (isTrue(oldVnode.isAsyncPlaceholder)) {
    if (isDef(vnode.asyncFactory.resolved)) {
      hydrate(oldVnode.elm, vnode, insertedVnodeQueue);
    } else {
      vnode.isAsyncPlaceholder = true;
    }
    return
  }

  // reuse element for static trees.
  // note we only do this if the vnode is cloned -
  // if the new node is not cloned it means the render functions have been
  // reset by the hot-reload-api and we need to do a proper re-render.
  if (isTrue(vnode.isStatic) &&
    isTrue(oldVnode.isStatic) &&
    vnode.key === oldVnode.key &&
    (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
  ) {
    vnode.componentInstance = oldVnode.componentInstance;
    return
  }

  var i;
  var data = vnode.data;
  if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
    i(oldVnode, vnode);
  }

  var oldCh = oldVnode.children;
  var ch = vnode.children;
  if (isDef(data) && isPatchable(vnode)) {
    for (i = 0; i < cbs.update.length; ++i) { cbs.update[i](oldVnode, vnode); }
    if (isDef(i = data.hook) && isDef(i = i.update)) { i(oldVnode, vnode); }
  }
  if (isUndef(vnode.text)) {
    if (isDef(oldCh) && isDef(ch)) {
      if (oldCh !== ch) { updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly); }
    } else if (isDef(ch)) {
      {
        checkDuplicateKeys(ch);
      }
      if (isDef(oldVnode.text)) { nodeOps.setTextContent(elm, ''); }
      addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue);
    } else if (isDef(oldCh)) {
      removeVnodes(elm, oldCh, 0, oldCh.length - 1);
    } else if (isDef(oldVnode.text)) {
      nodeOps.setTextContent(elm, '');
    }
  } else if (oldVnode.text !== vnode.text) {
    nodeOps.setTextContent(elm, vnode.text);
  }
  if (isDef(data)) {
    if (isDef(i = data.hook) && isDef(i = i.postpatch)) { i(oldVnode, vnode); }
  }
}
```

## Diff

这个阶段主要是对子节点打 **patch** 通过 ***Diff*** 算法 来对比新旧 **vnode** 的区别 然后更新 旧的 **vnode** 完成数据更新<br>
这里的 ***Diff*** 主要通过 两边向中间进行对比<br>
定义 ***oldChildren*** 开始的下标 和 ***newChildren*** 开始的下标 分别是 **oldStartIdx** **newStartIdx**<br>
定义 ***oldChildren*** 结束的下标 以及 开始元素 和 结束元素 分别是 **oldEndIdx** **oldStartVnode** **oldEndVnode**<br>
定义 ***newChildren*** 结束的下标 以及 开始元素 和 结束元素 分别是 **newEndIdx** **newStartVnode** **newEndVnode**<br>
* 先从起始位置开始对比 如果 ***旧节点的起始元素*** 和 ***新节点的起始元素*** 相同 则进行 **patchVnode** 并且将 ***oldStartIdx*** 和 ***newStartIdx*** 增加一位 利用更新过的下标取得下一个元素并且赋值为 ***旧节点的起始元素*** ***新节点的起始元素*** 然后重新走入循环进行对比
* 如果起始位置不同 则继续判断 ***oldChildren*** 和 ***newChildren*** 结束元素 是否相同 如果 ***旧节点的结束元素*** 和 ***新节点的结束元素*** 相同 则同样进行 **patchVnode** 并且将 ***oldEndIdx*** 和 ***newEndIdx*** 减去一位 利用更新过的下标取得上一个元素并且赋值为 ***旧节点的结束元素*** ***新节点的结束元素*** 然后重新走入循环进行对比
* 如果新旧节点起始元素和结束元素都不相同的话 则用 ***旧节点的起始元素*** 和 ***新节点的结束元素*** 进行对比 如果相同的话 则说明元素进行了移动 旧节点的这个元素已经移动到了右边 同样 先进行 **patchVnode** 然后获取旧节点结束元素的下个兄弟节点作为参照 将元素插入Dom中（参照之前） 然后将 ***oldStartIdx*** 增加一位 ***newEndIdx***减去一位 因为是旧节点的起始元素和新节点的结束元素做对比 所以一个向后 一个向前 同样利用更新过的下标取得 ***旧节点的起始元素*** 和 ***新节点的结束元素*** 然后重新走入循环进行对比
* 如果旧节点的起始元素和新节点的结束元素也不相同的话 则用 ***旧节点的结束元素*** 和 ***新节点的开始元素*** 进行对比 如果相同的话 同样说明进行移动 旧节点的这个元素已经移动到了左边 同样 先进行 **patchVnode** 然后将元素插入到 ***新节点起始元素*** 之前 然后将 ***oldEndIdx*** 减去一位 ***newStartIdx*** 增加一位 一个向前 一个向后 同样利用更新过的下标取得 ***旧节点的结束元素*** 和 ***新节点的开始元素*** 然后重新走入循环进行对比
* 如果以上对比方式都没找到
  * 则取出 ***oldChildren*** 元素 利用 **createKeyToOldIdx** 方法 创建出一个 ***map*** 以未处理的元素上面的key值作为属性 以未处理的元素对应的下标作为属性值
  * 然后判断 ***newStartVnode*** 上的key值是否有 如果有 则取出这个key值在 ***oldChildren*** 上对应的 index 也就是 ***idxInOld*** 然后根据这个 index找到 ***oldChildren*** 对应的元素 然后对比 ***newStartVnode*** 和 找出的这个元素是否相同 如果相同则 进行补丁 然后移动到 ***oldStartVnode*** 之前
  * 如果在 ***newStartVnode*** 找不到对应的key值 这说明这个元素是个新元素 然后调用 **createEle** 方法 创建Dom 然后插入 ***oldStartVnode*** 之前
  * 为啥走这里判断都是插入 ***oldStartVnode*** 之前呢？ 走这里的逻辑 就是拿 每次循环过后更新好的 ***newStartVnode*** 的 key 在 ***oldChildren*** 找 如果 ***oldChildren*** 上找到了 这个 key 则取得下标 则与之对比进行patch和移动 否则 就认为这个 ***newStartVnode*** 是个新元素 将其插入 ***oldStartVnode*** 之前
* 如果 **oldStartIdx > oldEndIdx** 说明 ***oldChildren*** 已经比 ***newChildren*** 先循环完了 说明 ***oldChildren*** 的长度要小于 ***newChildren*** 说明 ***newChildren*** 剩下的元素都是要新增的
* 如果 **newStartIdx > newEndIdx** 说明 ***newChildren*** 已经比 ***oldChildren*** 先循环完了 说明 ***newChildren*** 的长度要小于
***oldChildren*** 说明 ***oldChildren*** 剩下的元素都是要移除的

```javascript
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue, removeOnly) {
  var oldStartIdx = 0;
  var newStartIdx = 0;
  var oldEndIdx = oldCh.length - 1;
  var oldStartVnode = oldCh[0];
  var oldEndVnode = oldCh[oldEndIdx];
  var newEndIdx = newCh.length - 1;
  var newStartVnode = newCh[0];
  var newEndVnode = newCh[newEndIdx];
  var oldKeyToIdx, idxInOld, vnodeToMove, refElm;

  // removeOnly is a special flag used only by <transition-group>
  // to ensure removed elements stay in correct relative positions
  // during leaving transitions
  var canMove = !removeOnly;

  {
    checkDuplicateKeys(newCh);
  }

  while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
    if (isUndef(oldStartVnode)) {
      oldStartVnode = oldCh[++oldStartIdx]; // Vnode has been moved left
    } else if (isUndef(oldEndVnode)) {
      oldEndVnode = oldCh[--oldEndIdx];
    } else if (sameVnode(oldStartVnode, newStartVnode)) {
      patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx);
      oldStartVnode = oldCh[++oldStartIdx];
      newStartVnode = newCh[++newStartIdx];
    } else if (sameVnode(oldEndVnode, newEndVnode)) {
      patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx);
      oldEndVnode = oldCh[--oldEndIdx];
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
      patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx);
      canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm));
      oldStartVnode = oldCh[++oldStartIdx];
      newEndVnode = newCh[--newEndIdx];
    } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
      patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx);
      canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm);
      oldEndVnode = oldCh[--oldEndIdx];
      newStartVnode = newCh[++newStartIdx];
    } else {
      if (isUndef(oldKeyToIdx)) { oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx); }
      idxInOld = isDef(newStartVnode.key)
        ? oldKeyToIdx[newStartVnode.key]
        : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx);
      if (isUndef(idxInOld)) { // New element
        createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx);
      } else {
        vnodeToMove = oldCh[idxInOld];
        if (sameVnode(vnodeToMove, newStartVnode)) {
          patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx);
          oldCh[idxInOld] = undefined;
          canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm);
        } else {
          // same key but different element. treat as new element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx);
        }
      }
      newStartVnode = newCh[++newStartIdx];
    }
  }
  if (oldStartIdx > oldEndIdx) {
    refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm;
    addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue);
  } else if (newStartIdx > newEndIdx) {
    removeVnodes(parentElm, oldCh, oldStartIdx, oldEndIdx);
  }
}


function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}
```

# 5. nextTick

Vue里的Dom更新是异步 数据改变了并不会立即重新渲染 就是不会立即调用 生成好的 **render** 函数 这样的好处是 如果一个 ***Watcher***被多次触发 只会被推入到事件队列中一次


* 当数据更新的时候 会调用 **dep.notify** 去进行通知
* 然后将依赖拷贝一份出来 ***subs*** 其实里面维护的就是 ***Watcher***
* 通过循环依赖 调用每个 ***Watcher*** 上面的 **update** 方法
* **update** 主要进行几个判断 如果是惰性的 修改 dirty 的值 如果是同步的话 直接调用 ***Watcher*** 上的 **run** 方法重新渲染 否则就 执行 **queueWatcher** 方法
* **queueWatcher** 里面维护 ***Watcher*** 的队列 **queue**
* 判断如果队列不在冲洗中（队列不在执行中） 里面调用 **nextTick** 方法 传入一个默认的 回调函数 **flushSchedulerQueue**
* **nextTick** 方法 将传入的回调函数 推入到回调队列中 ***callbacks***
* 判断如果队列不在冲洗中（队列不在执行中） 则调用 **timerFunc** 方法
* ***timerFunc*** 方法 根据浏览器不同的能力 赋予不同的方法 最终都执行 **flushCallbacks** 方法 这个方法就是用来执行 回调队列里的 回调函数的
* **flushCallbacks** 将更新的回调函数 拷贝一份 然后循环调用每个回调函数 更新就是调用 **flushSchedulerQueue** 方法 因为之前就是 将这个回调函数 推入 回调队列中
* **flushSchedulerQueue** 方法 目的就是调用 ***Watcher*** 上面的 **run** 方法 去重新调用 **render** 函数 重新对页面进行渲染

所以就不难理解 ***nextTick*** 的原理了 调用 nextTick 将其回调函数直接 push 回调队列里 然后 通过 **flushCallbacks** 方法 对回调队列里的回调函数依次循环调用的时候 就会执行到 我们手动调用的nextTick传入的 回调函数 并且这个 回调函数在 Dom异步更新之后 所以我们可以立即获取更新好的Dom了 形式大致如下

```javascript
callbacks = [
  flushSchedulerQueue, // 渲染页面的调度函数
  () => {
    console.log(vm.$el.innerHTML)
  }, // 我们手动nextTick传入的回调函数
]
```

```javascript
Dep.prototype.notify = function notify () {
  // stabilize the subscriber list first
  var subs = this.subs.slice();
  if (!config.async) {
    // subs aren't sorted in scheduler if not running async
    // we need to sort them now to make sure they fire in correct
    // order
    subs.sort(function (a, b) { return a.id - b.id; });
  }
  for (var i = 0, l = subs.length; i < l; i++) {
    subs[i].update();
  }
};
```
```javascript
Watcher.prototype.update = function update () {
  /* istanbul ignore else */
  if (this.lazy) {
    this.dirty = true;
  } else if (this.sync) {
    this.run();
  } else {
    queueWatcher(this);
  }
};
```
```javascript
function queueWatcher (watcher) {
  var id = watcher.id;
  if (has[id] == null) {
    has[id] = true;
    if (!flushing) {
      queue.push(watcher);
    } else {
      // if already flushing, splice the watcher based on its id
      // if already past its id, it will be run next immediately.
      var i = queue.length - 1;
      while (i > index && queue[i].id > watcher.id) {
        i--;
      }
      queue.splice(i + 1, 0, watcher);
    }
    // queue the flush
    if (!waiting) {
      waiting = true;

      if (!config.async) {
        flushSchedulerQueue();
        return
      }
      nextTick(flushSchedulerQueue);
    }
  }
}
```
```javascript
function nextTick (cb, ctx) {
  var _resolve;
  callbacks.push(function () {
    if (cb) {
      try {
        cb.call(ctx);
      } catch (e) {
        handleError(e, ctx, 'nextTick');
      }
    } else if (_resolve) {
      _resolve(ctx);
    }
  });
  if (!pending) {
    pending = true;
    timerFunc();
  }
  // $flow-disable-line
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(function (resolve) {
      _resolve = resolve;
    })
  }
}
```
```javascript
if (typeof Promise !== 'undefined' && isNative(Promise)) {
  var p = Promise.resolve();
  timerFunc = function () {
    p.then(flushCallbacks);
    
    if (isIOS) { setTimeout(noop); }
  };
  isUsingMicroTask = true;
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  
  var counter = 1;
  var observer = new MutationObserver(flushCallbacks);
  var textNode = document.createTextNode(String(counter));
  observer.observe(textNode, {
    characterData: true
  });
  timerFunc = function () {
    counter = (counter + 1) % 2;
    textNode.data = String(counter);
  };
  isUsingMicroTask = true;
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  
  timerFunc = function () {
    setImmediate(flushCallbacks);
  };
} else {

  timerFunc = function () {
    setTimeout(flushCallbacks, 0);
  };
}
```
```javascript
function flushCallbacks () {
  pending = false;
  var copies = callbacks.slice(0);
  callbacks.length = 0;
  for (var i = 0; i < copies.length; i++) {
    copies[i]();
  }
}
```
```javascript
function flushSchedulerQueue () {
  currentFlushTimestamp = getNow();
  flushing = true;
  var watcher, id;

  queue.sort(function (a, b) { return a.id - b.id; });

  for (index = 0; index < queue.length; index++) {
    watcher = queue[index];
    if (watcher.before) {
      // 调用 beforeUpdate 函数
      watcher.before();
    }
    id = watcher.id;
    has[id] = null;
    watcher.run();
    // in dev build, check and stop circular updates.
    if (has[id] != null) {
      circular[id] = (circular[id] || 0) + 1;
      if (circular[id] > MAX_UPDATE_COUNT) {
        warn(
          'You may have an infinite update loop ' + (
            watcher.user
              ? ("in watcher with expression \"" + (watcher.expression) + "\"")
              : "in a component render function."
          ),
          watcher.vm
        );
        break
      }
    }
  }

  // keep copies of post queues before resetting state
  var activatedQueue = activatedChildren.slice();
  var updatedQueue = queue.slice();

  resetSchedulerState();

  // call component updated and activated hooks
  callActivatedHooks(activatedQueue);
  callUpdatedHooks(updatedQueue);

  // devtool hook
  /* istanbul ignore if */
  if (devtools && config.devtools) {
    devtools.emit('flush');
  }
}
```

# 6. 解析组件

在解析组件的时候 首先也是把它当做 普通标签来解析 但是在创建虚拟Dom的时候 会利用 `isHTMLTag` 方法 来判断 这个标签是不是 普通的HTML标签 先利用 **resolveAsset** 方法 拿到 组件里 ***components*** 里面的配置 然后通过 这个 tag 拿到对应的组件

```javascript
components: {
  componentA,
  componentB
}

components[tag] // 拿到对应的配置
```
然后通过 **createComponent** 方法来创建组件的vnode
这个方法主要包括三大部分
* 构造子类构造函数
  * 拿到 vm 上面的 $options 上 _base 配置 然后调用上面的 **extend** 方法 去构建一个 Vue 子类 **Sub** 并返回
  * 在 **extend** 方法里 首先拿到我们写的组件内的 一些 options 就一些

  ```javascript
  name: '',
  data() {
    retrun {

    }
  },
  methods: {},
  ...
  ```
  * 然后将 this (Vue) 赋给 **Super**
  * 将构造函数 **VueComponent(options)** 赋值给 **Sub** 并在里面调用 `this._init(options)` 方法
  * 然后利用原型继承的方式 创建 **Sub** 的 prototype  然后 将原型的 constructor 属性 指向 构造函数 **Sub**

  ```javascript
  Sub.prototype = Object.create(Super.prototype)
  Sub.prototype.constructor = Sub
  ```
  * 然后将我们组件里写的一些 options 和 Vue 自带的 options 通过 **mergeOptions** 方法去进行合并（策略模式） 然后赋值给 **Sub.options**

  ```javascript
  // Vue 自带的
  {
    components: {
      KeepAlive,
      Transition,
      TransitionGroup
    },
    directives: {
      model,
      show
    },
    _base,
    ...
  }
  ```
  * 然后在 **Sub** 上添加一些 属性和全局的API 最后对 **Sub** 进行缓存 并 返回出去

  ```javascript
  function initExtend (Vue) {
    /**
    * Each instance constructor, including Vue, has a unique
    * cid. This enables us to create wrapped "child
    * constructors" for prototypal inheritance and cache them.
    */
    Vue.cid = 0;
    var cid = 1;

    /**
    * Class inheritance
    */
    Vue.extend = function (extendOptions) {
      extendOptions = extendOptions || {};
      var Super = this;
      var SuperId = Super.cid;
      var cachedCtors = extendOptions._Ctor || (extendOptions._Ctor = {});
      if (cachedCtors[SuperId]) {
        return cachedCtors[SuperId]
      }

      var name = extendOptions.name || Super.options.name;
      if (name) {
        validateComponentName(name);
      }

      var Sub = function VueComponent (options) {
        this._init(options);
      };
      Sub.prototype = Object.create(Super.prototype);
      Sub.prototype.constructor = Sub;
      Sub.cid = cid++;
      Sub.options = mergeOptions(
        Super.options,
        extendOptions
      );
      Sub['super'] = Super;

      // For props and computed properties, we define the proxy getters on
      // the Vue instances at extension time, on the extended prototype. This
      // avoids Object.defineProperty calls for each instance created.
      if (Sub.options.props) {
        initProps$1(Sub);
      }
      if (Sub.options.computed) {
        initComputed$1(Sub);
      }

      // allow further extension/mixin/plugin usage
      Sub.extend = Super.extend;
      Sub.mixin = Super.mixin;
      Sub.use = Super.use;

      // create asset registers, so extended classes
      // can have their private assets too.
      ASSET_TYPES.forEach(function (type) {
        Sub[type] = Super[type];
      });
      // enable recursive self-lookup
      if (name) {
        Sub.options.components[name] = Sub;
      }

      // keep a reference to the super options at extension time.
      // later at instantiation we can check if Super's options have
      // been updated.
      Sub.superOptions = Super.options;
      Sub.extendOptions = extendOptions;
      Sub.sealedOptions = extend({}, Sub.options);

      // cache constructor
      cachedCtors[SuperId] = Sub;
      return Sub
    };
  }
  ```
* 安装组件的钩子函数
  * 这里主要是 提供组件 vnode 在 patch 流程中 暴露对外的钩子函数 以便Vue做一些额外的事
  * 主要有这几个钩子函数 `[ init, prepatch, insert, destroy ]`
  * 最后 vnode data 属性里 就会有 这几个 钩子函数了

  ```javascript
  function installComponentHooks (data) {
    var hooks = data.hook || (data.hook = {});
    for (var i = 0; i < hooksToMerge.length; i++) {
      var key = hooksToMerge[i];
      var existing = hooks[key];
      var toMerge = componentVNodeHooks[key];
      if (existing !== toMerge && !(existing && existing._merged)) {
        hooks[key] = existing ? mergeHook$1(toMerge, existing) : toMerge;
      }
    }
  }
  ```
* 生成组件的vnode
  * 通过 new Vnode 生成组件的 vnode 节点

```javascript
function createComponent (
  Ctor,
  data,
  context,
  children,
  tag
) {
  if (isUndef(Ctor)) {
    return
  }

  var baseCtor = context.$options._base;

  // plain options object: turn it into a constructor
  if (isObject(Ctor)) {
    Ctor = baseCtor.extend(Ctor);
  }

  // if at this stage it's not a constructor or an async component factory,
  // reject.
  if (typeof Ctor !== 'function') {
    {
      warn(("Invalid Component definition: " + (String(Ctor))), context);
    }
    return
  }

  // async component
  var asyncFactory;
  if (isUndef(Ctor.cid)) {
    asyncFactory = Ctor;
    Ctor = resolveAsyncComponent(asyncFactory, baseCtor);
    if (Ctor === undefined) {
      // return a placeholder node for async component, which is rendered
      // as a comment node but preserves all the raw information for the node.
      // the information will be used for async server-rendering and hydration.
      return createAsyncPlaceholder(
        asyncFactory,
        data,
        context,
        children,
        tag
      )
    }
  }

  data = data || {};

  // resolve constructor options in case global mixins are applied after
  // component constructor creation
  resolveConstructorOptions(Ctor);

  // transform component v-model data into props & events
  if (isDef(data.model)) {
    transformModel(Ctor.options, data);
  }

  // extract props
  var propsData = extractPropsFromVNodeData(data, Ctor, tag);

  // functional component
  if (isTrue(Ctor.options.functional)) {
    return createFunctionalComponent(Ctor, propsData, data, context, children)
  }

  // extract listeners, since these needs to be treated as
  // child component listeners instead of DOM listeners
  var listeners = data.on;
  // replace with listeners with .native modifier
  // so it gets processed during parent component patch.
  data.on = data.nativeOn;

  if (isTrue(Ctor.options.abstract)) {
    // abstract components do not keep anything
    // other than props & listeners & slot

    // work around flow
    var slot = data.slot;
    data = {};
    if (slot) {
      data.slot = slot;
    }
  }

  // install component management hooks onto the placeholder node
  installComponentHooks(data);

  // return a placeholder vnode
  var name = Ctor.options.name || tag;
  var vnode = new VNode(
    ("vue-component-" + (Ctor.cid) + (name ? ("-" + name) : '')),
    data, undefined, undefined, undefined, context,
    { Ctor: Ctor, propsData: propsData, listeners: listeners, tag: tag, children: children },
    asyncFactory
  );

  return vnode
}
```

# 7. 生成组件

生成组件的时候 同样要将组件的 vnode 转成真实的 dom 所以也需要调用 **createElm** 方法（如上）</br>
* 在这个方法里 首先调用一下 **createComponent** 方法 并且如果是组件 在这个方法里执行对应的组件逻辑 最后返回是否是组件的布尔值

  ```javascript
  //createElm 判断并调用createComponent 
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }
  ```
* **createComponent** 方法里主要判断 vnode **data** 属性里有没有组件的钩子函数 如果有 则直接调用 组件的钩子函数 **init**

  ```javascript
  //执行完createComponent 如果是组件 直接 return 否则继续执行 createElm
  function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
    var i = vnode.data;
    if (isDef(i)) {
      var isReactivated = isDef(vnode.componentInstance) && i.keepAlive;
      if (isDef(i = i.hook) && isDef(i = i.init)) {
        i(vnode, false /* hydrating */);
      }
      // after calling the init hook, if the vnode is a child component
      // it should've created a child instance and mounted it. the child
      // component also has set the placeholder vnode's elm.
      // in that case we can just return the element and be done.
      if (isDef(vnode.componentInstance)) {
        initComponent(vnode, insertedVnodeQueue);
        insert(parentElm, vnode.elm, refElm);
        if (isTrue(isReactivated)) {
          reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm);
        }
        return true
      }
    }
  }
  ```
* **init**方法 主要调用 **createComponentInstanceForVnode** 方法对 组件进行一些初始化 最后调用 $mount方法 进行挂载 对 template 里面东西进行模板解析 然后就和 模板解析下面的过程一样了

  ```javascript
  function init (vnode, hydrating) {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      var mountedNode = vnode; // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode);
    } else {
      var child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      );
      child.$mount(hydrating ? vnode.elm : undefined, hydrating);
    }
  }
  ```
* **createComponentInstanceForVnode** 方法 实例化 之前 构造出的子类的构造函数 构造函数里面调用 `this._init(options)` 方法去初始化 组件里面的一些生命周期 事件 state 等等 其实就是 _init 方法做的一些事情 然后返回 `this._init(options)` 构造出来的实例 赋值给 vnode.componentInstance 属性 在之后 渲染里 提供给 组件的钩子函数 **init** 用

  ```javascript
  function createComponentInstanceForVnode (
    vnode, // we know it's MountedComponentVNode but flow doesn't
    parent // activeInstance in lifecycle state
  ) {
    var options = {
      _isComponent: true,
      _parentVnode: vnode,
      parent: parent
    };
    // check inline-template render functions
    var inlineTemplate = vnode.data.inlineTemplate;
    if (isDef(inlineTemplate)) {
      options.render = inlineTemplate.render;
      options.staticRenderFns = inlineTemplate.staticRenderFns;
    }
    return new vnode.componentOptions.Ctor(options)
  }
  ```

# 8. keep-alive

解析 **keep-alive** 组件和解析普通的组件是一样的 参照上面逻辑 只不过是 **keep-alive** 组件的 options 是 Vue 提供的 如下 有些 `name, abstract, props created, destroyed, mounted, render` 配置

## props

接收三个属性：include exclude max<br>
include: 表示只有匹配的组件才会被缓存<br>
exclude: 表示任何匹配到的组件都不会被缓存<br>
max: 表示缓存组件的最大数量<br>

## created 

在 **created** 里面初始化两个属性 `this.cache` 和 `this.keys`<br>
`this.cache` 是一个对象 里面存储需要缓存的组件 `this.keys` 是一个数组 用来存储缓存组件的 **key**<br>

## destroyed

当 **keep-alive** 组件被销毁时 此时会调用 **destroyed** 钩子函数 该钩子函数里会遍历 `this.cache` 对象 然后将那些缓存的并且当前没有处于被渲染的组件都销毁并将其从 `this.cache` 对象中剔除

```javascript
function pruneCacheEntry (
  cache,
  key,
  keys,
  current
) {
  var cached$$1 = cache[key];
  if (cached$$1 && (!current || cached$$1.tag !== current.tag)) {
    cached$$1.componentInstance.$destroy();
  }
  cache[key] = null;
  remove(keys, key);
}
```

## mounted

在 mounted 钩子函数中观测 **include** **exclude** 的变化<br>
如果 **include** 和 **exclude** 发生了变化 即表示定义需要缓存的的组件规则或者不需要缓存的组件的规则发生了变化 那么就调用 **pruneCache** 方法
* 该方法对 `this.cache` 对象进行遍历 取出每一项的 **name** 值 用其与新的缓存规则进行匹配 如果匹配不上 则表示新的缓存规则下该组件已经不需要缓存 则调用 **pruneCacheEntry** 方法将这个已经不需要缓存的组件先销毁掉 然后将其从 `this.cache` 对象中剔除掉

```javascript
function pruneCache (keepAliveInstance, filter) {
  const { cache, keys, _vnode } = keepAliveInstance
  for (const key in cache) {
    const cachedNode = cache[key]
    if (cachedNode) {
      const name = getComponentName(cachedNode.componentOptions)
      if (name && !filter(name)) {
        pruneCacheEntry(cache, key, keys, _vnode)
      }
    }
  }
}
```

## render

* 在对 **keep-alive** 进行挂载的时候 同样会执行到 mounted 的阶段 走之前上面的 mounted 阶段 回调生成好的 render 函数 因为 **keep-alive** 的 **render** 函数 是 Vue 已经定义好了 所以直接调用 **keep-alive** 里的 **render** 函数（函数组件）
* 因为通常来说我们在 **keep-alive** 里面写的是组件或者Dom 所以 keep-alive 默认获取插槽里的第一个子节点或者第一个子节点
* 然后调用 **getComponentName** 方法去拿到组件的名称 这个方法优先取组件的 **name** 字段， 如果没有则获取 组件的tag
* 然后用组件的名称去和 **include** 和 **exclude** 的规则去匹配 
* 如果组件名称与 **include** 规则不匹配或者与 **exclude** 规则匹配，则表示不缓存该组件，直接返回这个组件的 **vnode** 否则就是走接下来的缓存逻辑
* 首先将 this 赋值 给 **ref$1** this 指向 就是在 render 函数 **call** 调用的时候 绑的 指向 `vm._renderProxy`
* 然后取出 **ref$1** 的 **cache** 再去 取 **key** 值 拿到 **key** 值后去 **cache** 中去找 如果缓存中有 直接去缓存中去拿到 **vnode** 的组件实例，此时重新调整该组件的 **key** 的顺序，将其从原来的地方删除掉 并重新放在 `this.keys` 中最后一个
* 如果 `this.cache` 中没有该 **key** 表明该组件还没有被缓存过，则以该组件的 **key** 为键，组件 **vnode** 为值，将其存入 `this.cache` 中，并且把 **key** 存入 `this.keys` 中。此时再判断 `this.keys` 中缓存组件的数量是否超过了设置的最大缓存数量值 `this.max`，如果超过了，则把第一个缓存组件删掉
* 最后将 `vnode.data.keepAlive = true` 最后将 **vnode** 返回

有一个问题：<font  color=red>**为什么要删除第一个缓存组件并且为什么命中缓存了还要调整组件key的顺序？**</font>

这其实应用了一个缓存淘汰策略LRU：
> LRU（**Least recently used**，最近最少使用）算法根据数据的历史访问记录来进行淘汰数据，其核心思想是“如果数据最近被访问过，那么将来被访问的几率也更高”。

它的算法是这样子的：
1. 将新数据从尾部插入到 `this.keys` 中
2. 每当缓存命中，则将数据移到 `this.keys` 尾部
3. 当 `this.keys` 满的时候，将头部数据丢弃

LRU的核心思想是思想如果数据被访问过 那么被访问的几率也更高，所以我们将命中缓存的组件 **key** 重新插入到 `this.keys` 的尾部，这样一来 `this.keys` 中越往头部的数据将来被访问的几率越低 所以当缓存数量达到最大值的时 我们就删除将来被访问几率最低的数据 即 `this.keys` 中第一个缓存的组件即 `this.keys` 中的头部组件

```javascript
var KeepAlive = {
  name: 'keep-alive',
  abstract: true,

  props: {
    include: patternTypes,
    exclude: patternTypes,
    max: [String, Number]
  },

  created: function created () {
    this.cache = Object.create(null);
    this.keys = [];
  },

  destroyed: function destroyed () {
    for (var key in this.cache) {
      pruneCacheEntry(this.cache, key, this.keys);
    }
  },

  mounted: function mounted () {
    var this$1 = this;

    this.$watch('include', function (val) {
      pruneCache(this$1, function (name) { return matches(val, name); });
    });
    this.$watch('exclude', function (val) {
      pruneCache(this$1, function (name) { return !matches(val, name); });
    });
  },

  render: function render () {
    var slot = this.$slots.default;
    var vnode = getFirstComponentChild(slot);
    var componentOptions = vnode && vnode.componentOptions;
    if (componentOptions) {
      // check pattern
      var name = getComponentName(componentOptions);
      var ref = this;
      var include = ref.include;
      var exclude = ref.exclude;
      if (
        // not included
        (include && (!name || !matches(include, name))) ||
        // excluded
        (exclude && name && matches(exclude, name))
      ) {
        return vnode
      }

      var ref$1 = this;
      var cache = ref$1.cache;
      var keys = ref$1.keys;
      var key = vnode.key == null
        // same constructor may get registered as different local components
        // so cid alone is not enough (#3269)
        ? componentOptions.Ctor.cid + (componentOptions.tag ? ("::" + (componentOptions.tag)) : '')
        : vnode.key;
      if (cache[key]) {
        vnode.componentInstance = cache[key].componentInstance;
        // make current key freshest
        remove(keys, key);
        keys.push(key);
      } else {
        cache[key] = vnode;
        keys.push(key);
        // prune oldest entry
        if (this.max && keys.length > parseInt(this.max)) {
          pruneCacheEntry(cache, keys[0], keys, this._vnode);
        }
      }

      vnode.data.keepAlive = true;
    }
    return vnode || (slot && slot[0])
  }
};
```
## 生命周期函数

组件一旦被 **keep-alive** 缓存 那么再次渲染的时候就不会执行 **created** 和 **mounted** 等钩子函数<br>
有时候我们需要再次渲染的时候需要做一些事情 Vue 提供了 两个钩子函数 **activated** 和 **deactivated** 他们的执行时机是 **keep-alive** 包裹的组件激活时调用和停用时调用