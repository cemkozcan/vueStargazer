<script setup>
import MarkdownIt from 'markdown-it'
import { computed, ref } from 'vue'

const props = defineProps({
  star: { type: Object, required: true },
  content: { type: String, default: '' },
})

const emit = defineEmits(['close', 'save', 'edit-details'])
const initialMarkdown = props.content || `# ${props.star.name}\n\n`
const markdown = ref(initialMarkdown)
const savedMarkdown = ref(initialMarkdown)
const activeTab = ref('write')
const confirmDiscard = ref(false)
const parser = new MarkdownIt({ html: false, linkify: true, typographer: true })
const previewHtml = computed(() => parser.render(markdown.value))
const fileName = computed(() => `${props.star.name.trim().replace(/[^a-z0-9-_]+/gi, '-').replace(/^-|-$/g, '') || 'star'}.md`)
const isDirty = computed(() => markdown.value !== savedMarkdown.value)

function save() {
  savedMarkdown.value = markdown.value
  emit('save', markdown.value)
}

function requestClose() {
  if (isDirty.value) {
    confirmDiscard.value = true
    return
  }
  emit('close')
}

function download() {
  const file = new Blob([markdown.value], { type: 'text/markdown;charset=utf-8' })
  const url = URL.createObjectURL(file)
  const link = document.createElement('a')
  link.href = url
  link.download = fileName.value
  link.hidden = true
  document.body.append(link)
  link.click()
  link.remove()
  window.setTimeout(() => URL.revokeObjectURL(url), 1000)
}
</script>

<template>
  <div class="dialog-backdrop note-backdrop" @click.self="requestClose">
    <section class="note-dialog" role="dialog" aria-modal="true" :aria-labelledby="'note-title'">
      <header class="note-header">
        <div class="note-heading">
          <span class="note-star-mark" :style="{ '--star-color': star.color }"></span>
          <div>
            <p class="dialog-eyebrow">STAR FIELDNOTE · {{ fileName }}</p>
            <h2 id="note-title">{{ star.name }}</h2>
          </div>
        </div>
        <div class="note-header-actions">
          <button class="text-button note-details-button" type="button" @click="emit('edit-details')">Star details</button>
          <button class="icon-button" type="button" aria-label="Close note" @click="requestClose">
            <span aria-hidden="true">×</span>
          </button>
        </div>
      </header>

      <div class="note-toolbar">
        <div class="note-tabs" role="tablist" aria-label="Markdown view">
          <button
            class="note-tab"
            :class="{ 'is-selected': activeTab === 'write' }"
            type="button"
            role="tab"
            :aria-selected="activeTab === 'write'"
            @click="activeTab = 'write'"
          >Write</button>
          <button
            class="note-tab"
            :class="{ 'is-selected': activeTab === 'preview' }"
            type="button"
            role="tab"
            :aria-selected="activeTab === 'preview'"
            @click="activeTab = 'preview'"
          >Preview</button>
        </div>
        <button class="text-button download-note-button" type="button" @click="download">Download .md</button>
      </div>

      <textarea
        v-if="activeTab === 'write'"
        v-model="markdown"
        class="markdown-editor"
        :aria-label="`Markdown note for ${star.name}`"
        spellcheck="false"
        autofocus
      ></textarea>
      <article v-else class="markdown-preview" v-html="previewHtml"></article>

      <footer class="note-footer">
        <span class="note-save-state">{{ markdown.length }} characters</span>
        <div class="dialog-actions">
          <button class="text-button" type="button" @click="requestClose">Cancel</button>
          <button class="primary-button" type="button" @click="save">Save note</button>
        </div>
      </footer>

      <div v-if="confirmDiscard" class="confirm-backdrop" @click.self="confirmDiscard = false">
        <section class="confirm-dialog" role="alertdialog" aria-modal="true" aria-labelledby="discard-note-title">
          <p class="dialog-eyebrow">UNSAVED FIELDNOTE</p>
          <h3 id="discard-note-title">Discard unsaved changes?</h3>
          <p class="confirm-copy">Your Markdown edits have not been saved.</p>
          <div class="dialog-actions">
            <button class="text-button" type="button" @click="confirmDiscard = false">Keep editing</button>
            <button class="delete-button" type="button" @click="emit('close')">Discard changes</button>
          </div>
        </section>
      </div>
    </section>
  </div>
</template>
