# TypeScript + Vue3 系统学习

> **定位**：从"会用"到"能讲出深度"——这是面试官判定"熟练"的标准
> **前置依赖**：商云前台项目中的Vue3+TS实战经验
> **学习目标**：能讲Vue3响应式原理、能手写TS工具类型、能回答Vue vs React区别
> [← 返回学习框架](./学习框架.md) | 相关: [面试问答.md](./面试问答.md) · [Go-Python后端.md](./Go-Python后端.md)

---

## 一、TypeScript 类型系统

### 1.1 基础类型

```typescript
// 原始类型
let name: string = "张三";
let age: number = 27;
let isActive: boolean = true;

// 数组
let items: string[] = ["a", "b"];
let items2: Array<number> = [1, 2, 3];

// 元组（固定长度+类型的数组）
let pair: [string, number] = ["age", 27];

// 枚举
enum Status { Pending, Approved, Rejected }

// 字面量类型
let direction: "left" | "right" = "left";

// 联合类型
let id: string | number = "abc123";

// 交叉类型
type A = { name: string };
type B = { age: number };
type C = A & B; // { name: string; age: number }
```

### 1.2 interface vs type ⭐高频考点

```typescript
// interface
interface User {
  name: string;
  age: number;
}
// 声明合并（interface独有）
interface User {
  email: string;
}
// 最终 User = { name, age, email }

// type
type User2 = {
  name: string;
  age: number;
};
// type不能声明合并

// type独有的能力
type StringOrNumber = string | number;  // 联合类型
type Pair = [string, number];           // 元组
type StringKeys<T> = keyof T & string;  // 复杂工具类型
```

**选型原则**：
- 描述对象形状 → interface（更好的扩展性和IDE提示）
- 需要联合类型/交叉类型/工具类型 → type

### 1.3 泛型（Generics）⭐⭐

```typescript
// 基础泛型
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

// 泛型约束 T 拥有 length 属性，且为number
function getLength<T extends { length: number }>(item: T): number {
  return item.length;
}

// 泛型接口
interface ApiResponse<T> {
  code: number;
  data: T;
  message: string;
}

// 使用
type UserResponse = ApiResponse<User>;
type ListResponse = ApiResponse<User[]>;
```

### 1.4 工具类型（Utility Types）⭐⭐⭐面试最爱考

```typescript
interface User {
  id: number;
  name: string;
  email: string;
  password: string;
  age: number;
}

// Partial<T> — 所有属性变可选（常用于更新接口）
type UpdateUser = Partial<User>;
// { id?: number; name?: string; email?: string; ... }

// Pick<T, K> — 选取若干属性（常用于查询投影）
type UserBrief = Pick<User, "id" | "name">;
// { id: number; name: string }

// Omit<T, K> — 排除若干属性（常用于脱敏，去掉password）
type UserWithoutPassword = Omit<User, "password">;
// { id, name, email, age }

// Record<K, V> — 构建键值对映射
type UserMap = Record<string, User>;
// { [key: string]: User }

// Required<T> — 所有属性变必填
type RequiredUser = Required<UpdateUser>;

// Readonly<T> — 所有属性变只读
type ReadonlyUser = Readonly<User>;

// ReturnType<T> — 获取函数返回值类型
function getUser() { return { id: 1, name: "test" }; }
type UserReturn = ReturnType<typeof getUser>;

// Exclude<T, U> / Extract<T, U> — 联合类型过滤
type Status = "pending" | "approved" | "rejected";
type NonPendingStatus = Exclude<Status, "pending">; // "approved" | "rejected"
type ActiveStatus = Extract<Status, "pending" | "approved">; // "pending" | "approved"
```

**手写一个简易的Partial（面试可能会问）**：
```typescript
type MyPartial<T> = {
  [K in keyof T]?: T[K];
};
```

### 1.5 类型守卫（Type Guards）

```typescript
// typeof 守卫
function process(value: string | number) {
  if (typeof value === "string") {
    return value.toUpperCase(); // 这里value是string
  }
  return value.toFixed(2); // 这里value是number
}

// instanceof 守卫
function handleError(error: Error | string) {
  if (error instanceof Error) {
    console.log(error.message);
  } else {
    console.log(error);
  }
}

// 自定义类型守卫
interface Cat { meow(): void }
interface Dog { bark(): void }

function isCat(animal: Cat | Dog): animal is Cat {
  return (animal as Cat).meow !== undefined;
}
```

### 1.6 unknown vs any vs never vs void

| 类型 | 含义 | 使用场景 |
|------|------|---------|
| `any` | 关闭类型检查 | 尽量不用 |
| `unknown` | 不知道类型但要安全 | 替代any，用之前必须类型收窄 |
| `never` | 永远不会到达的值 | 抛出异常的函数返回值、穷尽检查 |
| `void` | 没有返回值 | 函数没有return语句 |

### 1.7 项目结合点

> "商云前台项目里用了Vue3 + TypeScript + Pinia。defineProps和defineEmits都用泛型定义类型，Pinia的store用TypeScript定义State和Actions的类型。interface描述业务模型（商品、订单、会员），泛型工具类型做数据转换。这些能力在AI交互页面开发中同样适用——对话消息的类型定义、API响应的泛型封装、流式数据的类型约束。"

---

## 二、Vue3 核心

### 2.1 Composition API vs Options API

```vue
<!-- Options API -->
<script>
export default {
  data() { return { count: 0 } },
  methods: { increment() { this.count++ } },
  mounted() { console.log("mounted") }
}
</script>

<!-- Composition API -->
<script setup lang="ts">
import { ref, onMounted } from 'vue';

const count = ref(0);
function increment() { count.value++; }
onMounted(() => { console.log("mounted"); });
</script>
```

**Composition API的优势**：
- 逻辑复用更方便（组合式函数）
- TypeScript支持更好
- 按功能组织代码而不是按选项类型

### 2.2 ref vs reactive ⭐⭐⭐

```typescript
// ref — 包装基本类型（内部用class的getter/setter）
const count = ref(0);
count.value++; // 必须用.value

// reactive — 包装对象（内部用Proxy）
const user = reactive({ name: "张三", age: 27 });
user.name = "李四"; // 直接访问

// reactive的坑1：不能解构
// S6 引入的语法，让你一次性从对象/数组中提取多个属性，省去逐条 obj.xxx 的写法
const { name } = user; // name失去响应式！
const { name: nameRef } = toRefs(user); // 用toRefs保持响应式

// reactive的坑2：不能整体替换
let state = reactive({ count: 0 });
state = { count: 1 }; // 失去响应式！
// 用ref包一层
const state = ref({ count: 0 });
state.value = { count: 1 }; // 正常

// 推荐策略：基本类型用ref，对象也用ref（可整体替换）
```

### 2.3 Vue3响应式原理 ⭐⭐⭐必考题

```
Vue2用Object.defineProperty
  ↓ 问题：
  - 不能检测属性添加/删除（需要Vue.set）
  - 不能直接处理数组索引
  - 初始化时需要递归遍历所有属性（性能差）

Vue3用Proxy
  ↓ 优势：
  - 拦截所有操作（get/set/deleteProperty/has等）
  - 能检测属性增删
  - 直接处理数组索引
  - 惰性响应（访问时才递归，初始化快）
```

**简化版的reactive实现思路**：
```typescript
function reactive<T extends object>(obj: T): T {
  return new Proxy(obj, {
    get(target, key, receiver) {
      // 依赖收集：记录"谁在用这个属性"
      track(target, key);
      const result = Reflect.get(target, key, receiver);
      // 嵌套对象也做响应式（惰性——用到才递归）
      if (typeof result === 'object' && result !== null) {
        return reactive(result);
      }
      return result;
    },
    set(target, key, value, receiver) {
      const oldValue = target[key];
      const result = Reflect.set(target, key, value, receiver);
      // 触发更新：通知所有依赖"这个属性变了"
      if (oldValue !== value) {
        trigger(target, key);
      }
      return result;
    }
  });
}
```

**为什么ref需要.value**：
> ref内部用一个class包装了get value()和set value()，因为基本类型（number、string）不能直接Proxy。通过.value访问时触发get收集依赖，修改.value时触发set通知更新。

### 2.4 computed vs watch vs watchEffect

```typescript
// computed — 有返回值，依赖变化后重新计算，有缓存
const double = computed(() => count.value * 2);

// watch — 监听特定数据变化后执行副作用，无返回值
watch(count, (newVal, oldVal) => {
  console.log(`count从${oldVal}变成${newVal}`);
});

// watchEffect — 自动追踪回调里用的所有响应式数据，立即执行
watchEffect(() => {
  console.log(`count是${count.value}, double是${double.value}`);
});
```

| 特性 | computed | watch | watchEffect |
|------|----------|-------|-------------|
| 返回值 | 有 | 无 | 无 |
| 缓存 | 有 | 无 | 无 |
| 立即执行 | 否（访问时计算） | 可配 | 是 |
| 依赖追踪 | 自动 | 手动指定 | 自动 |

### 2.5 组件通信

```vue
<!-- 父传子：props -->
<script setup lang="ts">
const props = defineProps<{ title: string; count?: number }>();
</script>

<!-- 子传父：emits -->
<script setup lang="ts">
const emit = defineEmits<{
  (e: 'update', value: number): void;
  (e: 'delete', id: string): void;
}>();
emit('update', 42);
</script>

<!-- v-model 本质是 props + emit 的语法糖 -->
<!-- v-model:title 等价于 :title + @update:title -->

<!-- 跨层级：provide / inject -->
<!-- 祖先 -->
<script setup>
provide('theme', 'dark');
</script>
<!-- 任意后代 -->
<script setup>
const theme = inject('theme');
</script>

<!-- 模板引用：ref获取子组件实例 -->
<script setup>
const childRef = ref<InstanceType<typeof ChildComp>>();
</script>
```

### 2.6 生命周期

| Vue2 Options | Vue3 Composition | 说明 |
|-------------|-----------------|------|
| beforeCreate | setup()本身 | 初始化前 |
| created | setup()本身 | 初始化后 |
| beforeMount | onBeforeMount | DOM挂载前 |
| mounted | onMounted | DOM挂载后 |
| beforeUpdate | onBeforeUpdate | 数据更新前 |
| updated | onUpdated | 数据更新后 |
| beforeUnmount | onBeforeUnmount | 组件卸载前 |
| unmounted | onUnmounted | 组件卸载后 |

### 2.7 虚拟DOM与Diff优化

**Vue3的diff优化**：
- **静态提升**：不变的静态节点提升到render函数外，不参与diff
- **PatchFlag**：动态节点打标记（文本是动态的、class是动态的），diff时只比标记的部分
- **Block Tree**：动态节点收集到数组，diff时只遍历数组而不是整棵树

> **为什么v-for不能用index当key**：key是VNode的身份标识。用index时，如果数组中间插入一项，后面所有项的index都变了——Vue会错误地认为它们都是新节点，导致不必要的DOM重建和状态错乱。

### 2.8 性能优化

- **计算属性缓存**：computed有缓存，methods每次调用都重新计算
- **异步组件**：`defineAsyncComponent(() => import('./Heavy.vue'))`
- **keep-alive**：缓存组件状态，避免重新创建
- **v-once/v-memo**：只渲染一次或按条件跳过
- **防抖/节流**：高频事件（搜索框输入、滚动）用防抖节流限制执行频率

---

## 三、Vue3工程化

### 3.1 Pinia（状态管理）

```typescript
// stores/user.ts
import { defineStore } from 'pinia';

export const useUserStore = defineStore('user', () => {
  // state
  const user = ref<User | null>(null);
  const token = ref('');

  // getters（类似computed）
  const isLoggedIn = computed(() => !!token.value);

  // actions
  async function login(username: string, password: string) {
    const res = await api.login(username, password);
    token.value = res.token;
    user.value = res.user;
  }

  return { user, token, isLoggedIn, login };
});
```

**比Vuex的优势**：没有mutations（直接改State）、更好的TS支持、模块化自然。

### 3.2 Vue Router

```typescript
const routes = [
  {
    path: '/admin',
    component: () => import('@/layouts/Admin.vue'),
    meta: { requiresAuth: true, roles: ['admin'] },
    children: [...]
  }
];

// 全局守卫：权限控制
router.beforeEach((to, from) => {
  const userStore = useUserStore();
  if (to.meta.requiresAuth && !userStore.isLoggedIn) {
    return '/login';
  }
  if (to.meta.roles && !to.meta.roles.includes(userStore.user?.role)) {
    return '/403';
  }
});
```

### 3.3 Vite

- 开发时用ES Modules做热更新（秒级启动，不用打包）
- 生产用Rollup打包
- 相比Webpack：启更快、热更新更快、配置更简单

### 3.4 TypeScript在Vue3中的使用

```vue
<script setup lang="ts">
import { ref, computed } from 'vue';

// Props类型
const props = defineProps<{
  messages: ChatMessage[];
  loading?: boolean;
}>();

// Emits类型
const emit = defineEmits<{
  (e: 'send', content: string): void;
  (e: 'clear'): void;
}>();

// 模板ref类型
const inputRef = ref<HTMLInputElement>();

// 组合式函数返回类型
function useChat() {
  const messages = ref<ChatMessage[]>([]);
  const sendMessage = async (content: string) => { ... };
  return { messages, sendMessage };
}
</script>
```

---

## 四、React速查

### 4.1 函数组件与Hooks

```tsx
import { useState, useEffect, useMemo, useCallback, useRef } from 'react';

function ChatApp({ initialMessages }: { initialMessages: Message[] }) {
  // useState = ref
  const [messages, setMessages] = useState<Message[]>(initialMessages);
  const [loading, setLoading] = useState(false);

  // useEffect = watch/watchEffect
  useEffect(() => {
    fetchMessages();
  }, []); // 空数组 = onMounted

  // useMemo = computed
  const messageCount = useMemo(() => messages.length, [messages]);

  // useCallback = 缓存函数引用
  const sendMessage = useCallback((content: string) => {
    setMessages(prev => [...prev, { content, role: 'user' }]);
  }, []);

  // useRef = ref（但修改不触发重渲染）
  const inputRef = useRef<HTMLInputElement>(null);

  return <div>{messages.map(m => <p key={m.id}>{m.content}</p>)}</div>;
}
```

### 4.2 Vue vs React 对比

| 维度 | Vue3 | React |
|------|------|-------|
| 响应式 | Proxy自动追踪 | setState手动触发 |
| 模板 | HTML-like模板 | JSX |
| 状态管理 | Pinia | Redux/Zustand |
| 组件复用 | Composables | Custom Hooks |
| 学习曲线 | 平缓 | 稍陡（JSX/Hooks心智模型） |
| 生态大小 | 中等 | 巨大 |

---

## 五、前端AI交互开发要点

### 5.1 流式输出处理

```typescript
async function* streamChat(message: string) {
  const response = await fetch('/api/chat', {
    method: 'POST',
    body: JSON.stringify({ message }),
    headers: { 'Content-Type': 'application/json' }
  });
  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    yield decoder.decode(value);
  }
}

// 在Vue中使用
const content = ref('');
for await (const chunk of streamChat(userInput)) {
  content.value += chunk;
}
```

### 5.2 对话UI的状态管理

```typescript
interface ChatMessage {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  timestamp: number;
  sources?: Document[]; // RAG引用来源
}

// 用Pinia管理对话状态
const useChatStore = defineStore('chat', () => {
  const messages = ref<ChatMessage[]>([]);
  const isStreaming = ref(false);
  const currentMessage = ref('');

  async function sendMessage(content: string) {
    messages.value.push({ id: uuid(), role: 'user', content, timestamp: Date.now() });
    isStreaming.value = true;
    // 处理流式响应...
  }

  return { messages, isStreaming, currentMessage, sendMessage };
});
```

---

## 六、面试高频题 + 自查清单

### 面试高频题（答案详见 [面试问答.md](./面试问答.md) §9）

1. interface和type的区别
2. 泛型怎么用？手写一个示例
3. Partial/Pick/Omit/Record的用法
4. Vue3响应式原理（Proxy vs defineProperty）
5. ref和reactive的区别和坑点
6. computed和watch的区别
7. Vue3组件通信方式
8. v-for为什么不能用index当key
9. Pinia和Vuex的区别
10. Vue和React的核心区别
11. 前端怎么处理大模型流式输出
12. TypeScript在Vue3中怎么用
13. useEffect的依赖数组怎么用
14. React Hooks了解多少

### 自查清单

- [ ] 能手写 Partial/Pick/Omit 的实现
- [ ] 能画出Vue3响应式原理的Proxy→依赖收集→触发更新流程图
- [ ] 能讲清ref为什么需要.value
- [ ] 能列举至少五种Vue3组件通信方式
- [ ] 能讲清Pinia相比Vuex的三个改进
- [ ] 能写出一个前端处理SSE流式输出的示例
- [ ] 能用Vue和React各写一个简单的计数器组件（对比语法差异）
- [ ] 能讲清Vue3 diff算法的三个优化点

---

> [← 返回学习框架](./学习框架.md)
