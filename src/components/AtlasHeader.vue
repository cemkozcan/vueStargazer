<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'
import AtlasSwitcher from './AtlasSwitcher.vue'

defineProps({
  atlases: { type: Array, required: true },
  activeAtlasId: { type: String, required: true },
  moveMode: { type: Boolean, default: false },
})

const emit = defineEmits(['reset', 'toggle-stars', 'toggle-move', 'switch-atlas', 'create-atlas'])
const mobileMenuOpen = ref(false)
const mobileMenuRoot = ref(null)

function closeOnOutsideClick(event) {
  if (!mobileMenuRoot.value?.contains(event.target)) mobileMenuOpen.value = false
}

function handleKeydown(event) {
  if (event.key === 'Escape') mobileMenuOpen.value = false
}

function resetFromMenu() {
  emit('reset')
  mobileMenuOpen.value = false
}

function toggleStarsFromMenu() {
  emit('toggle-stars')
  mobileMenuOpen.value = false
}

function toggleMoveFromMenu() {
  emit('toggle-move')
  mobileMenuOpen.value = false
}

function switchAtlasFromMenu(id) {
  emit('switch-atlas', id)
  mobileMenuOpen.value = false
}

function createAtlasFromMenu(name) {
  emit('create-atlas', name)
  mobileMenuOpen.value = false
}

onMounted(() => {
  document.addEventListener('pointerdown', closeOnOutsideClick)
  document.addEventListener('keydown', handleKeydown)
})

onBeforeUnmount(() => {
  document.removeEventListener('pointerdown', closeOnOutsideClick)
  document.removeEventListener('keydown', handleKeydown)
})
</script>

<template>
  <header class="topbar">
    <a class="wordmark" href="#" @click.prevent="$emit('reset')">
      <span class="wordmark-mark" aria-hidden="true">✳</span>
      <span>Stargazer</span>
    </a>
    <div class="desktop-header-controls">
      <div class="topbar-meta">
        <span class="map-status"><span class="status-dot"></span> FIELD NOTES / 01</span>
        <AtlasSwitcher
          :atlases="atlases"
          :active-atlas-id="activeAtlasId"
          @switch="emit('switch-atlas', $event)"
          @create="emit('create-atlas', $event)"
        />
        <button class="map-count" type="button" @click="emit('toggle-stars')">03 PATTERNS</button>
      </div>
      <div class="map-actions">
        <button
          class="move-mode-button"
          :class="{ 'is-active': moveMode }"
          type="button"
          :aria-pressed="moveMode"
          :aria-label="moveMode ? 'Turn off move mode' : 'Turn on move mode'"
          :title="moveMode ? 'Turn off move mode' : 'Move stars'"
          @click="emit('toggle-move')"
        >
          <svg viewBox="0 0 24 24" aria-hidden="true">
            <path d="M12 3v18M3 12h18M12 3l-2.5 2.5M12 3l2.5 2.5M21 12l-2.5-2.5M21 12l-2.5 2.5M12 21l-2.5-2.5M12 21l2.5-2.5M3 12l2.5-2.5M3 12l2.5 2.5" />
          </svg>
        </button>
        <button class="reset-button" type="button" aria-label="Reset map" title="Reset map" @click="emit('reset')">
          <svg viewBox="0 0 24 24" aria-hidden="true">
            <path d="M4.5 10a7.5 7.5 0 1 1 .8 5.7M4.5 10V4.5M4.5 10H10" />
          </svg>
        </button>
      </div>
    </div>
    <div ref="mobileMenuRoot" class="mobile-header-menu">
      <button
        class="burger-button"
        type="button"
        :aria-label="mobileMenuOpen ? 'Close menu' : 'Open menu'"
        :aria-expanded="mobileMenuOpen"
        @click="mobileMenuOpen = !mobileMenuOpen"
      >
        <svg viewBox="0 0 24 24" aria-hidden="true">
          <path v-if="!mobileMenuOpen" d="M4 7h16M4 12h16M4 17h16" />
          <path v-else d="m6 6 12 12M18 6 6 18" />
        </svg>
      </button>
      <nav v-if="mobileMenuOpen" class="burger-menu-panel" aria-label="Main menu">
        <p class="burger-menu-label">ATLAS CONTROLS</p>
        <AtlasSwitcher
          :atlases="atlases"
          :active-atlas-id="activeAtlasId"
          @switch="switchAtlasFromMenu"
          @create="createAtlasFromMenu"
        />
        <button class="burger-menu-action" type="button" @click="toggleStarsFromMenu">
          <span class="burger-action-symbol" aria-hidden="true">☷</span>
          <span>03 Patterns</span>
        </button>
        <button
          class="burger-menu-action"
          :class="{ 'is-active': moveMode }"
          type="button"
          :aria-pressed="moveMode"
          @click="toggleMoveFromMenu"
        >
          <span class="burger-action-symbol" aria-hidden="true">↗</span>
          <span>{{ moveMode ? 'Move mode on' : 'Move stars' }}</span>
        </button>
        <button class="burger-menu-action" type="button" @click="resetFromMenu">
          <span class="burger-action-symbol" aria-hidden="true">⟲</span>
          <span>Reset view</span>
        </button>
      </nav>
    </div>
  </header>
</template>