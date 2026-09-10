<!--
  TodoItem.vue — 子组件
  演示: defineProps / defineEmits / v-model / computed
-->
<template>
  <div class="todo-item" :class="{ completed: todo.done }">
    <!-- 1. 来自父组件的 prop：显示文本 -->
    <span class="todo-text">{{ todo.text }}</span>

    <!-- 2. 来自祖父组件的 inject -->
    <span class="todo-theme">主题: {{ theme }}</span>

    <!-- 3. v-model（本质是 :modelValue + @update:modelValue） -->
    <input
      type="checkbox"
      :checked="todo.done"
      @change="onToggle"
    />

    <!-- 4. 子传父：emit 事件 -->
    <button @click="onEdit">✏️ 编辑</button>
    <button @click="onDelete">🗑️ 删除</button>
  </div>
</template>

<script setup lang="ts">
import { computed, inject, type Ref } from 'vue';

// ============================================================
// defineProps — 声明父组件传入的数据
// ============================================================
// 运行时不需 import，编译宏由 @vue/compiler-sfc 在构建时处理。
// ? 表示可选（父组件可以不传），无 ? 表示必传。
interface Props {
  todo: {
    id: number;
    text: string;
    done: boolean;
  };
  index?: number;       // 可选 prop
  extraStyle?: string;  // 可选 prop
}

const props = defineProps<Props>();

// 也可以用 withDefaults 给可选 prop 设置默认值：
// const props = withDefaults(defineProps<Props>(), {
//   index: 0,
//   extraStyle: '',
// });

// ============================================================
// defineEmits — 声明子组件可以向父组件触发的事件
// ============================================================
// 函数签名写法：每个签名定义一个合法的事件名 + 参数类型。
const emit = defineEmits<{
  //      事件名       参数类型
  (e: 'edit',   id: number): void;
  (e: 'delete', id: number): void;
  (e: 'toggle', id: number, done: boolean): void;
}>();

// ============================================================
// inject — 从任意祖先组件注入数据（跨层级通信）
// ============================================================
const theme = inject<string>('theme', 'light'); // 第二个参数是默认值

// ============================================================
// 事件处理函数
// ============================================================
function onEdit() {
  emit('edit', props.todo.id);
}

function onDelete() {
  emit('delete', props.todo.id);
}

function onToggle(event: Event) {
  const checked = (event.target as HTMLInputElement).checked;
  emit('toggle', props.todo.id, checked);
}
</script>

<style scoped>
.todo-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 8px 12px;
  border: 1px solid #ddd;
  border-radius: 6px;
  margin-bottom: 8px;
}
.completed .todo-text {
  text-decoration: line-through;
  color: #999;
}
.todo-theme {
  font-size: 11px;
  color: #888;
  background: #f0f0f0;
  padding: 2px 6px;
  border-radius: 4px;
}
button {
  cursor: pointer;
  padding: 4px 10px;
  border: 1px solid #ccc;
  border-radius: 4px;
  background: #fff;
}
button:hover {
  background: #f5f5f5;
}
</style>
