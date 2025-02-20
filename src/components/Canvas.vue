<script setup lang="ts">
import { computed, inject, nextTick, provide, reactive, useTemplateRef, watch, type WatchHandle } from 'vue'
import { usePointer } from '../composables/pointer'
import { useCanvas } from '../composables/canvas'
import { useRouter } from 'vue-router'
import { useArrowKeys, useKeymap } from '../composables/keys'
import { useSettings } from '../composables/settings'
import { useToaster } from '../composables/toaster'
import { useEventListener } from '@vueuse/core'
import { clone, distance, isTrackpad, midpoint, moveThreshold, onceChanged, usingInput } from '../utils'
import { copyCards, pasteOnCanvas } from '../clipboard'
import CanvasBackground from './CanvasBackground.vue'
import Card from './Card.vue'
import DrawSelection from './DrawSelection.vue'

const canvasRef = useTemplateRef('canvas-ref')
const cardRefs = useTemplateRef('card-refs')
const state = reactive({
	selecing: false,
	panning: false,
	pinching: false,
	pointerOver: false,
	gesture: {
		pointers: [] as PointerState[],
		initialZoom: 1,
		prevTranslate: { x: 0, y: 0 }
	},
	loading: false
})
const { pointer, pointers } = usePointer()
const selection = reactive<CanvasSelection>({
	cards: [],
	box: null,
	boxVisible: false,
	draw: false,
	clear: () => {
		selection.box = null
		selection.cards = []
	}
})
const canvas = useCanvas(canvasRef, cardRefs)
const { cards, createCard, deleteMany } = inject('board') as BoardContext
const router = useRouter()
const arrowKeys = useArrowKeys()
const settings = useSettings()
const { toast, untoast } = useToaster()
const cursor = computed(() => {
	if (state.panning && pointer.moved) return 'move'
	if (state.selecing) return 'crosshair'
	if (state.loading) return 'wait'

	return 'default'
})
const animating = computed(() => state.panning
	|| state.pinching
	|| canvas.smoothZoom !== canvas.zoom
	|| canvas.smoothScroll.x !== canvas.scroll.x
	|| canvas.smoothScroll.y !== canvas.scroll.y
)
const boxSelectionStyle = computed(() => {
	const boxRect = selection.box ?? new DOMRect()
	const translateX = canvas.smoothScroll.x + boxRect.left * canvas.smoothZoom
	const translateY = canvas.smoothScroll.y + boxRect.top * canvas.smoothZoom

	return {
		width: `${Math.abs(boxRect.width) * canvas.smoothZoom}px`,
		height: `${Math.abs(boxRect.height) * canvas.smoothZoom}px`,
		translate: `${translateX}px ${translateY}px`,
		opacity: selection.boxVisible ? 1 : 0,
		transition: `opacity ${selection.boxVisible ? 0 : .2}s`
	}
})
let clickAllowed = false
let longPressTimeout: ReturnType<typeof setTimeout>
let lastTrackpadTime = 0
let unwatchCanvas: WatchHandle
let unwatchPointerMove: WatchHandle
let unwatchPointerUp: WatchHandle
let unwatchGestureChange: WatchHandle

provide('card-refs', cardRefs)
provide('canvas', canvas)
provide('selection', selection)

// Stack cards in the order they were modified
watch(() => cards.map(card => card.modified), () => {
	cards.sort((a, b) => {
		// Sort boxes before other cards
		if (a.type === 'box' && b.type !== 'box') return -1
		if (a.type !== 'box' && b.type === 'box') return 1

		// Sort boxes by position
		if (a.type === 'box' && b.type === 'box')
			return (a.pos.x + a.pos.y) - (b.pos.x + b.pos.y)

		return a.modified - b.modified
	})
}, { immediate: true })

// Clear the selection when cards are removed
watch(() => cards.length, (length, oldLength) => {
	if (oldLength > length)
		selection.clear()
})

useKeymap({
	'Home': canvas.home,
	'End': canvas.overview,
	'CtrlMeta +': keyboardZoom,
	'CtrlMeta -': keyboardZoom,
	'CtrlMeta A': () => selection.cards = cards,
	'CtrlMeta C': copySelectedCards,
	'CtrlMeta X': () => {
		copySelectedCards()
		deleteSelectedCards()
	},
	'CtrlMeta B': surroundWithBox,
	'Backspace': deleteSelectedCards,
	'Delete': deleteSelectedCards,
	'Escape': selection.clear,
	'CtrlMeta ,': () => router.push('/settings')
})

// Pan the canvas using the arrow keys
watch(arrowKeys, () => {
	if (state.panning)
		return

	// This is a funny way to convert booleans to numbers btw
	canvas.scrollSpeed = {
		x: +arrowKeys.left - +arrowKeys.right,
		y: +arrowKeys.up - +arrowKeys.down,
	}
	canvas.anyArrowKey = arrowKeys.left
		|| arrowKeys.right
		|| arrowKeys.up
		|| arrowKeys.down

	if (pointer.down && !pointer.moved)
		pointer.moved = true
})

// Allow typing anywhere on the canvas to create a new card
useEventListener('keydown', (event: KeyboardEvent) => {
	if (!settings.typeAnywhere || !state.pointerOver || pointer.down || usingInput())
		return

	if (event.ctrlKey || event.metaKey || (event.key !== 'Enter' && event.key.length !== 1))
		return

	event.preventDefault()
	selection.clear()

	createCard({
		new: true,
		pos: canvas.toCanvasPos(pointer),
		content: { text: event.key !== ' ' && event.key !== 'Enter' ? event.key : '' }
	})
})

// Add the paste listener to the document because non-input elements don't receive paste events
useEventListener('paste', onPaste)

function keyboardZoom(event: KeyboardEvent) {
	const delta = event.key === '+' ? -.2 : .2
	const canvasRect = canvas.ref.getBoundingClientRect()

	canvas.zoomTo(canvas.zoom * (1 - delta), {
		x: canvasRect.x + canvasRect.width / 2,
		y: canvasRect.y + canvasRect.height / 2
	})
	canvas.animate()
}

function copySelectedCards() {
	if (selection.cards.length)
		copyCards(selection.cards)
}

function deleteSelectedCards() {
	if (selection.cards.length)
		deleteMany(selection.cards)
}

function surroundWithBox() {
	if (!selection.cards.length)
		return

	const rect = canvas.getCardRects(selection.cards)!

	createCard({
		type: 'box',
		pos: {
			x: rect.x - 20,
			y: rect.y - 48
		},
		content: {
			label: `Box ${cards.filter(card => card.type === 'box').length + 1}`,
			width: rect.width + 40,
			height: rect.height + 40
		}
	})

	selection.clear()
}

function onPointerDown(event: PointerEvent) {
	// Wait until the event has bubbled to the listener on the document that updates the pointer state
	// Use one pointer to pan the canvas or select, and two pointers for pinch zoom
	// More than two pointers will be ignored
	onceChanged(pointers, () => {
		if (pointers.length === 1 && event.target === canvas.ref && !canvas.anyArrowKey) {
			if (event.ctrlKey || event.metaKey) {
				selection.clear()

				state.selecing = true
				unwatchCanvas = watch(canvas, onPointerMove)
			} else {
				state.panning = true
				clickAllowed = !usingInput()

				// Only allow long press if Ctrl / Meta is not pressed
				longPressTimeout = setTimeout(() => {
					if (navigator.vibrate)
						navigator.vibrate(50)

					selection.clear()

					state.panning = false
					state.selecing = true

					if (settings.selectionMode === 'draw')
						selection.draw = true

					unwatchCanvas = watch(canvas, onPointerMove)
				}, 500)
			}

			unwatchPointerMove = watch(pointer, onPointerMove)
			unwatchPointerUp = watch([
				() => pointer.down,
				() => pointers.length
			], onPointerUp, { flush: 'sync' })
		} else if (pointers.length === 2) {
			state.pinching = true
			state.gesture.pointers = pointers.map(clone)
			state.gesture.initialZoom = canvas.smoothZoom
			state.gesture.prevTranslate = { x: 0, y: 0 }
			clickAllowed = false
			unwatchGestureChange = watch(pointers, onGestureChange)
		}
	})
}

function onPointerMove() {
	if (!pointer.moved)
		return

	clearTimeout(longPressTimeout)

	// Pan the canvas
	if (state.panning) {
		canvas.scroll.x += pointer.movementX
		canvas.scroll.y += pointer.movementY
		canvas.smoothScroll.x = canvas.scroll.x
		canvas.smoothScroll.y = canvas.scroll.y
	}

	if (state.selecing) {
		canvas.edgeScroll()

		if (selection.draw)
			return

		// Init the selection box
		if (!selection.box) {
			const downPos = canvas.toCanvasPos(pointer.down as Pos)

			selection.box = new DOMRect(downPos.x, downPos.y, 0, 0)
			selection.boxVisible = true
		}

		// Update the selection box
		const pointerPos = canvas.toCanvasPos(pointer)

		selection.box = new DOMRect(
			selection.box.x,
			selection.box.y,
			pointerPos.x - selection.box.x,
			pointerPos.y - selection.box.y
		)
	}
}

function onPointerUp() {
	if (pointer.down && pointers.length === 1)
		return

	clearTimeout(longPressTimeout)

	if (state.panning && pointer.moved) {
		clickAllowed = false

		// Make scrolling feel like it has inertia on touch devices
		const velocity = { x: pointer.movementX, y: pointer.movementY }

		if (pointer.type === 'touch' && moveThreshold(velocity, { x: 0, y: 0 }, 4))
			canvas.kineticScroll(velocity)
	}

	if (state.selecing) {
		selection.boxVisible = false
		selection.draw = false
		clickAllowed = false

		canvas.stopEdgeScroll()
		unwatchCanvas()
	}

	state.panning = false
	state.selecing = false

	unwatchPointerMove()
	unwatchPointerUp()
}

function onGestureChange() {
	// Only track the pointers involved in the gesture
	const activePointers = pointers.filter(p => state.gesture.pointers.some(g => g.id === p.id))

	// Continue as long as the two involved pointers are still down
	if (activePointers.length !== 2)
		return onGestureEnd()

	const origin = midpoint(state.gesture.pointers)
	const currentMidpoint = midpoint(activePointers)
	const transform = {
		scale: distance(activePointers) / distance(state.gesture.pointers),
		translate: {
			x: currentMidpoint.x - origin.x,
			y: currentMidpoint.y - origin.y
		}
	}
	const dX = transform.translate.x - state.gesture.prevTranslate.x
	const dY = transform.translate.y - state.gesture.prevTranslate.y

	// Elastic zoom
	canvas.zoomTo(state.gesture.initialZoom * transform.scale, {
		x: origin.x + transform.translate.x,
		y: origin.y + transform.translate.y
	}, true)

	// Apply transformations to the canvas
	canvas.scroll.x += dX
	canvas.scroll.y += dY
	canvas.smoothScroll.x = canvas.scroll.x
	canvas.smoothScroll.y = canvas.scroll.y
	canvas.smoothZoom = canvas.zoom
	state.gesture.prevTranslate = transform.translate
}

function onGestureEnd() {
	state.pinching = false

	// Elastic zoom bounce back
	if (canvas.zoom > 2 || canvas.zoom < .2) {
		const canvasRect = canvas.ref.getBoundingClientRect()

		canvas.zoomTo(canvas.zoom, {
			x: canvasRect.x + canvasRect.width / 2,
			y: canvasRect.y + canvasRect.height / 2
		})
		canvas.animate(300)
	}

	unwatchGestureChange()
}

function onClick(event: MouseEvent) {
	if (!clickAllowed)
		return

	if (selection.cards.length) {
		selection.clear()

		if (pointer.type === 'touch' || !settings.doubleClickCreateCard)
			return
	}

	// Require doubleclick to create a card when using a mouse
	if (pointer.type === 'mouse' && event.detail !== 2 && settings.doubleClickCreateCard)
		return

	createCard({ new: true, pos: canvas.toCanvasPos(pointer) })
}

function onWheel(event: WheelEvent) {
	if (state.panning || state.pinching)
		return

	const isMouseWheel = !(Date.now() - lastTrackpadTime < 1000 || isTrackpad(event))
	let deltaX = event.deltaX
	let deltaY = event.deltaY

	// Normalize scroll speed
	if (isMouseWheel) {
		deltaX = Math.sign(deltaX) * 100
		deltaY = Math.sign(deltaY) * 100
	} else {
		deltaX *= settings.trackpadSensitivity
		deltaY *= settings.trackpadSensitivity
		lastTrackpadTime = Date.now()
	}

	if (event.ctrlKey || event.metaKey) {
		// Zoom the canvas
		const delta = isMouseWheel
			? Math.sign(event.deltaY) * .2
			: event.deltaY / 150

		canvas.zoomTo(canvas.zoom * (1 - delta), { x: event.clientX, y: event.clientY })

		if (isMouseWheel)
			canvas.animate()
		else {
			canvas.smoothScroll.x = canvas.scroll.x
			canvas.smoothScroll.y = canvas.scroll.y
			canvas.smoothZoom = canvas.zoom
		}
	} else {
		// Scroll horizontally when holding shift or vertically otherwise
		if (isMouseWheel && event.shiftKey && !deltaX)
			[deltaX, deltaY] = [deltaY, deltaX]

		canvas.scroll.x -= deltaX
		canvas.scroll.y -= deltaY

		if (isMouseWheel)
			canvas.animate()
		else {
			canvas.smoothScroll.x = canvas.scroll.x
			canvas.smoothScroll.y = canvas.scroll.y
		}

		if (pointer.down && !pointer.moved)
			pointer.moved = true
	}
}

async function onPaste(event: ClipboardEvent | DragEvent) {
	if (!state.pointerOver || usingInput())
		return

	state.loading = true

	const pasteToast = toast('Processing your content...', 'yellow')
	const isClipboardEvent = event instanceof ClipboardEvent
	const dataTransfer = isClipboardEvent ? event.clipboardData : event.dataTransfer
	const pos = canvas.toCanvasPos(isClipboardEvent ? pointer : { x: event.clientX, y: event.clientY })
	const pasted = await pasteOnCanvas(dataTransfer, pos)
	const pastedCards = (pasted.cards as Card[]).map(card => createCard(card))

	untoast(pasteToast)

	if (pastedCards.length) {
		if (pastedCards.length > 1)
			toast(`Pasted ${pastedCards.length} ${pasted.type}s`)
		else
			toast(`Pasted ${pasted.type}`)

		await nextTick()

		selection.cards = pastedCards
	} else if (pasted.error)
		toast(pasted.error, 'red')

	state.loading = false
}
</script>

<template>
	<div
		ref="canvas-ref"
		class="canvas"
		:style="{ cursor }"
		@pointerdown.left="onPointerDown"
		@pointerdown.middle="onPointerDown"
		@pointerenter="state.pointerOver = true"
		@pointerleave="state.pointerOver = false"
		@pointercancel ="state.pointerOver = false"
		@click.left.self="onClick"
		@click.middle.self.prevent
		@wheel.prevent="onWheel"
		@contextmenu.self.prevent
		@dragenter="state.pointerOver = true"
		@dragleave="state.pointerOver = false"
		@dragover.self.stop.prevent
		@drop.self.prevent="onPaste"
	>
		<CanvasBackground :canvas />
		<div
			class="cards"
			:style="{
				translate: `${canvas.smoothScroll.x}px ${canvas.smoothScroll.y}px`,
				scale: canvas.smoothZoom,
				willChange: animating ? 'transform' : 'auto'
			}"
		>
			<Card
				v-for="card of cards"
				:key="card.id"
				ref="card-refs"
				:card
			/>
		</div>
		<div
			class="box-selection"
			:style="boxSelectionStyle"
		/>
		<DrawSelection />
	</div>
</template>

<style>
.canvas {
	position: relative;
	flex-grow: 1;
	overflow: clip;
	user-select: none;
	touch-action: none;

	.cards {
		position: absolute;
	}

	.box-selection {
		position: absolute;
		background-color: var(--color-accent-25);
		border: 1px solid var(--color-accent);
		pointer-events: none;
	}
}
</style>
