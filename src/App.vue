<script setup>
import { computed, ref, watch } from 'vue'
import AtlasFooter from './components/AtlasFooter.vue'
import AtlasHeader from './components/AtlasHeader.vue'
import ConstellationEditorDialog from './components/ConstellationEditorDialog.vue'
import SolitaryPairDialog from './components/SolitaryPairDialog.vue'
import LinkedStarDialog from './components/LinkedStarDialog.vue'
import StarNoteDialog from './components/StarNoteDialog.vue'
import StarEditorDialog from './components/StarEditorDialog.vue'
import StarSidebar from './components/StarSidebar.vue'
import StarMap from './components/StarMap.vue'
import { constellations as initialConstellations } from './data/constellations.js'

const atlasStorageKey = 'stargazer.atlases.v1'
const legacyStorageKey = 'stargazer.catalog.v1'
const atlasStore = loadAtlasStore()
const atlases = ref(atlasStore.atlases)
const activeAtlasId = ref(atlasStore.activeAtlasId)
const activeAtlas = computed(
  () => atlases.value.find((atlas) => atlas.id === activeAtlasId.value) ?? atlases.value[0],
)
const constellations = computed({
  get: () => activeAtlas.value.constellations,
  set: (value) => { activeAtlas.value.constellations = value },
})
const freeStars = computed({
  get: () => activeAtlas.value.freeStars,
  set: (value) => { activeAtlas.value.freeStars = value },
})
const links = computed({
  get: () => activeAtlas.value.links,
  set: (value) => { activeAtlas.value.links = value },
})
const notes = computed({
  get: () => activeAtlas.value.notes,
  set: (value) => { activeAtlas.value.notes = value },
})
const activeId = ref(null)
const starMap = ref(null)
const moveMode = ref(false)
const editorStar = ref(null)
const noteStar = ref(null)
const editorMode = ref('edit')
const constellationEditor = ref(null)
const linkedStarRequest = ref(null)
const solitaryPairRequest = ref(null)
const showSidebar = ref(false)
const allStars = computed(() => [
  ...constellations.value.flatMap((constellation) =>
    constellation.stars.map((star) => ({
      ...star,
      color: constellation.color,
      group: constellation.name,
      constellationId: constellation.id,
    })),
  ),
  ...freeStars.value.map((star) => ({ ...star, group: 'Unassigned', constellationId: null })),
])
const selectedConstellation = computed(() =>
  constellations.value.find((constellation) => constellation.id === activeId.value),
)
const dialogOpen = computed(() => Boolean(
  editorStar.value
    || noteStar.value
    || constellationEditor.value
    || linkedStarRequest.value
    || solitaryPairRequest.value,
))

watch([atlases, activeAtlasId], () => {
  localStorage.setItem(
    atlasStorageKey,
    JSON.stringify({ atlases: atlases.value, activeAtlasId: activeAtlasId.value }),
  )
}, { deep: true })

function normalizeCatalog(catalog) {
  const normalizedConstellations = normalizeConstellations(
    catalog?.constellations ?? seedConstellations(),
  )
  return {
    constellations: normalizedConstellations,
    freeStars: catalog?.freeStars ?? [],
    links: Array.isArray(catalog?.links)
      ? catalog.links
      : linksFromConstellations(normalizedConstellations),
    notes: catalog?.notes ?? {},
  }
}

function loadAtlasStore() {
  try {
    const saved = JSON.parse(localStorage.getItem(atlasStorageKey))
    if (Array.isArray(saved?.atlases) && saved.atlases.length) {
      const normalizedAtlases = saved.atlases.map((atlas) => ({
        id: atlas.id ?? crypto.randomUUID(),
        name: atlas.name?.trim() || 'Untitled atlas',
        ...normalizeCatalog(atlas),
      }))
      return {
        atlases: normalizedAtlases,
        activeAtlasId: normalizedAtlases.some((atlas) => atlas.id === saved.activeAtlasId)
          ? saved.activeAtlasId
          : normalizedAtlases[0].id,
      }
    }

    const legacy = JSON.parse(localStorage.getItem(legacyStorageKey))
    if (legacy && Array.isArray(legacy.constellations)) {
      const migrated = {
        id: 'default-atlas',
        name: 'Night Atlas',
        ...normalizeCatalog(legacy),
      }
      const store = { atlases: [migrated], activeAtlasId: migrated.id }
      localStorage.setItem(atlasStorageKey, JSON.stringify(store))
      return store
    }
  } catch {
    localStorage.removeItem(atlasStorageKey)
  }

  const initialAtlas = {
    id: 'default-atlas',
    name: 'Night Atlas',
    ...normalizeCatalog(null),
  }
  return { atlases: [initialAtlas], activeAtlasId: initialAtlas.id }
}

function seedConstellations() {
  return structuredClone(initialConstellations).map((constellation) => ({
    ...constellation,
    stars: constellation.stars.map((star, index) => ({
      ...star,
      id: `${constellation.id}-${index}`,
    })),
  }))
}

function normalizeConstellations(items) {
  return items.map((constellation) => ({
    ...constellation,
    description: constellation.description ?? constellation.subtitle ?? '',
    stars: constellation.stars.map((star, index) => {
      const normalizedStar = { ...star, id: star.id ?? `${constellation.id}-${index}` }
      delete normalizedStar.color
      return normalizedStar
    }),
  }))
}

function linksFromConstellations(items) {
  return items.flatMap((constellation) =>
    (constellation.connections ?? []).map(([from, to], index) => ({
      id: `${constellation.id}-link-${index}`,
      sourceId: constellation.stars[from]?.id,
      targetId: constellation.stars[to]?.id,
    })).filter((link) => link.sourceId && link.targetId),
  )
}

function resetMap() {
  activeId.value = null
  starMap.value?.reset()
}

function focusConstellation(id) {
  if (!constellations.value.some((constellation) => constellation.id === id)) return
  activeId.value = id
  starMap.value?.focusConstellation(id)
}

function moveStar({ starId, position }) {
  const solitary = freeStars.value.find((star) => star.id === starId)
  if (solitary) {
    solitary.x = position.x
    solitary.y = position.y
    return
  }

  const constellation = constellations.value.find((item) =>
    item.stars.some((star) => star.id === starId),
  )
  const star = constellation?.stars.find((item) => item.id === starId)
  if (!star) return
  star.x = position.x - constellation.x
  star.y = position.y - constellation.y
}

function switchAtlas(atlasId) {
  if (atlasId === activeAtlasId.value || !atlases.value.some((atlas) => atlas.id === atlasId)) return
  activeAtlasId.value = atlasId
  activeId.value = null
  editorStar.value = null
  noteStar.value = null
  constellationEditor.value = null
  linkedStarRequest.value = null
  solitaryPairRequest.value = null
  showSidebar.value = false
  starMap.value?.reset()
}

function createAtlas(name) {
  const atlasName = name.trim()
  if (!atlasName) return
  const atlas = {
    id: crypto.randomUUID(),
    name: atlasName,
    constellations: [],
    freeStars: [],
    links: [],
    notes: {},
  }
  atlases.value.push(atlas)
  switchAtlas(atlas.id)
}

function createStar(position) {
  editorMode.value = 'create'
  editorStar.value = {
    id: crypto.randomUUID(),
    name: '',
    x: position.x,
    y: position.y,
    size: 6,
    color: '#f0c879',
  }
}

function createStarInConstellation({ position, constellationId }) {
  editorMode.value = 'create-member'
  const constellation = constellations.value.find((item) => item.id === constellationId)
  if (!constellation) return
  editorStar.value = {
    id: crypto.randomUUID(),
    name: '',
    x: position.x - constellation.x,
    y: position.y - constellation.y,
    size: 6,
    constellationId,
  }
}

function editStar({ star, constellationId }) {
  const editableStar = { ...star }
  delete editableStar.group
  delete editableStar.constellationId
  const constellation = constellations.value.find((item) => item.id === constellationId)
  noteStar.value = {
    ...editableStar,
    color: editableStar.color ?? constellation?.color ?? '#f0c879',
    constellationId,
  }
  if (notes.value[noteStar.value.id] === undefined) {
    notes.value[noteStar.value.id] = `# ${noteStar.value.name}\n\n`
  }
}

function editStarDetails() {
  if (!noteStar.value) return
  editorMode.value = 'edit'
  editorStar.value = { ...noteStar.value }
}

function saveStarNote(markdown) {
  if (!noteStar.value) return
  notes.value[noteStar.value.id] = markdown
  noteStar.value = null
}

function saveStar(star) {
  if (editorMode.value === 'create') {
    freeStars.value.push(star)
  } else if (editorMode.value === 'create-member') {
    const constellation = constellations.value.find((item) => item.id === star.constellationId)
    if (constellation) {
      const newStar = { ...star }
      delete newStar.constellationId
      delete newStar.color
      constellation.stars.push(newStar)
    }
  } else if (star.constellationId) {
    const constellation = constellations.value.find((item) => item.id === star.constellationId)
    const index = constellation?.stars.findIndex((item) => item.id === star.id)
    if (index >= 0) {
      const updatedStar = { ...star }
      delete updatedStar.constellationId
      delete updatedStar.color
      constellation.stars[index] = updatedStar
    }
  } else {
      const index = freeStars.value.findIndex((item) => item.id === star.id)
    if (index >= 0) freeStars.value[index] = star
  }
  if (noteStar.value?.id === star.id) {
    noteStar.value = { ...noteStar.value, name: star.name, color: star.color ?? noteStar.value.color }
  }
  editorStar.value = null
}

function deleteStar() {
  const { id, constellationId } = editorStar.value
  delete notes.value[id]
  if (noteStar.value?.id === id) noteStar.value = null
  links.value = links.value.filter((link) => link.sourceId !== id && link.targetId !== id)
  if (constellationId) {
    const constellation = constellations.value.find((item) => item.id === constellationId)
    const starIndex = constellation.stars.findIndex((item) => item.id === id)
    if (starIndex < 0) return
    constellation.stars.splice(starIndex, 1)
    constellation.connections = constellation.connections
      .filter(([from, to]) => from !== starIndex && to !== starIndex)
      .map(([from, to]) => [from > starIndex ? from - 1 : from, to > starIndex ? to - 1 : to])
  } else {
    freeStars.value = freeStars.value.filter((star) => star.id !== id)
  }
  editorStar.value = null
}

function linkStars({ sourceId, targetId }) {
  if (sourceId === targetId) return
  const existingIndex = links.value.findIndex((link) =>
    (link.sourceId === sourceId && link.targetId === targetId)
      || (link.sourceId === targetId && link.targetId === sourceId),
  )
  if (existingIndex >= 0) {
    links.value.splice(existingIndex, 1)
    for (const starId of [sourceId, targetId]) {
      moveToSolitaryIfIsolated(starId)
    }
    const emptyIds = new Set(constellations.value.filter((item) => item.stars.length === 0).map((item) => item.id))
    if (emptyIds.size) {
      constellations.value = constellations.value.filter((item) => !emptyIds.has(item.id))
      if (emptyIds.has(activeId.value)) activeId.value = null
    }
  } else {
    const sourceConstellation = constellations.value.find((constellation) =>
      constellation.stars.some((star) => star.id === sourceId),
    )
    const targetConstellation = constellations.value.find((constellation) =>
      constellation.stars.some((star) => star.id === targetId),
    )
    const sourceIndex = freeStars.value.findIndex((star) => star.id === sourceId)
    const targetIndex = freeStars.value.findIndex((star) => star.id === targetId)

    if (sourceIndex >= 0 && targetIndex >= 0) {
      solitaryPairRequest.value = { sourceId, targetId }
      return
    }

    if (sourceConstellation && targetIndex >= 0) {
      const target = { ...freeStars.value[targetIndex] }
      delete target.color
      target.x -= sourceConstellation.x
      target.y -= sourceConstellation.y
      freeStars.value.splice(targetIndex, 1)
      sourceConstellation.stars.push(target)
    } else if (targetConstellation && sourceIndex >= 0) {
      const source = { ...freeStars.value[sourceIndex] }
      delete source.color
      source.x -= targetConstellation.x
      source.y -= targetConstellation.y
      freeStars.value.splice(sourceIndex, 1)
      targetConstellation.stars.push(source)
    }

    links.value.push({ id: crypto.randomUUID(), sourceId, targetId })
  }
}

function saveSolitaryPairConstellation({ name, description }) {
  if (!solitaryPairRequest.value) return
  const { sourceId, targetId } = solitaryPairRequest.value
  const sourceIndex = freeStars.value.findIndex((star) => star.id === sourceId)
  const targetIndex = freeStars.value.findIndex((star) => star.id === targetId)
  if (sourceIndex < 0 || targetIndex < 0) {
    solitaryPairRequest.value = null
    return
  }

  const [source] = freeStars.value.splice(sourceIndex, 1)
  const [target] = freeStars.value.splice(freeStars.value.findIndex((star) => star.id === targetId), 1)
  const center = { x: (source.x + target.x) / 2, y: (source.y + target.y) / 2 }
  const constellationId = crypto.randomUUID()
  const members = [source, target].map((star) => ({
    id: star.id,
    name: star.name,
    x: star.x - center.x,
    y: star.y - center.y,
    size: star.size,
  }))

  constellations.value.push({
    id: constellationId,
    name: name.trim(),
    subtitle: description.trim(),
    description: description.trim(),
    x: center.x,
    y: center.y,
    color: source.color,
    stars: members,
    connections: [],
  })
  links.value.push({ id: crypto.randomUUID(), sourceId, targetId })
  activeId.value = constellationId
  solitaryPairRequest.value = null
}

function moveToSolitaryIfIsolated(starId) {
  const constellation = constellations.value.find((item) =>
    item.stars.some((star) => star.id === starId),
  )
  if (!constellation) return

  const constellationStarIds = new Set(constellation.stars.map((star) => star.id))
  const hasInternalLink = links.value.some((link) => {
    const otherId = link.sourceId === starId
      ? link.targetId
      : link.targetId === starId
        ? link.sourceId
        : null
    return otherId && otherId !== starId && constellationStarIds.has(otherId)
  })
  if (hasInternalLink) return

  const starIndex = constellation.stars.findIndex((star) => star.id === starId)
  const [star] = constellation.stars.splice(starIndex, 1)
  constellation.connections = (constellation.connections ?? [])
    .filter(([from, to]) => from !== starIndex && to !== starIndex)
    .map(([from, to]) => [from > starIndex ? from - 1 : from, to > starIndex ? to - 1 : to])
  freeStars.value.push({
    ...star,
    x: constellation.x + star.x,
    y: constellation.y + star.y,
    color: randomColorDifferentFrom(constellation.color),
  })
}

function randomColorDifferentFrom(color) {
  let candidate
  do {
    candidate = `#${Math.floor(Math.random() * 0x1000000).toString(16).padStart(6, '0')}`
  } while (candidate.toLowerCase() === color.toLowerCase())
  return candidate
}

function handleEmptySpace(position) {
  if (editorStar.value || noteStar.value || constellationEditor.value || linkedStarRequest.value || solitaryPairRequest.value) {
    editorStar.value = null
    noteStar.value = null
    constellationEditor.value = null
    linkedStarRequest.value = null
    solitaryPairRequest.value = null
    return
  }
  if (activeId.value) {
    activeId.value = null
    return
  }
  createStar(position)
}

function requestLinkedStar(request) {
  linkedStarRequest.value = request
}

function saveLinkedStar({ name, constellationName, description }) {
  const { sourceId, position } = linkedStarRequest.value
  const source = findStar(sourceId)
  if (!source) return

  const newStar = {
    id: crypto.randomUUID(),
    name: name.trim(),
    x: position.x,
    y: position.y,
    size: 6,
  }

  if (source.constellationId) {
    const constellation = constellations.value.find((item) => item.id === source.constellationId)
    newStar.x -= constellation.x
    newStar.y -= constellation.y
    constellation.stars.push(newStar)
    links.value.push({ id: crypto.randomUUID(), sourceId, targetId: newStar.id })
  } else {
    const constellationId = crypto.randomUUID()
    const center = { x: (source.x + position.x) / 2, y: (source.y + position.y) / 2 }
    const originalStar = {
      id: source.id,
      name: source.name,
      x: source.x - center.x,
      y: source.y - center.y,
      size: source.size,
    }
    newStar.x -= center.x
    newStar.y -= center.y
    const constellation = {
      id: constellationId,
      name: constellationName.trim(),
      subtitle: description.trim(),
      description: description.trim(),
      x: center.x,
      y: center.y,
      color: source.color,
      stars: [originalStar, newStar],
      connections: [],
    }
    freeStars.value = freeStars.value.filter((star) => star.id !== source.id)
    constellations.value.push(constellation)
    links.value.push({ id: crypto.randomUUID(), sourceId, targetId: newStar.id })
    activeId.value = constellationId
  }

  linkedStarRequest.value = null
}

function findStar(id) {
  for (const constellation of constellations.value) {
    const star = constellation.stars.find((item) => item.id === id)
    if (star) {
      return {
        ...star,
        x: constellation.x + star.x,
        y: constellation.y + star.y,
        color: constellation.color,
        constellationId: constellation.id,
      }
    }
  }
  return freeStars.value.find((star) => star.id === id)
}

function editConstellation(id) {
  const constellation = constellations.value.find((item) => item.id === id)
  if (!constellation) return
  constellationEditor.value = { ...constellation }
}

function saveConstellation(updated) {
  const constellation = constellations.value.find((item) => item.id === updated.id)
  if (!constellation) return
  constellation.name = updated.name.trim()
  constellation.description = updated.description.trim()
  constellation.subtitle = updated.description.trim()
  constellationEditor.value = null
}
</script>

<template>
  <main class="atlas-shell">
    <AtlasHeader
      :atlases="atlases"
      :active-atlas-id="activeAtlasId"
      :move-mode="moveMode"
      @reset="resetMap"
      @toggle-stars="showSidebar = !showSidebar"
      @toggle-move="moveMode = !moveMode"
      @switch-atlas="switchAtlas"
      @create-atlas="createAtlas"
    />
    <StarMap
      ref="starMap"
      :constellations="constellations"
      :free-stars="freeStars"
      :links="links"
      :active-id="activeId"
      :move-mode="moveMode"
      :dialog-open="dialogOpen"
      :selected-name="selectedConstellation?.name"
      @select="activeId = activeId === $event ? null : $event"
      @create-star="createStar"
      @create-star-in-constellation="createStarInConstellation"
      @empty-space="handleEmptySpace"
      @edit-star="editStar"
      @edit-constellation="editConstellation"
      @link-stars="linkStars"
      @create-linked-star="requestLinkedStar"
      @move-star="moveStar"
    />
    <AtlasFooter
      :constellations="constellations"
      @focus-constellation="focusConstellation"
    />
    <StarNoteDialog
      v-if="noteStar"
      :key="noteStar.id"
      :star="noteStar"
      :content="notes[noteStar.id]"
      @close="noteStar = null"
      @save="saveStarNote"
      @edit-details="editStarDetails"
    />
    <StarSidebar
      v-if="showSidebar"
      :stars="allStars"
      @close="showSidebar = false"
      @edit-star="editStar"
    />
    <StarEditorDialog
      v-if="editorStar"
      :key="editorStar.id"
      :star="editorStar"
      :mode="editorMode"
      @cancel="editorStar = null"
      @save="saveStar"
      @delete="deleteStar"
    />
    <ConstellationEditorDialog
      v-if="constellationEditor"
      :key="constellationEditor.id"
      :constellation="constellationEditor"
      @cancel="constellationEditor = null"
      @save="saveConstellation"
    />
    <LinkedStarDialog
      v-if="linkedStarRequest"
      :key="linkedStarRequest.sourceId"
      :source="findStar(linkedStarRequest.sourceId)"
      :in-constellation="Boolean(findStar(linkedStarRequest.sourceId)?.constellationId)"
      @cancel="linkedStarRequest = null"
      @save="saveLinkedStar"
    />
    <SolitaryPairDialog
      v-if="solitaryPairRequest"
      :key="solitaryPairRequest.sourceId + solitaryPairRequest.targetId"
      :source="freeStars.find((star) => star.id === solitaryPairRequest.sourceId)"
      :target="freeStars.find((star) => star.id === solitaryPairRequest.targetId)"
      @cancel="solitaryPairRequest = null"
      @save="saveSolitaryPairConstellation"
    />
  </main>
</template>
