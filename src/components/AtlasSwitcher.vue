<script setup>
import { onBeforeUnmount, onMounted, ref } from 'vue'

defineProps({
  atlases: { type: Array, required: true },
  activeAtlasId: { type: String, required: true },
})

const emit = defineEmits(['switch', 'create'])
const open = ref(false)
const creating = ref(false)
const atlasName = ref('')
const root = ref(null)

function closeOnOutsideClick(event) {
  if (!root.value?.contains(event.target)) {
    open.value = false
    creating.value = false
  }
}

onMounted(() => document.addEventListener('pointerdown', closeOnOutsideClick))
onBeforeUnmount(() => document.removeEventListener('pointerdown', closeOnOutsideClick))

function switchTo(id) {
  emit('switch', id)
  open.value = false
}

function createAtlas() {
  const name = atlasName.value.trim()
  if (!name) return
  emit('create', name)
  atlasName.value = ''
  creating.value = false
  open.value = false
}

function handleKeydown(event) {
  if (event.key === 'Escape') {
    open.value = false
    creating.value = false
  }
}
</script>

<template>
  <div ref="root" class="atlas-switcher" @keydown="handleKeydown">
    <button
      class="atlas-switcher-trigger"
      type="button"
      aria-haspopup="menu"
      :aria-expanded="open"
      @click="open = !open"
    >
      <span class="atlas-trigger-icon" aria-hidden="true">✳</span>
      <span class="atlas-trigger-name">{{ atlases.find((atlas) => atlas.id === activeAtlasId)?.name }}</span>
      <svg class="atlas-trigger-chevron" viewBox="0 0 12 12" aria-hidden="true">
        <path d="m2.5 4.5 3.5 3 3.5-3" />
      </svg>
    </button>

    <div v-if="open" class="atlas-switcher-menu" role="menu" aria-label="Atlases">
      <p class="atlas-menu-label">YOUR ATLASES</p>
      <button
        v-for="atlas in atlases"
        :key="atlas.id"
        class="atlas-menu-item"
        :class="{ 'is-current': atlas.id === activeAtlasId }"
        type="button"
        role="menuitemradio"
        :aria-checked="atlas.id === activeAtlasId"
        @click="switchTo(atlas.id)"
      >
        <span class="atlas-item-symbol" aria-hidden="true">✳</span>
        <span class="atlas-item-name">{{ atlas.name }}</span>
        <span v-if="atlas.id === activeAtlasId" class="atlas-current-mark" aria-hidden="true">✓</span>
      </button>

      <div class="atlas-menu-divider"></div>
      <form v-if="creating" class="atlas-create-form" @submit.prevent="createAtlas">
        <label class="atlas-menu-label" for="new-atlas-name">ATLAS NAME</label>
        <input
          id="new-atlas-name"
          v-model="atlasName"
          class="text-field atlas-name-field"
          type="text"
          maxlength="40"
          placeholder="Atlas name"
          autocomplete="off"
          required
          autofocus
        />
        <div class="atlas-create-actions">
          <button class="text-button" type="button" @click="creating = false">Cancel</button>
          <button class="primary-button" type="submit">Create atlas</button>
        </div>
      </form>
      <button v-else class="atlas-create-trigger" type="button" role="menuitem" @click="creating = true">
        <span aria-hidden="true">+</span>
        Create atlas
      </button>
    </div>
  </div>
</template>
