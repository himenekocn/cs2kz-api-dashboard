<template>
  <div>
    <p class="my-2 font-medium">Mappers</p>
    <n-dynamic-input
      v-model:value="mappers"
      item-style="margin-bottom: 1rem;"
      :on-create="onCreateMapper"
      :min="1"
      #="{ index }"
    >
      <div class="flex gap-4">
        <n-input v-model:value="mappers[index]" placeholder="Steam ID" @keydown.enter.prevent />
      </div>
    </n-dynamic-input>
  </div>
</template>

<script setup lang="ts">
import { NInput, NDynamicInput } from "naive-ui"

const mappers = defineModel<string[]>("mappers", { required: true })

function onCreateMapper() {
  return ""
}
</script>
