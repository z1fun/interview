<!--
  App.vue — 入口组件
  演示三种父子通信模式的完整用法
-->
<template>
  <div class="app">
    <!-- ============================================================
        场景一：普通 props + emits（父→子→父）
        父传子: :todo / :index
        子传父: @edit / @delete / @toggle
    ============================================================ -->
    <TodoList />

    <hr />

    <!-- ============================================================
        场景二：v-model 双向绑定（本质还是 props + emits）
        v-model="rating" 等价于:
          :modelValue="rating"
          @update:modelValue="(val) => rating = val"
    ============================================================ -->
    <div class="rating-demo">
      <h3>⭐ 给这篇文章评分</h3>
      <RatingInput v-model="rating" />
      <p>当前评分: {{ rating }} 分</p>
    </div>

    <!-- ============================================================
        场景三：多个 v-model
        父组件可以绑定 modelValue 以外的 prop:
          v-model:title   → props.title + emit('update:title')
          v-model:content → props.content + emit('update:content')
    ============================================================ -->
  </div>
</template>

<script setup lang="ts">
import { ref } from 'vue';
import TodoList from './TodoList.vue';
import RatingInput from './RatingInput.vue';

const rating = ref(3); // v-model 绑定的值
</script>

<style>
.app { max-width: 600px; margin: 0 auto; padding: 20px; font-family: sans-serif; }
hr { margin: 40px 0; border: none; border-top: 1px solid #eee; }
.rating-demo { text-align: center; }
.rating-demo h3 { margin-bottom: 12px; }
</style>
