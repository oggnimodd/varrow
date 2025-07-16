<script setup lang="ts">
import { useMouse } from "@vueuse/core";

const { x, y } = useMouse();
const counterStores = useCounterStore();
const { message } = useHello();

const { count, doubleCount } = storeToRefs(counterStores);
const { increment } = counterStores;
</script>

<template>
  <div>
    <h1>{{ count }}</h1>
    <h2>{{ doubleCount }}</h2>
    <button @click="() => increment()">Increment</button>
    <p>Mouse position is at: {{ x }}, {{ y }}</p>
    <p>{{ message }}</p>
  </div>
</template>
