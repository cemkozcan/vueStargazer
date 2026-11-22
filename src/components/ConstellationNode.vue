<script setup>
import { computed } from 'vue'

const props = defineProps({
  constellation: { type: Object, required: true },
  active: { type: Boolean, default: false },
})

const emit = defineEmits(['select', 'edit-star', 'edit-constellation'])

const bounds = computed(() => {
  const stars = props.constellation.stars
  if (!stars.length) return { x: 0, y: 0, radius: 48 }

  const center = stars.reduce(
    (sum, star) => ({ x: sum.x + star.x / stars.length, y: sum.y + star.y / stars.length }),
    { x: 0, y: 0 },
  )
  const radius = Math.max(...stars.map((star) => Math.hypot(star.x - center.x, star.y - center.y))) + 34

  return { ...center, radius }
})

</script>

<template>
  <g
    class="constellation"
    :class="{ 'is-active': active }"
    :style="{ '--constellation-color': constellation.color }"
    :transform="`translate(${constellation.x} ${constellation.y})`"
  >
    <template v-if="active">
      <circle
        class="constellation-boundary"
        :cx="bounds.x"
        :cy="bounds.y"
        :r="bounds.radius"
        aria-hidden="true"
        pointer-events="fill"
      />
      <g
        v-for="star in constellation.stars"
        :key="star.id"
        class="named-star"
        :style="{ '--star-color': constellation.color }"
        :data-star-id="star.id"
        role="button"
        tabindex="0"
        :aria-label="`Edit ${star.name}`"
        @click.stop="$emit('edit-star', { star, constellationId: constellation.id })"
        @keydown.enter.prevent="$emit('edit-star', { star, constellationId: constellation.id })"
        @keydown.space.prevent="$emit('edit-star', { star, constellationId: constellation.id })"
      >
        <circle class="star-halo" :cx="star.x" :cy="star.y" :r="star.size + 10" />
        <circle class="star-core" :cx="star.x" :cy="star.y" :r="star.size" />
        <text class="star-label" :x="star.x + 13" :y="star.y - 12">{{ star.name }}</text>
      </g>
      <text
        class="expanded-title"
        :x="bounds.x"
        :y="bounds.y - bounds.radius - 40"
        role="button"
        tabindex="0"
        :aria-label="`Collapse ${constellation.name}`"
        aria-expanded="true"
        @click.stop="emit('select', constellation.id)"
        @keydown.enter.prevent="emit('select', constellation.id)"
        @keydown.space.prevent="emit('select', constellation.id)"
      >{{ constellation.name }}</text>
      <text
        class="expanded-description"
        :x="bounds.x"
        :y="bounds.y - bounds.radius - 20"
      >{{ constellation.description || constellation.subtitle }}</text>
      <g
        class="constellation-edit-trigger"
        :transform="`translate(${bounds.x + bounds.radius - 6} ${bounds.y - bounds.radius + 7})`"
        role="button"
        tabindex="0"
        :aria-label="`Edit ${constellation.name} details`"
        @click.stop="emit('edit-constellation', constellation.id)"
        @keydown.enter.prevent="emit('edit-constellation', constellation.id)"
        @keydown.space.prevent="emit('edit-constellation', constellation.id)"
      >
        <circle r="15" />
        <path d="M-4 4l1-4 6-6 3 3-6 6-4 1zm5-9l3 3" />
      </g>
    </template>
    <template v-else>
      <g
        class="constellation-expand-trigger"
        :transform="`translate(${bounds.x} ${bounds.y})`"
        role="button"
        tabindex="0"
        :aria-label="`${constellation.name}, ${constellation.stars.length} stars`"
        aria-expanded="false"
        @click.stop="emit('select', constellation.id)"
        @keydown.enter.prevent="emit('select', constellation.id)"
        @keydown.space.prevent="emit('select', constellation.id)"
      >
        <circle class="node-halo" r="32" />
        <circle class="node-core" r="5" />
        <text class="node-label" x="0" y="54">{{ constellation.name }}</text>
        <text class="node-subtitle" x="0" y="74">{{ constellation.subtitle }}</text>
      </g>
    </template>
  </g>
</template>