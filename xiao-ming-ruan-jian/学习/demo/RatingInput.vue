<!--
  RatingInput.vue — 演示 v-model 本质是 defineProps + defineEmits 的语法糖

  父组件写 <RatingInput v-model="score" /> 等价于：
  <RatingInput :modelValue="score" @update:modelValue="(val) => score = val" />

  也可以有多个 v-model：
  <RatingInput v-model:score="s" v-model:comment="c" />
-->
<template>
  <div class="rating">
    <span
      v-for="star in 5"
      :key="star"
      class="star"
      :class="{ filled: star <= modelValue }"
      @click="setRating(star)"
    >
      ★
    </span>
    <span class="label">{{ labels[modelValue] ?? '未评分' }}</span>
  </div>
</template>

<script setup lang="ts">
// ============================================================
// v-model 的单向数据流拆解
// ============================================================
// v-model 不是双向绑定黑魔法，而是——
//   父→子（props）:   子组件接收 :modelValue
//   子→父（emits）:    子组件 emit('update:modelValue', newValue)
//
// 所以 defineProps + defineEmits 完全覆盖了 v-model。

const props = defineProps<{
  modelValue: number;      // v-model 绑定的值（默认 prop 名叫 modelValue）
}>();

const emit = defineEmits<{
  // v-model 要求 emit 名必须是 'update:modelValue'
  (e: 'update:modelValue', value: number): void;
}>();

// ============================================================
// 子组件内部只 emit，不修改 props
// ============================================================
// 单向数据流原则：props 是只读的，子组件永远不直接改 props.modelValue
// 而是通过 emit('update:modelValue', ...) 通知父组件去改。

const labels: Record<number, string> = {
  1: '很差',
  2: '一般',
  3: '不错',
  4: '很好',
  5: '超棒',
};

function setRating(star: number) {
  emit('update:modelValue', star);
}
</script>

<style scoped>
.rating {
  display: inline-flex;
  align-items: center;
  gap: 2px;
}
.star {
  font-size: 28px;
  color: #ddd;
  cursor: pointer;
  transition: color 0.15s;
  user-select: none;
}
.star.filled { color: #f5a623; }
.star:hover { color: #f5a623; }
.label {
  margin-left: 10px;
  font-size: 14px;
  color: #666;
}
</style>
