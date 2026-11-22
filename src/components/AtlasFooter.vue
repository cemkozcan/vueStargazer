<script setup>
import starQuotes from '../data/starQuotes.json'

defineProps({
  constellations: { type: Array, required: true },
})

const emit = defineEmits(['focus-constellation'])
const quote = starQuotes[Math.floor(Math.random() * starQuotes.length)]
const romanYear = toRoman(new Date().getFullYear())

function toRoman(number) {
  const numerals = [
    [1000, 'M'], [900, 'CM'], [500, 'D'], [400, 'CD'],
    [100, 'C'], [90, 'XC'], [50, 'L'], [40, 'XL'],
    [10, 'X'], [9, 'IX'], [5, 'V'], [4, 'IV'], [1, 'I'],
  ]
  let remainder = number
  let result = ''
  for (const [value, numeral] of numerals) {
    while (remainder >= value) {
      result += numeral
      remainder -= value
    }
  }
  return result
}
</script>

<template>
  <footer class="bottom-bar">
    <p class="map-caption">{{ quote }}</p>
    <div class="legend" aria-label="Constellations">
      <button
        v-for="constellation in constellations"
        :key="constellation.id"
        class="legend-item"
        type="button"
        :aria-label="`Focus ${constellation.name}`"
        @click="emit('focus-constellation', constellation.id)"
      >
        <span class="legend-dot" :style="{ backgroundColor: constellation.color }"></span>
        {{ constellation.name }}
      </button>
    </div>
    <span class="map-edition">SKY ATLAS <span>—</span> {{ romanYear }}</span>
  </footer>
</template>