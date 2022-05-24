<script setup lang="ts">
import GameSpace from "@/components/GameSpace.vue";

defineProps<{
  subgroup: string;
  showWarts: boolean;
}>();
</script>

<template>
  <main>
    <GameSpace :subgroup="subgroup" :showWarts="showWarts" />
  </main>
</template>
