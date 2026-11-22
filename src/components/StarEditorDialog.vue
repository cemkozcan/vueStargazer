<script setup>
import { ref, watch } from 'vue'

const props = defineProps({
  star: { type: Object, required: true },
  mode: { type: String, default: 'edit' },
})

const emit = defineEmits(['cancel', 'save', 'delete'])
const draft = ref({ ...props.star })
const confirmDelete = ref(false)

watch(() => props.star, (star) => {
  draft.value = { ...star }
})

function save() {
  const name = draft.value.name.trim()
  if (!name) return
  emit('save', { ...draft.value, name })
}
</script>

<template>
  <div class="dialog-backdrop" @click.self="emit('cancel')">
    <section class="star-dialog" role="dialog" aria-modal="true" :aria-labelledby="'star-dialog-title'">
      <div class="dialog-heading">
        <div>
          <p class="dialog-eyebrow">STAR DETAILS</p>
          <h2 id="star-dialog-title">
            {{ mode === 'create-member' ? 'Create star in constellation' : mode === 'create' ? 'Create a star' : 'Edit star' }}
          </h2>
        </div>
        <button class="icon-button" type="button" aria-label="Close" @click="emit('cancel')">
          <span aria-hidden="true">×</span>
        </button>
      </div>

      <form class="star-form" @submit.prevent="save">
        <label class="field-label" for="star-name">Name</label>
        <input
          id="star-name"
          v-model="draft.name"
          class="text-field"
          type="text"
          maxlength="40"
          placeholder="e.g. Polaris"
          autocomplete="off"
          required
          autofocus
        />
        <template v-if="mode !== 'create-member' && !star.constellationId">
          <label class="field-label" for="star-color">Color</label>
          <div class="color-field">
            <input id="star-color" v-model="draft.color" type="color" aria-label="Star color" />
            <span>{{ draft.color }}</span>
          </div>
        </template>
        <div class="dialog-actions">
          <button v-if="mode === 'edit'" class="delete-button" type="button" @click="confirmDelete = true">
            Delete
          </button>
          <span class="action-spacer"></span>
          <button class="text-button" type="button" @click="emit('cancel')">Cancel</button>
          <button class="primary-button" type="submit">{{ mode === 'edit' ? 'Save' : 'Create' }}</button>
        </div>
      </form>

      <div v-if="confirmDelete" class="confirm-backdrop" @click.self="confirmDelete = false">
        <section class="confirm-dialog" role="alertdialog" aria-modal="true" aria-labelledby="delete-title">
          <p class="dialog-eyebrow">CONFIRM DELETE</p>
          <h3 id="delete-title">Delete {{ draft.name || 'this star' }}?</h3>
          <p class="confirm-copy">This star will be removed from your map.</p>
          <div class="dialog-actions">
            <button class="text-button" type="button" @click="confirmDelete = false">Keep star</button>
            <button class="delete-button" type="button" @click="emit('delete')">Delete star</button>
          </div>
        </section>
      </div>
    </section>
  </div>
</template>