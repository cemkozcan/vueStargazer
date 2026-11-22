<script setup>
import { ref } from 'vue'

const props = defineProps({
  source: { type: Object, required: true },
  inConstellation: { type: Boolean, default: false },
})

const emit = defineEmits(['cancel', 'save'])
const starName = ref('')
const constellationName = ref('')
const description = ref('')

function save() {
  if (!starName.value.trim()) return
  if (!props.inConstellation && !constellationName.value.trim()) return

  emit('save', {
    name: starName.value.trim(),
    constellationName: constellationName.value.trim(),
    description: description.value.trim(),
  })
}
</script>

<template>
  <div class="dialog-backdrop" @click.self="emit('cancel')">
    <section class="star-dialog" role="dialog" aria-modal="true" aria-labelledby="linked-star-title">
      <div class="dialog-heading">
        <div>
          <p class="dialog-eyebrow">LINK FROM {{ source.name.toUpperCase() }}</p>
          <h2 id="linked-star-title">{{ inConstellation ? 'Create linked star' : 'Create a constellation' }}</h2>
        </div>
        <button class="icon-button" type="button" aria-label="Close" @click="emit('cancel')">
          <span aria-hidden="true">×</span>
        </button>
      </div>

      <form class="star-form" @submit.prevent="save">
        <template v-if="!inConstellation">
          <label class="field-label" for="linked-constellation-name">Constellation name</label>
          <input
            id="linked-constellation-name"
            v-model="constellationName"
            class="text-field"
            type="text"
            maxlength="40"
            autocomplete="off"
            required
            autofocus
          />
          <label class="field-label" for="linked-constellation-description">Description</label>
          <textarea
            id="linked-constellation-description"
            v-model="description"
            class="text-field description-field"
            maxlength="160"
            rows="2"
          ></textarea>
        </template>
        <label class="field-label" for="linked-star-name">New star name</label>
        <input
          id="linked-star-name"
          v-model="starName"
          class="text-field"
          type="text"
          maxlength="40"
          autocomplete="off"
          required
          :autofocus="inConstellation"
        />
        <p class="linked-star-note">The new star inherits {{ inConstellation ? 'this constellation' : 'the source star’s color' }}.</p>
        <div class="dialog-actions">
          <span class="action-spacer"></span>
          <button class="text-button" type="button" @click="emit('cancel')">Cancel</button>
          <button class="primary-button" type="submit">{{ inConstellation ? 'Create linked star' : 'Create constellation' }}</button>
        </div>
      </form>
    </section>
  </div>
</template>