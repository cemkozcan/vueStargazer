<script setup>
import { ref } from 'vue'

defineProps({
  source: { type: Object, required: true },
  target: { type: Object, required: true },
})

const emit = defineEmits(['cancel', 'save'])
const name = ref('')
const description = ref('')

function save() {
  const normalizedName = name.value.trim()
  if (!normalizedName) return
  emit('save', { name: normalizedName, description: description.value.trim() })
}
</script>

<template>
  <div class="dialog-backdrop" @click.self="emit('cancel')">
    <section class="star-dialog" role="dialog" aria-modal="true" aria-labelledby="pair-dialog-title">
      <div class="dialog-heading">
        <div>
          <p class="dialog-eyebrow">LINK {{ source.name.toUpperCase() }} + {{ target.name.toUpperCase() }}</p>
          <h2 id="pair-dialog-title">Create a constellation</h2>
        </div>
        <button class="icon-button" type="button" aria-label="Close" @click="emit('cancel')">
          <span aria-hidden="true">×</span>
        </button>
      </div>

      <form class="star-form" @submit.prevent="save">
        <label class="field-label" for="pair-constellation-name">Constellation name</label>
        <input
          id="pair-constellation-name"
          v-model="name"
          class="text-field"
          type="text"
          maxlength="40"
          autocomplete="off"
          required
          autofocus
        />
        <label class="field-label" for="pair-constellation-description">Description</label>
        <textarea
          id="pair-constellation-description"
          v-model="description"
          class="text-field description-field"
          maxlength="160"
          rows="2"
        ></textarea>
        <p class="linked-star-note">Both existing stars will become members. No star will be created.</p>
        <div class="dialog-actions">
          <span class="action-spacer"></span>
          <button class="text-button" type="button" @click="emit('cancel')">Cancel</button>
          <button class="primary-button" type="submit">Create constellation</button>
        </div>
      </form>
    </section>
  </div>
</template>