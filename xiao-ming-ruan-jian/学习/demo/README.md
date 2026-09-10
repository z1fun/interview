# Vue 3 父子组件通信 — 完整示例

> 演示 `defineProps` / `defineEmits` / `provide` / `inject` / `v-model` 的实战用法。

## 文件结构

```
demo/
├── README.md          ← 当前文件
├── TodoItem.vue       ← 子组件: defineProps + defineEmits + inject
├── TodoList.vue       ← 父组件: 使用 TodoItem + provide
├── RatingInput.vue    ← 子组件: v-model 本质拆解（props + emit 语法糖）
└── App.vue            ← 入口: 把所有组件串起来的完整示例
```

## 数据流总览

```
                      defineProps（父传子）
   父组件  ──────────────────────────────→  子组件
   TodoList        :todo / :index          TodoItem
          ←──────────────────────────────
                  defineEmits（子传父）
               @edit / @delete / @toggle

                      provide（祖先传后代）
   祖先组件 ──────────────────────────────→ 任意深度的后代
   (App)         provide('theme')          (TodoItem)
                                           inject('theme')
```

## 三种通信模式对照

| 模式 | 方向 | 语法 | 适用场景 |
|------|------|------|----------|
| **defineProps** | 父→子 | `<Child :prop="val" />` | 父组件向子组件传递数据 |
| **defineEmits** | 子→父 | `<Child @event="handler" />` | 子组件通知父组件发生了某事 |
| **v-model** | 双向 | `<Child v-model="val" />` | 表单输入、评分等需要同步的场景 |
| **provide / inject** | 祖先→后代 | `provide('key', val)` / `inject('key')` | 跨越多层，如主题、语言、用户信息 |

## 核心要点

### 1. 编译宏（Compiler Macros）

`defineProps` 和 `defineEmits` 是 Vue 3 的**编译时宏**，不是运行时函数：

- **不需要** `import` —— 编译器自动注入
- 在 `node_modules` 里找不到它们的源码
- 由 `@vue/compiler-sfc` 在构建时转换为运行时代码

### 2. 单向数据流

props 是**只读**的。子组件永远不直接修改 `props.xxx`，而是通过 emit 通知父组件去改：

```typescript
// ❌ 错误：直接修改 prop
props.modelValue = 5;

// ✅ 正确：通过 emit 通知父组件
emit('update:modelValue', 5);
```

### 3. v-model 本质

```vue
<!-- 这两行完全等价 -->
<RatingInput v-model="score" />
<RatingInput :modelValue="score" @update:modelValue="(v) => score = v" />
```

v-model 就是 `defineProps`（接收 `modelValue`）+ `defineEmits`（触发 `update:modelValue`）的语法糖。

### 4. TypeScript 类型安全

```typescript
// Props 类型 —— 编译期就能捕获传错类型的问题
const props = defineProps<{
  todo: { id: number; text: string; done: boolean };  // 必传
  index?: number;                                       // 可选
}>();

// Emits 类型 —— 编译期就能捕获事件名或参数类型错误
const emit = defineEmits<{
  (e: 'edit',   id: number): void;
  (e: 'delete', id: number): void;
  (e: 'toggle', id: number, done: boolean): void;
}>();
```

## 运行方式

如果你有 Vue 3 项目，把这些 `.vue` 文件放到 `src/components/` 下即可使用。纯学习用途无需运行——直接阅读代码中的注释即可理解原理。
