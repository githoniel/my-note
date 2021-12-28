# Glaucus

无论开发 slatejs，还是开发 glaucus 并不是一件轻松的事情。

## 设计问题

指的是一些设计缺陷，导致书写起来并不舒服，有些是 slatejs 带来的，有些是插件中间层带来的。

### slatejs

slatejs 分为四个库

- slate: 核心库，防止歧义下面称为`slate-core`
- slate-history
- slate-hyperscript: jsx辅助包，其实没啥用
- slate-react

#### slate-core

- 插件采用函数覆盖式扩展

```ts
const withImages = editor => {
  const { isVoid } = editor

  editor.isVoid = element => {
    return element.type === 'image' ? true : isVoid(element)
  }

  return editor
}
```

1. 如果默认处理函数很复杂，希望部分修改它，是没办法的，只能整体覆盖
2. 嵌套取值原函数，多层嵌套书写困难

- mark 和属性混合, 无法区分，只能用黑白名单
- 无法指定 normalize 指定节点，无法操作 `dirtyPath`
- 无法手动控制 `FLUSHING`, 策略是同步内容都算一个 `Flushing`
- 一些OP存在BUG，在 inline / void 处理上有一些操作不符常理
- Transform 测试覆盖不够
- 序列化没自带实现
- `schema` 缺乏 `global attr`

#### slate-history

- 缺乏手动控制merge步骤的函数，`withoutMerging` / `withMerging` 等

#### slate-react

- 多平台兼容问题
- 生命周期依赖于 `react`，这个其实是 `react hook` 反 JS 的毛病引起的，`hook` 无法声明，触发在组件之外
- Input输入没有实现虚拟输入框，IME兼容问题
- `dirtyPath` 无法手动控制，无法指定刷新某些组件
- 没有统一的事件，全部依赖于 `react`
- 依赖多层级的 `contentEditable` 来控制 `selection` 落点(` hasEditableTarget`）
- 出错直接崩溃

### glaucus

#### `glaucus-plugin`

- `BailHook` / `WaterfallHook` 都不够好

`WaterfallHook` 将多个函数串行，每次处理函数入参为上一个函数的返回值，最终返回值为最后函数处理返回的结果；`BailHook`每次处理函数入参都相同，返回空, 则认为本函数无法处理此内容, 让后续函数处理, 返回其他值则认为本函数的已经做了足够处理, 应当直接返回。

实际开发中就发现两个 `hook` 需要融合，不得不给 `BailHook` 加了一个丑陋的 `updateParam` 函数。

- 状态都丢在 `useContext` 里，实在太多，`react` 还会引发 useContext刷新问题
- `toolbar` 中的指令缺乏抽象成 command，导致书写困难，而且使得快捷键必须依赖`toolbar`
- 插件中间层似乎遗忘了对 `editor` 的扩展新函数，而是只扩展原有接口
- `renderLeaf` / `renderElement` 的区分太过于绝对，需要跨组件就很蛋疼, 
- 所有覆盖式函数，如果不创建组件，就无法使用 `hook`
- 所有事件依赖 `react` 事件来实现，兼容性必须自己分开处理
- `toolbar` 依赖于 `DOM` 的渲染

#### 插件本身

首先理想上的插件应该满足交换群的特性 - 交换律和结合律。实际上还是不太行，所以提供了以下辅助函数

1. 检测函数，`hasPluin`
2. 排序函数，`Before`, `after`
3. 检测函数用于插件主动感知，也有被动感知

目前检测可用性是依托于 `toolbar` 上的按钮或者下拉菜单，根据 `Element.closest` 方式实现

#### glaucus-mark

此类插件的实现原理是通过 `renderLeaf` 函数中 `attribute` 属性的修改实现

```ts
export interface RenderLeafProps {
  children: any
  leaf: Text
  text: Text
  attributes: {
    'data-slate-leaf': true
  } & {
    style?: React.CSSProperties
  }
}

type renderLeaf = (renderLeafProps: IRenderLeafProps) => any
```

当时考虑是为了DOM更加简洁一些，官网的`Demo`是通过 wrap 一层 DOM 来实现，和 `slatejs` 中类似。这两种方法都不完美

1. 插件正交问题，两种都可能出现上下干扰的问题
2. 插件次序问题，两种都可能出现因为插件次序不同而导致的不同结果

#### asd