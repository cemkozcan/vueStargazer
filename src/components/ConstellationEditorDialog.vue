<script setup>
import { ref } from 'vue'

const props = defineProps({
  constellation: { type: Object, required: true },
})

const emit = defineEmits(['cancel', 'save'])
const name = ref(props.constellation.name)
const description = ref(props.constellation.description ?? props.constellation.subtitle ?? '')

function save() {
  const normalizedName = name.value.trim()
  if (!normalizedName) return
  emit('save', {
    id: props.constellation.id,
    name: normalizedName,
    description: description.value.trim(),
  })
}
</script>

<template>
  <div class="dialog-backdrop" @click.self="emit('cancel')">
    <section class="star-dialog" role="dialog" aria-modal="true" aria-labelledby="constellation-dialog-title">
      <div class="dialog-heading">
        <div>
          <p class="dialog-eyebrow">CONSTELLATION DETAILS</p>
          <h2 id="constellation-dialog-title">Edit constellation</h2>
        </div>
        <button class="icon-button" type="button" aria-label="Close" @click="emit('cancel')">
          <span aria-hidden="true">×</span>
        </button>
      </div>

      <form class="star-form" @submit.prevent="save">
        <label class="field-label" for="constellation-name">Name</label>
        <input
          id="constellation-name"
          v-model="name"
          class="text-field"
          type="text"
          maxlength="40"
          autocomplete="off"
          required
          autofocus
        />
        <label class="field-label" for="constellation-description">Description</label>
        <textarea
          id="constellation-description"
          v-model="description"
          class="text-field description-field"
          maxlength="160"
          rows="3"
        ></textarea>
        <div class="dialog-actions">
          <span class="action-spacer"></span>
          <button class="text-button" type="button" @click="emit('cancel')">Cancel</button>
          <button class="primary-button" type="submit">Save</button>
        </div>
      </form>
    </section>
  </div>
</template>