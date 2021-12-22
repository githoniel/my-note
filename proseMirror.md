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

PS: slatejs 实践中缺乏 persistence global state，需要保存在 schema 之中，共享的编辑器状态。之前实现的方法是丢在第一个 block 中，这种方法显然存在缺陷（保护块不被移除）。最好是在 schema 中有一个特殊的 key 用于保存共享状态

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
- Commands: undo/redo 等，slatejs 直接扩展在 editor 上
- Content: 渲染出来的 DOM 内容，slatejs 因为没有自带 dom parser，没有这个概念
- Document: schema 数据，ProseMirror 例子中特意说到一点，mark 需要摊平，不允许多层嵌套(防止出现额外的冗余位置)

### Schema

1. ProseMirror 例子中特意说到一点，markup需要摊平，不允许多层嵌套(防止出现额外的冗余位置)
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

PS: slatejs 中的限制中，trade off 在于，复杂度总是存在的，你可以堆到更严格的schema限制，让一次性的normalize中，也可以堆到不可控的的`transform` 中，显然前者更好一点

### 不可变性

ProseMirror 的 document 是不可变的，这个和 slatejs 一致。

ProseMirror 认为 immutable 的优势

1. 避免更新时，document出现不合法的中间态。因为可以在新文档生成后直接替换原内容
2. 更容易追踪文档变化
3. 使得 collaborative 成为可能，根据文档Diff来实现高效的DOM更新

从文档上看，ProseMirror不像 slatejs 这样使用 immerjs 进行了 freezing 属性

确实好像不可变性更好，immutable 更符合 OP 化的设定

### Schema

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

### Indexing

ProseMirror 支持两种坐标体系

- offset: number
- path: number[]

slatejs 只支持 path 坐标体系，如果需要 offset，可以调用方法来动态计算。


`offset` 还额外带来了 `nodeSize` 属性，这样子也可以高效的计算出 `contentSize`

### Slices

copy-paste 和 drag-drop 操作中需要用到 fragment 的概念。

ProseMirror 支持通过 offset 进行 `slice`，slatejs 则是常规的通过选区来实现。

### Node Types


