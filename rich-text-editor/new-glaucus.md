# New Glaucus

## Design

- 改变在于扩展 `editor`，而不是 `slate-react`
- 更好的 `ts` 支持

### Schema

```ts
interface Schema {
  children: Descendant[]
  attrs: {
    [key: string]: unknown,
  }
}

interface ExtendProp {
  marks?: {
    [key: string]: unknown,
  }
  attrs: {
    [key: string]: unknown,
  }
}

interface GlaucusElement extends ExtendProp, Element {
  type?: string
}

interface GlaucusText extends ExtendProp, Element {
}
```

改进点：

- 拆分 `attrs`/ `marks`, 尽管定义上


### 扩展状态 `EditorState`

```ts
interface EditorState extends Editor {
  state: {
    [key: string]: unknown,
  },
  transformState(fn: (state: object) => void): void
}
```

意义何在 ？ `State` 用于各插件触发刷新

//TODO 是否需要

### 扩展操作 `EditorTransform`

```ts
interface EditorTransform extends Editor {
  transforms: (editor: Editor, ...args: any[]): void
}
```

插件扩展 `Transform` 系列操作

### 扩展API

```ts
interface EditorAPI extends Editor {
  withoutFlushing(editor: Editor, fn: () => void): void
  withoutNormalize(editor: Editor, fn: () => void): void
  // 让强制使 Node 进入 Normalize
  makeDirty(editor: Editor, paths: Path[]): void
  // history扩展 强制不合并
  withoutMerging(editor: Editor, fn: () => void): void
  // history扩展 强制合并
  withMerging(editor: Editor, fn: () => void): void
  // history扩展 强制保存
  withoutSaving(editor: Editor, fn: () => void): void
}
```

### 扩展View层

#### 事件代理

目前事件必须通过 `react` 处理

#### 虚拟光标和输入框

更好的实现选框控制，以及输入支持

### Selection限制

精确匹配，只允许文字选区

#### 扩展状态

控制内容是否受控，目前是依赖 `DOM` 检测。

目前利用的是`contentEditable=false`内的`input/textArea`可以不受控的输入来实现不受控内容内的输入内容。

也就是一个Tree中，需要包含结构 `可编辑 -> 不可编辑 -> 可编辑`

在精确Text选区后这个依旧是需要的，用于 `contentEditable=true` 时依旧不受控的内容

```ts
interface GlaucusEditorView extends Editor {
  isUnselectable(editor: Editor, target: DomNode): boolean
}
```

目前使用地方有

1. onPaste / onKeyDown / onFocus / onCompositionStart / onDOMBeforeInput 等一系列事件的前置判断条件
2. onDOMSelectionChange 中同步 `selection` 到 `editor` 的条件
3. useEffect 中， 同步 `editor.selection` 到 DOM 的条件

包含了 `editable` 所有的交互内容了

1. 显然不属于应当保存的文档内容，属于一次性状态，设置给 DOM 合适
2. hasEditableTarget & isTargetInsideVoid 的关系，需要分情况
   - 对于事件，则应该取 `isEditorSelectable & hasEditableTarget`
   - 对于 `onDOMSelectionChange` 要求 `isEditorSelectable && (hasEditableTarget || isTargetInsideVoid)`
   - 对于 `useEffect`, 要求同上

参考实现如下

```ts
export const SelectableKey = 'slate-not-selectable'

function isEditorSelectable(editor: Editor, target: DomNode) {
  const targetEl = isDOMElement(target)
        ? target
        : target.parentElement) as HTMLElement
  return !targetEl.closest(SelectableKey)!
}
```

是否要实现多层结构？比如 `可编辑 -> 不可编辑 -> 可编辑 -> 不可编辑 -> 可编辑`



### Toolbar优化

抽离 view/state

### Plugin插件优化

- function化
- 提供刷新渲染的方法
- bailHook优化
