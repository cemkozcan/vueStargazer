<script setup>
import { computed, onBeforeUnmount, onMounted, ref } from 'vue'
import ConstellationNode from './ConstellationNode.vue'
import { backgroundStars } from '../data/constellations.js'

const props = defineProps({
  constellations: { type: Array, required: true },
  freeStars: { type: Array, required: true },
  links: { type: Array, required: true },
  activeId: { type: String, default: null },
  selectedName: { type: String, default: '' },
  moveMode: { type: Boolean, default: false },
  dialogOpen: { type: Boolean, default: false },
})

const emit = defineEmits([
  'select',
  'create-star',
  'create-star-in-constellation',
  'empty-space',
  'edit-star',
  'edit-constellation',
  'link-stars',
  'create-linked-star',
  'move-star',
])
const offset = ref({ x: 0, y: 0 })
const zoom = ref(1)
const currentTime = ref(formatSystemTime())
const dragging = ref(false)
const dragged = ref(false)
const suppressClick = ref(false)
const pointerStart = ref({ x: 0, y: 0 })
const offsetStart = ref({ x: 0, y: 0 })
const linkDrag = ref(null)
const moveDrag = ref(null)
const linkPreview = ref(null)
const svgElement = ref(null)
let clockInterval

function formatSystemTime(date = new Date()) {
  const hours = String(date.getHours()).padStart(2, '0')
  const minutes = String(date.getMinutes()).padStart(2, '0')
  return `${hours}H ${minutes}M`
}

function updateSystemTime() {
  currentTime.value = formatSystemTime()
}

const activeConstellation = computed(() =>
  props.constellations.find((constellation) => constellation.id === props.activeId),
)

const activeBounds = computed(() => {
  const constellation = activeConstellation.value
  if (!constellation) return null
  const stars = constellation.stars
  if (!stars.length) return { x: constellation.x, y: constellation.y, radius: 48 }
  const center = stars.reduce(
    (sum, star) => ({ x: sum.x + star.x / stars.length, y: sum.y + star.y / stars.length }),
    { x: 0, y: 0 },
  )
  return {
    x: constellation.x + center.x,
    y: constellation.y + center.y,
    radius: Math.max(...stars.map((star) => Math.hypot(star.x - center.x, star.y - center.y))) + 34,
  }
})

const hiddenFreeStarIds = computed(() => new Set(props.freeStars.filter((star) => {
  const bounds = activeBounds.value
  return bounds && Math.hypot(star.x - bounds.x, star.y - bounds.y) <= bounds.radius
}).map((star) => star.id)))

const visibleFreeStars = computed(() => props.freeStars.filter((star) => !hiddenFreeStarIds.value.has(star.id)))

const linkLines = computed(() => props.links.flatMap((link) => {
  if (hiddenFreeStarIds.value.has(link.sourceId) || hiddenFreeStarIds.value.has(link.targetId)) return []
  const source = starPosition(link.sourceId)
  const target = starPosition(link.targetId)
  return source && target ? [{ ...link, x1: source.x, y1: source.y, x2: target.x, y2: target.y }] : []
}))

function pointerPosition(event) {
  return new DOMPoint(event.clientX, event.clientY).matrixTransform(
    svgElement.value.getScreenCTM().inverse(),
  )
}

function mapPosition(point) {
  return {
    x: (point.x - offset.value.x) / zoom.value,
    y: (point.y - offset.value.y) / zoom.value,
  }
}

function starPosition(id) {
  const solitary = props.freeStars.find((star) => star.id === id)
  if (solitary) return solitary
  for (const constellation of props.constellations) {
    const star = constellation.stars.find((item) => item.id === id)
    if (star) return { x: constellation.x + star.x, y: constellation.y + star.y }
  }
  return null
}

function starEditPayload(id) {
  for (const constellation of props.constellations) {
    const star = constellation.stars.find((item) => item.id === id)
    if (star) return { star, constellationId: constellation.id }
  }
  const star = props.freeStars.find((item) => item.id === id)
  return star ? { star, constellationId: null } : null
}

function startPan(event) {
  if (event.button !== 0) return
  const point = pointerPosition(event)

  const starTarget = event.target.closest('[data-star-id]')
  if (starTarget) {
    const sourceId = starTarget.getAttribute('data-star-id')
    const position = starPosition(sourceId)
    if (!position) return

    if (props.moveMode) {
      dragging.value = true
      dragged.value = false
      suppressClick.value = true
      pointerStart.value = { x: point.x, y: point.y }
      moveDrag.value = { sourceId }
      event.currentTarget.setPointerCapture(event.pointerId)
      event.preventDefault()
      return
    }

    dragging.value = true
    dragged.value = false
    suppressClick.value = true
    pointerStart.value = { x: point.x, y: point.y }
    linkDrag.value = { sourceId, position }
    linkPreview.value = { x1: position.x, y1: position.y, x2: position.x, y2: position.y }
    event.currentTarget.setPointerCapture(event.pointerId)
    event.preventDefault()
    return
  }

  if (event.target.closest('.constellation-expand-trigger, .expanded-title, .constellation-edit-trigger, .map-zoom-controls')) {
    return
  }

  dragging.value = true
  dragged.value = false
  pointerStart.value = { x: point.x, y: point.y }
  offsetStart.value = { ...offset.value }
  event.target.setPointerCapture(event.pointerId)
}

function pan(event) {
  if (!dragging.value) return
  const point = pointerPosition(event)
  if (moveDrag.value) {
    if (Math.hypot(point.x - pointerStart.value.x, point.y - pointerStart.value.y) > 4) {
      dragged.value = true
      emit('move-star', {
        starId: moveDrag.value.sourceId,
        position: mapPosition(point),
      })
    }
    return
  }
  if (linkDrag.value) {
    const position = mapPosition(point)
    linkPreview.value = { ...linkPreview.value, x2: position.x, y2: position.y }
    if (Math.hypot(point.x - pointerStart.value.x, point.y - pointerStart.value.y) > 4) dragged.value = true
    return
  }
  const deltaX = point.x - pointerStart.value.x
  const deltaY = point.y - pointerStart.value.y
  if (Math.abs(deltaX) + Math.abs(deltaY) > 4) dragged.value = true
  offset.value = {
    x: offsetStart.value.x + deltaX / zoom.value,
    y: offsetStart.value.y + deltaY / zoom.value,
  }
}

function endPan(event) {
  if (moveDrag.value) {
    if (!dragged.value) {
      const payload = starEditPayload(moveDrag.value.sourceId)
      if (payload) emit('edit-star', payload)
    }
  } else if (linkDrag.value) {
    const sourceId = linkDrag.value.sourceId
    const target = document.elementFromPoint(event.clientX, event.clientY)?.closest('[data-star-id]')
    const targetId = target?.getAttribute('data-star-id')
    if (targetId && targetId !== sourceId) {
      emit('link-stars', { sourceId, targetId })
    } else if (!targetId && dragged.value) {
      emit('create-linked-star', { sourceId, position: mapPosition(pointerPosition(event)) })
    } else if (targetId === sourceId && !dragged.value) {
      const payload = starEditPayload(sourceId)
      if (payload) emit('edit-star', payload)
    }
  }
  const wasStarGesture = Boolean(linkDrag.value || moveDrag.value)
  linkDrag.value = null
  moveDrag.value = null
  linkPreview.value = null
  dragging.value = false
  if (wasStarGesture) {
    window.setTimeout(() => { suppressClick.value = false }, 0)
  }
  requestAnimationFrame(() => {
    dragged.value = false
  })
}

function cancelPan() {
  linkDrag.value = null
  moveDrag.value = null
  linkPreview.value = null
  dragging.value = false
  dragged.value = false
  suppressClick.value = false
}

function selectConstellation(id) {
  if (dragged.value) return
  emit('select', id)
}

function handleEmptyClick(event) {
  if (
    dragged.value
    || suppressClick.value
    || event.target.closest('[data-star-id], .constellation-expand-trigger, .expanded-title, .constellation-edit-trigger, .map-zoom-controls')
  ) return
  const position = mapPosition(pointerPosition(event))
  emit('empty-space', position)
}

function collapseConstellationFromContext(event) {
  if (!activeConstellation.value || !activeBounds.value) return
  const position = mapPosition(pointerPosition(event))
  if (Math.hypot(position.x - activeBounds.value.x, position.y - activeBounds.value.y) <= activeBounds.value.radius) {
    emit('select', activeConstellation.value.id)
  }
}

function zoomBy(amount) {
  const nextZoom = Math.min(2.5, Math.max(0.65, zoom.value + amount))
  const center = { x: 800, y: 500 }
  const mapCenter = {
    x: (center.x - offset.value.x) / zoom.value,
    y: (center.y - offset.value.y) / zoom.value,
  }
  offset.value = {
    x: center.x - mapCenter.x * nextZoom,
    y: center.y - mapCenter.y * nextZoom,
  }
  zoom.value = nextZoom
}

function zoomAtPointer(event) {
  if (props.dialogOpen) return
  event.preventDefault()

  const point = pointerPosition(event)
  const nextZoom = Math.min(2.5, Math.max(0.65, zoom.value * Math.exp(-event.deltaY * 0.001)))
  if (nextZoom === zoom.value) return

  const mapPoint = {
    x: (point.x - offset.value.x) / zoom.value,
    y: (point.y - offset.value.y) / zoom.value,
  }
  offset.value = {
    x: point.x - mapPoint.x * nextZoom,
    y: point.y - mapPoint.y * nextZoom,
  }
  zoom.value = nextZoom
}

function reset() {
  offset.value = { x: 0, y: 0 }
  zoom.value = 1
}

function focusConstellation(id) {
  const constellation = props.constellations.find((item) => item.id === id)
  if (!constellation) return
  const center = constellation.stars.length
    ? constellation.stars.reduce(
      (sum, star) => ({
        x: sum.x + star.x / constellation.stars.length,
        y: sum.y + star.y / constellation.stars.length,
      }),
      { x: 0, y: 0 },
    )
    : { x: 0, y: 0 }
  const worldCenter = { x: constellation.x + center.x, y: constellation.y + center.y }
  offset.value = {
    x: 800 - worldCenter.x * zoom.value,
    y: 500 - worldCenter.y * zoom.value,
  }
}

defineExpose({ reset, focusConstellation })

onMounted(() => {
  svgElement.value.addEventListener('wheel', zoomAtPointer, { passive: false })
  clockInterval = window.setInterval(updateSystemTime, 1000)
})

onBeforeUnmount(() => {
  svgElement.value?.removeEventListener('wheel', zoomAtPointer)
  window.clearInterval(clockInterval)
})
</script>

<template>
  <section
    class="map-viewport"
    :class="{ 'is-moving-mode': moveMode }"
    aria-label="Interactive star map"
  >
    <svg
      ref="svgElement"
      class="star-map"
      :class="{ 'is-dragging': dragging, 'is-moving-stars': moveMode }"
      viewBox="0 0 1600 1000"
      preserveAspectRatio="xMidYMid slice"
      role="img"
      aria-label="Star map with three clickable constellations"
      @pointerdown="startPan"
      @pointermove="pan"
      @pointerup="endPan"
      @pointercancel="cancelPan"
      @click="handleEmptyClick"
      @contextmenu.prevent="collapseConstellationFromContext"
    >
      <defs>
        <pattern
          id="starfield-pattern"
          patternUnits="userSpaceOnUse"
          width="1600"
          height="1000"
          :patternTransform="`translate(${offset.x} ${offset.y}) scale(${zoom})`"
        >
          <circle
            v-for="(star, index) in backgroundStars"
            :key="`field-${index}`"
            class="background-star"
            :cx="star.x"
            :cy="star.y"
            :r="star.size"
            :opacity="star.opacity"
          />
        </pattern>
      </defs>
      <rect
        class="starfield-background"
        x="0"
        y="0"
        width="1600"
        height="1000"
        fill="url(#starfield-pattern)"
        pointer-events="none"
      />
      <g :transform="`translate(${offset.x} ${offset.y}) scale(${zoom})`">
        <line
          v-for="link in linkLines"
          :key="link.id"
          class="star-link"
          :x1="link.x1"
          :y1="link.y1"
          :x2="link.x2"
          :y2="link.y2"
        />
        <line
          v-if="linkPreview"
          class="star-link link-preview"
          :x1="linkPreview.x1"
          :y1="linkPreview.y1"
          :x2="linkPreview.x2"
          :y2="linkPreview.y2"
        />

        <g
          v-for="star in visibleFreeStars"
          :key="star.id"
          class="free-star"
          :style="{ '--constellation-color': star.color }"
          :transform="`translate(${star.x} ${star.y})`"
          :data-star-id="star.id"
          role="button"
          tabindex="0"
          :aria-label="`${star.name}, star`"
          @click.stop="emit('edit-star', { star, constellationId: null })"
          @keydown.enter.prevent="emit('edit-star', { star, constellationId: null })"
          @keydown.space.prevent="emit('edit-star', { star, constellationId: null })"
        >
          <circle class="free-star-halo" r="18" />
          <circle class="free-star-core" :r="star.size" />
          <text class="free-star-label" x="12" y="-12">{{ star.name }}</text>
        </g>

        <ConstellationNode
          v-for="constellation in constellations"
          :key="constellation.id"
          :constellation="constellation"
          :active="activeId === constellation.id"
          @select="selectConstellation"
          @edit-star="emit('edit-star', $event)"
          @edit-constellation="emit('edit-constellation', $event)"
        />
      </g>
    </svg>

    <div v-if="moveMode" class="moving-mode-indicator" role="status" aria-live="polite">
      <span class="moving-mode-dot" aria-hidden="true"></span>
      <span>Moving stars</span>
    </div>
    <div v-if="activeId" class="selection-index" :class="{ 'selection-index-below-mode': moveMode }" aria-live="polite">
      <span class="selection-kicker">SELECTED PATTERN</span>
      <span class="selection-name">{{ selectedName }}</span>
    </div>
    <div class="map-coordinates" aria-label="Local time">NORTH SKY <span>·</span> {{ currentTime }}</div>
    <div class="map-zoom-controls" aria-label="Map zoom controls" @pointerdown.stop>
      <button type="button" aria-label="Zoom out" :disabled="zoom <= 0.65" @click="zoomBy(-0.2)">−</button>
      <span>{{ Math.round(zoom * 100) }}%</span>
      <button type="button" aria-label="Zoom in" :disabled="zoom >= 2.5" @click="zoomBy(0.2)">+</button>
    </div>
  </section>
</template>