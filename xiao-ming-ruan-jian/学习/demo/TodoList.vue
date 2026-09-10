<!--
  TodoList.vue — 父组件
  演示: 使用子组件的 props / emits / provide / v-model
-->
<template>
  <div class="todo-list">
    <h2>📋 待办列表</h2>

    <!-- 新增输入框 -->
    <div class="add-row">
      <input
        v-model="newText"
        placeholder="输入新待办..."
        @keyup.enter="addTodo"
      />
      <button @click="addTodo">添加</button>
    </div>

    <!--
      ============================================================
      使用子组件 TodoItem
      ============================================================
      父传子（props）：
        :todo       → 传入必传 prop
        :index      → 传入可选 prop

      子传父（emits）：
        @edit       → 监听子组件的 emit('edit', id)
        @delete     → 监听子组件的 emit('delete', id)
        @toggle     → 监听子组件的 emit('toggle', id, done)
    -->
    <TodoItem
      v-for="(item, idx) in todos"
      :key="item.id"
      :todo="item"
      :index="idx"
      @edit="handleEdit"
      @delete="handleDelete"
      @toggle="handleToggle"
    />

    <p v-if="todos.length === 0" class="empty">暂无待办事项</p>
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import TodoItem from './TodoItem.vue';

// ============================================================
// provide — 向所有后代组件提供数据
// ============================================================
// TodoItem 及其任意深度的子组件都可以用 inject('theme') 读取。
// 这里用 provide 传下去的是静态值，如需响应式用 ref 包一层。
import { provide } from 'vue';
provide('theme', 'dark');

// ============================================================
// 响应式状态
// ============================================================
interface Todo {
  id: number;
  text: string;
  done: boolean;
}

const todos = ref<Todo[]>([
  { id: 1, text: '学习 defineProps', done: true },
  { id: 2, text: '学习 defineEmits', done: false },
  { id: 3, text: '学习 provide/inject', done: false },
]);

const newText = ref('');
let nextId = 4;

// ============================================================
// 方法 — 响应子组件的 emit 事件
// ============================================================

function addTodo() {
  const text = newText.value.trim();
  if (!text) return;
  todos.value.push({ id: nextId++, text, done: false });
  newText.value = '';
}

/** 子组件 emit('edit', id) → 父组件收到后执行此方法 */
function handleEdit(id: number) {
  const newText = prompt('修改待办内容：');
  if (newText) {
    const item = todos.value.find(t => t.id === id);
    if (item) item.text = newText;
  }
}

/** 子组件 emit('delete', id) → 父组件删除对应项 */
function handleDelete(id: number) {
  todos.value = todos.value.filter(t => t.id !== id);
}

/** 子组件 emit('toggle', id, done) → 父组件更新完成状态 */
function handleToggle(id: number, done: boolean) {
  const item = todos.value.find(t => t.id === id);
  if (item) item.done = done;
}
</script>

<style scoped>
.todo-list {
  max-width: 520px;
  margin: 40px auto;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
}
h2 { margin-bottom: 16px; }
.add-row {
  display: flex;
  gap: 8px;
  margin-bottom: 16px;
}
.add-row input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 14px;
}
.add-row button {
  padding: 8px 18px;
  background: #4a90d9;
  color: #fff;
  border: none;
  border-radius: 6px;
  cursor: pointer;
}
.empty { color: #999; text-align: center; margin-top: 40px; }
</style>
