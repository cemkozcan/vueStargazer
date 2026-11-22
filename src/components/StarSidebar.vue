<script setup>
defineProps({
  stars: { type: Array, required: true },
})

const emit = defineEmits(['close', 'edit-star'])
</script>

<template>
  <div class="sidebar-layer">
    <button class="sidebar-scrim" type="button" aria-label="Close patterns" @click="emit('close')"></button>
    <aside class="star-sidebar" aria-label="All stars">
      <header class="sidebar-header">
        <div>
          <p class="dialog-eyebrow">STAR CATALOG</p>
          <h2>Patterns <span>{{ stars.length }}</span></h2>
        </div>
        <button class="icon-button" type="button" aria-label="Close patterns" @click="emit('close')">
          <span aria-hidden="true">×</span>
        </button>
      </header>
      <ul class="star-list">
        <li v-for="star in stars" :key="star.id">
          <button
            class="star-list-item"
            type="button"
            @click="emit('edit-star', { star, constellationId: star.constellationId })"
          >
            <span class="catalog-star-dot" :style="{ '--star-color': star.color }"></span>
            <span class="catalog-star-copy">
              <span class="catalog-star-name">{{ star.name }}</span>
              <span class="catalog-star-group">{{ star.group }}</span>
            </span>
            <span class="catalog-edit-mark" aria-hidden="true">↗</span>
          </button>
        </li>
      </ul>
    </aside>
  </div>
</template>