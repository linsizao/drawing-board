<template>
  <div class="py-12">
    <div class="max-w-7xl mx-auto px-4 sm:px-6 lg:px-8">
      <div class="bg-white overflow-hidden shadow-sm sm:rounded-lg">
        <div class="p-6 bg-white border-b border-gray-200">
          <h2 class="text-2xl font-bold mb-4">涂鸦画板</h2>
          <p class="text-gray-600 mb-6">使用鼠标或触摸屏进行绘画</p>
          <DrawingBoard />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import DrawingBoard from '../components/DrawingBoard.vue'
</script>