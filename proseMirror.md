# ProseMirror

## Link

ProseMirror: https://prosemirror.net/
slatejs: https://docs.slatejs.org/

## ProseMirror Guide

主要介绍基础概念，与 slatejs 类似，但是提供了更多的可配置属性

### 核心机制

核心机制与 slatejs 极为相似，也是 react 核心机制那一套

- schema 映射渲染到 dom，状态决定文档内容
- 用户操作转换为 `transaction`, `dispatchTransaction` 操作 schema , schema 改变触发刷新 DOM
- 由 DOM 触发的操作为 state 中的 selection / focus 等

### 分包

- prosemirror-model: 定义编辑器数据结构
- prosemirror-state: 定义编辑器状态，selection / transaction 等
- prosemirror-view: 视图层与渲染
- prosemirror-transform: modal的可用操作

与 slatejs 相比

- slate: 相当于 prosemirror-model && prosemirror-transform
- slate-react: 相当于 prosemirror-state && prosemirror-view
  
此外 ProseMirror 自带的插件机制，无需重新封装中间层

PS: slatejs 实践中缺乏 persistence global state，需要保存在 `schema` 之中，共享的编辑器状态。之前实现的方法是丢在第一个 block 中，这种方法显然存在缺陷（保护块不被移除）。最好是在 `schema` 中有一个特殊的 key 用于保存共享状态

### 基础概念

schema -> state -> view

```ts
import { schema } from "prosemirror-schema-basic"
import { EditorState } from "prosemirror-state"
import { EditorView } from "prosemirror-view"

let state = EditorState.create({schema})
let view = new EditorView(document.body, {state})
```

- Transactions: 应该是类似 slatejs 中的 `Editor.transform`
- Plugins: 插件，挂载在 state 上，通过 `EditorState.create` 注入 state
- Commands: undo/redo 等，slatejs 直接扩展在 `editor` 上
- Content: 渲染出来的 DOM 内容，slatejs 因为没有自带 dom parser，没有这个概念
- Document: schema 数据，ProseMirror 例子中特意说到一点，mark 需要摊平，不允许多层嵌套(防止出现额外的冗余位置)

### Schema

1. ProseMirror 例子中特意说到一点，markup 需要摊平，不允许多层嵌套(防止出现额外的冗余位置)
2. 此外和 slatejs 一样，为了唯一性，要求相邻Mark合并、不允许空文本节点。
3. ProseMirror 通用节点属性
  - isBlock/isInline
  - inlineContent: 子节点全是 inline
  - isTextblock: block节点，而且 inlineContent
  - isLeaf: 不允许嵌套子内容

回顾一下 slatejs的基础[schema规范](https://docs.slatejs.org/concepts/11-normalizing#built-in-constraints)

1. All Element nodes must contain at least one Text descendant — even Void Elements. If an element node does not contain any children, an empty text node will be added as its only child. This constraint exists to ensure that the selection's anchor and focus points (which rely on referencing text nodes) can always be placed inside any node. With this, empty elements (or void elements) wouldn't be selectable. PS: Selection限制
2. Two adjacent texts with the same custom properties will be merged. If two adjacent text nodes have the same formatting, they're merged into a single text node with a combined text string of the two. This exists to prevent the text nodes from only ever expanding in count in the document, since both adding and removing formatting results in splitting text nodes. PS: 确保唯一性以及节点数量
3. Block nodes can only contain other blocks, or inline and text nodes. For example, a paragraph block cannot have another paragraph block element and a link inline element as children at the same time. The type of children allowed is determined by the first child, and any other non-conforming children are removed. This ensures that common richtext behaviors like "splitting a block in two" function consistently. PS：为了OP的便利性以及行的准确计算
4. Inline nodes cannot be the first or last child of a parent block, nor can it be next to another inline node in the children array. If this is the case, an empty text node will be added to correct this to be in compliance with the constraint. PS: 为了方便前后插入文本[Issue](https://github.com/ianstormtaylor/slate/issues/2865)
5. The top-level editor node can only contain block nodes. If any of the top-level children are inline or text nodes they will be removed. This ensures that there are always block nodes in the editor so that behaviors like "splitting a block in two" work as expected. PS：同3

PS: slatejs 中的限制中，trade off 在于，复杂度总是存在的，你可以堆到更严格的 schema 限制，让一次性的 `normalize` 中，也可以堆到不可控的的 `transform` 中，显然前者更好一点

#### 不可变性

ProseMirror 的 `document` 是不可变的，这个和 slatejs 一致。

ProseMirror 认为 immutable 的优势

1. 避免更新时，document出现不合法的中间态。因为可以在新文档生成后直接替换原内容
2. 更容易追踪文档变化
3. 使得 collaborative 成为可能，根据文档Diff来实现高效的DOM更新

从文档上看，ProseMirror不像 slatejs 这样使用 immerjs 进行了 freezing 属性

确实好像不可变性更好，immutable 更符合 OP 化的设定

#### Schema

ProseMirror 显然比 slatejs 近一步抽象

```ts
// ProseMirror
interface Mark {
  type: string, // MarkType
  attrs: Object
}

interface Node {
  type: string // NodeType
  content: Node[] // Fragment
  attrs: Object, // only for inline content
  marks: Mark[] // Marks
}
```


```ts
// slatejs
interface Node {
  children: Node[],
  [key: string]: unknown
}

interface Text {
  text: string,
  [key: string]: unknown
}
```

slatejs 也是需要诸如 `type` 之类的常用属性，等于上层封装全开放给用户自由定义。这也带来了困扰，比如 `Editor.Marks` 就需要覆盖实现 omit 掉部分属性

#### Indexing

ProseMirror 支持两种坐标体系

- offset: number
- path: number[]

slatejs 只支持 path 坐标体系，如果需要 offset，可以调用方法来动态计算。


`offset` 还额外带来了 `nodeSize` 属性，这样子也可以高效的计算出 `contentSize`

#### Slices

copy-paste 和 drag-drop 操作中需要用到 `fragment` 的概念。

ProseMirror 支持通过 offset 进行 `slice`，slatejs 则是常规的通过选区来实现。

#### Schemas

ProseMirror 的每一个 `document` 有一个 `schema` 属性，用于规定可能的 nodes 类型

```ts
interface SchemaSpec {
  nodes: NodeSpec | Map<NodeSpec> // https://prosemirror.net/docs/ref/#model.NodeSpec
  marks?: MarkSpec | Map<MarkSpec> // 
  topNode?: string // name of the default top-level node for the schema
}
```
slatejs 则没有这么复杂的机制，只有简单的 `Editor.normalize` 函数

#### Content Expressions

`NodeSpec` 里有一个特殊的属性 `content`，用于说明可以包含的子节点类型，类似正则语法。

- 量词参考正则，支持分组
- 字符类为节点 `type` 的值，也可以通过 `group` 字段实现分组

特别要注意，`NodeSpec` 次序是很重要的，`group`中的首个类型将是默认类型，不慎使用将导致死循环

PS：这个设计并不好

Schema检查时机在更高级抽象的 Transform 中进行而不是原子化的 OP 中。这个和 slate 类似。

#### Marks

ProseMirror Marks实现更为复杂，`MarkSpec`有诸多属性。

`NodeSpec` 的 marks 是一个文本，规定了允许的 `mark`。

- 使用空白分隔符
- _匹配所有
- 空字符串为不使用任何 `mark`

inline 内容默认允许所有 `mark`

#### Attributes

ProseMirror 中， Node 和 Mark 都可以有 `attrs` 属性, `default` 是其默认值，没有默认值则必须手动指定。

#### Serialization and Parsing

ProseMirror 中，对于 `Node` 是定义在 `NodeType.toDOM` 函数中，`Mark` 也是类似。

此外也有对应的 `parseDOM` 属性。`parserRule` 是根据配置来实现，[ParserRule](https://prosemirror.net/docs/ref/#model.ParseRule)

## Document Transformations

PromseMirror 中，单独 Op 称之为 `Step`, 集合多个 `Step` 的操作，称为`Transform`, 一般不直接调用 `Step`。

特别的是 ProseMirror 中的 `Step` 可以返回 `result` 为错误。

### Mapping

类似 slatejs 中的 `ref` 概念，用于指定一个跨状态保持恒定的位置。

API 显得比较奇怪，依赖于具体的操作中。

```ts
// Step 的 Mapping
let step = new ReplaceStep(4, 6, Slice.empty) // Delete 4-5
let map = step.getMap()
console.log(map.map(8)) // → 6
console.log(map.map(2)) // → 2 (nothing changes before the change)
```

```ts
// Transform 的 Mapping
let tr = new Transform(myDoc)
tr.split(10)    // split a node, +2 tokens at 10
tr.delete(2, 5) // -3 tokens at 2
console.log(tr.mapping.map(15)) // → 14
console.log(tr.mapping.map(6))  // → 3
console.log(tr.mapping.map(10)) // → 9
```

slatejs 中的 `ref` 自动跨任何操作，直到主动 `unRef()`

### Rebasing

不同 OP 执行次序，导致的非首个 OP 都要利用 Mapping 更新坐标。

```
stepA(doc) = docA
stepB(doc) = docB
stepB(docA) = MISMATCH!
rebase(stepB, mapA) = stepB'
stepB'(docA) = docAB
```

超长链接下确实很难理解

## Editor State

内置状态基础状态，可以插件扩展

- doc
- selection
- storedMarks

### Selection

- anchor: 编辑后不动的点
- head: 随着编辑移动的点

这个和 DOM 定义的 anchor / focus 不同。

ProseMirror 中，选区还有不同的类别，

- TextSelection: 起止点都是文字或者inline
- NodeSelection: 起止点都是节点

### Transactions

会随动 selection 和其他状态的 `Transform`

```ts
// tr is transaction
let tr = state.tr
console.log(tr.selection.from) // → 10
tr.delete(6, 8)
console.log(tr.selection.from) // → 8 (moved back)
tr.setSelection(TextSelection.create(tr.doc, 3))
console.log(tr.selection.from) // → 3
```

mark 当然也是随动的

### Plugins

插件设置在 state 上

```js
let state = EditorState.create({schema, plugins: [myPlugin]})
```

插件通过以下几个机制来实现功能

- `Plugin.props: EditorProps` 中的事件

诸如 `handleKeyDown`, `handleClick`，`handleDrop`等

- `Plugin.props: EditorProps` 中注入辅助函数

诸如 `clipboardParser`, `transformPastedHTML`，`nodeViews`, `decorator`等


- `Plugin.spec: PluginSpec` 中可以对 `view` 和 `Transaction` 进一步处理

- `Plugin.constructor` 中可以传入 `state` 来新增状态， 注意由于 `state` 是注入到 `editor` state 中，所以也是 `immutable`

Trasaction 有一个 `metadata: string` 用于区分是否是本插件的操作，或者用于其他分类，通过 `tr.setMeta`/ `tr.getMeta` 实现。

系统也有一些内置的 `metadata` 比如 `addToHistory`，`paste` 等。

PS： 与 `slatejs` 不同的是扩展的 editor 属性也是不可变的，也是算入 `Transaction` 中

## View

和 slatejs 一样，内置的东西不多，只是核心的事件等等支持

### Editable DOM

