<script setup lang="ts">
import { computed, type Ref, inject, reactive, toRef, useTemplateRef, watch, type WatchHandle } from 'vue'
import { usePointer } from '../composables/pointer'
import { useSettings } from '../composables/settings'
import { useGateway } from '../composables/gateway'
import { onceChanged, rectContains, rectsOverlap, suppressClick } from '../utils'
import type Card from './Card.vue'
import CardContentBox from './CardContentBox.vue'
import CardContentText from './CardContentText.vue'
import CardContentImage from './CardContentImage.vue'
import CardContentLink from './CardContentLink.vue'
import CardContentAudio from './CardContentAudio.vue'
import CardContentVideo from './CardContentVideo.vue'

type CardRef = InstanceType<typeof Card>
type CardContentRef = InstanceType<ReturnType<typeof getContentComponent>>
type CardContentBoxRef = InstanceType<typeof CardContentBox>
type CardContentImageRef = InstanceType<typeof CardContentImage>

const { card } = defineProps<{ card: Card }>()
const cardRef = useTemplateRef('card-ref')
const contentRef = useTemplateRef<CardContentRef>('content-ref')
const selection = inject('selection') as CanvasSelection
const canvas = inject('canvas') as CanvasContext
const { board, updateCard, updateMany } = inject('board') as BoardContext
const state = reactive({
	selected: false,
	dragging: false,
	resizing: false as false | string,
	downState: {
		offsetX: 0,
		offsetY: 0,
		zoom: 0,
		width: 0,
		height: 0
	}
})
const { pointer, pointers } = usePointer()
const cardRefs = inject('card-refs') as Ref<CardRef[]>
const { gateway } = useGateway()
const settings = useSettings()
const cursor = computed(() => {
	if (contentRef.value?.active) return 'auto'
	if (selection.boxVisible || selection.draw) return 'inherit'
	if (pointer.down) return 'grabbing'

	return 'grab'
})
const zIndex = computed(() => {
	const isActive = contentRef.value?.active
		|| state.dragging
		|| card.type !== 'box' && state.selected
		|| card.type !== 'box' && state.resizing

	return isActive ? 1 : 0
})
let cardRefMap = new Map<Card, CardRef>()
let relatedCards = new Set<Card>()
let unwatchPointer: WatchHandle
let unwatchPointerMove: WatchHandle
let unwatchPointerUp: WatchHandle

watch(() => selection.cards, () => {
	state.selected = selection.cards.includes(card)
})

watch(() => selection.box, () => {
	if (selection.box) {
		const cardRect = canvas.toCanvasRect(cardRef.value!.getBoundingClientRect())
		let inBox = rectsOverlap(selection.box, cardRect)

		if (card.type === 'box') {
			const boxRect = canvas.toCanvasRect((contentRef.value as CardContentBoxRef).boxRef!.getBoundingClientRect())

			inBox ||= rectsOverlap(selection.box, boxRect)
		}

		// Check if the card is in the selection box
		state.selected = !!selection.box && inBox
	}
})

watch(() => selection.draw, () => {
	if (selection.draw) {
		// Select the card when the pointer is moved over it during draw selection
		unwatchPointer = watch(pointer, () => {
			const cardRect = cardRef.value!.getBoundingClientRect()
			const pointerRect = new DOMRect(pointer.x - 10, pointer.y - 10, 20, 20)

			if (rectsOverlap(cardRect, pointerRect))
				state.selected = true
		})
	} else
		unwatchPointer()
})

watch(() => state.selected, () => {
	if (state.selected) {
		if (!selection.cards.includes(card)) {
			selection.cards.push(card)

			if (navigator.vibrate)
				navigator.vibrate(50)
		}
	} else
		selection.cards.splice(selection.cards.indexOf(card), 1)
})

function onPointerDown(event: PointerEvent) {
	// Wait until the event has bubbled to the listener on the document that updates the pointer state
	onceChanged(pointer, () => {
		if (contentRef.value!.active || pointers.length !== 1)
			return

		cardRefMap = new Map(cardRefs.value.map(cardRef => [cardRef.card, cardRef]))

		const targetClassName = (event.target as HTMLElement).className

		if (targetClassName.startsWith('resize')) {
			state.resizing = targetClassName

			selection.clear()
			relatedCards.clear()
		}

		const cardRect = cardRef.value!.getBoundingClientRect()

		state.downState.offsetX = event.clientX - cardRect.x
		state.downState.offsetY = event.clientY - cardRect.y
		state.downState.zoom = canvas.smoothZoom

		if (card.type === 'box' || card.type === 'image') {
			state.downState.width = card.content.width
			state.downState.height = card.content.height
		} else if (card.type === 'text')
			state.downState.width = cardRect.width / canvas.smoothZoom

		unwatchPointerMove = watch([pointer, canvas], onPointerMove)
		unwatchPointerUp = watch([
			() => pointer.down,
			() => pointers.length
		], onPointerUp, { flush: 'sync' })
	})
}

function onPointerMove() {
	if (!pointer.moved)
		return

	if (!state.dragging && !state.resizing) {
		state.dragging = true

		if (!state.selected)
			selection.clear()

		// Collect all cards that should be dragged along
		relatedCards = new Set(selection.cards.filter(c => c !== card))

		function addRelatedBoxCards(boxCard: Card) {
			if (boxCard.type !== 'box')
				return

			const boxCardRef = cardRefMap.get(boxCard)!
			const boxRect = (boxCardRef.contentRef! as CardContentBoxRef).boxRef!.getBoundingClientRect()

			cardRefMap.forEach(cardRef => {
				if (cardRef.card !== card && !relatedCards.has(cardRef.card) && rectContains(boxRect, cardRef.ref!.getBoundingClientRect())) {
					cardRef.dragging = true

					relatedCards.add(cardRef.card)
					addRelatedBoxCards(cardRef.card)
				}
			})
		}

		[card, ...relatedCards].forEach(addRelatedBoxCards)
	}

	const prevPos = card.pos
	const newPos = canvas.toCanvasPos({
		x: pointer.x - state.downState.offsetX * canvas.smoothZoom / state.downState.zoom,
		y: pointer.y - state.downState.offsetY * canvas.smoothZoom / state.downState.zoom
	})

	if (state.dragging) {
		card.pos = {
			x: snap(newPos.x, 'x', 'drag'),
			y: snap(newPos.y, 'y', 'drag')
		}

		if (!pointer.altKey) {
			relatedCards.forEach(c => {
				c.pos.x += card.pos.x - prevPos.x
				c.pos.y += card.pos.y - prevPos.y
			})
		}

		gateway.send({
			type: 'updateCards',
			board: board.id,
			cards: [card, ...relatedCards].map(card => ({ id: card.id, pos: card.pos, modified: Date.now() }))
		})
	}

	if (state.resizing) {
		// Boxes can be resized freely
		if (card.type === 'box') {
			if (state.resizing === 'resize-h' || state.resizing === 'resize-d')
				card.content.width = Math.max(snap(state.downState.width + newPos.x, 'x', 'resize') - card.pos.x, 40)

			if (state.resizing === 'resize-v' || state.resizing === 'resize-d')
				card.content.height = Math.max(snap(state.downState.height + newPos.y, 'y', 'resize') - card.pos.y, 40)
		}

		// Images need to retain aspect ratio while resizing
		if (card.type === 'image') {
			const { imgWidth, imgHeight } = contentRef.value as CardContentImageRef
			const aspectRatio = imgWidth / imgHeight
			let newWidth = Math.max(snap(state.downState.width + newPos.x, 'x', 'resize') - card.pos.x, 40)
			let newHeight = Math.max(snap(state.downState.height + newPos.y, 'y', 'resize') - card.pos.y, 40)

			if (newWidth / newHeight < aspectRatio)
				newWidth = newHeight * aspectRatio
			else
				newHeight = newWidth / aspectRatio

			card.content.width = Math.min(newWidth, imgWidth)
			card.content.height = Math.min(newHeight, imgHeight)
		}

		// Text cards can be resized horizontally
		if (card.type === 'text')
			card.content.width = Math.max(snap(state.downState.width + newPos.x, 'x', 'resize') - card.pos.x, 0)
	}

	canvas.edgeScroll()
}

function onPointerUp() {
	if (pointer.down && pointers.length === 1)
		return

	if (pointer.moved) {
		if (pointer.type === 'mouse')
			suppressClick()

		canvas.stopEdgeScroll()

		if (relatedCards.size)
			updateMany([card, ...relatedCards])
		else
			updateCard(card)
	}

	relatedCards.forEach(relatedCard => cardRefMap.get(relatedCard)!.dragging = false)

	state.dragging = false
	state.resizing = false

	unwatchPointerMove()
	unwatchPointerUp()
}

function snap(value: number, direction: 'x' | 'y', mode: 'drag' | 'resize') {
	if (!(pointer.ctrlKey || pointer.metaKey))
		return value

	const gridSize = canvas.gridSize / canvas.smoothZoom

	// Vertical offset between box label and box
	const ownRect = canvas.toCanvasRect((card.type === 'box' ? (contentRef.value as CardContentBoxRef).boxRef : cardRef.value)!.getBoundingClientRect())
	const boxLabelRect = canvas.toCanvasRect(cardRef.value!.getBoundingClientRect())
	let offset = 0

	if (card.type === 'box' && direction === 'y')
		offset = (ownRect.top - boxLabelRect.top)

	// Snap to other cards
	if (settings.snap === 'cards') {
		const snap = gridSize / 2
		const marginStart = mode === 'resize' ? 4 : 0
		const marginEnd = mode === 'drag' ? 4 : 0
		const size = direction === 'x' ? ownRect.width : ownRect.height

		// Get visible card rects
		const canvasRect = canvas.toCanvasRect(canvas.ref.getBoundingClientRect())
		const cardRects = Array.from(cardRefMap.values())
			.filter(cardRef => cardRef.card !== card && !relatedCards.has(cardRef.card))
			.flatMap(cardRef => {
				const cardRect = canvas.toCanvasRect(cardRef.ref!.getBoundingClientRect())

				if (cardRef.card.type === 'box') {
					const boxRef = (cardRef.contentRef as CardContentBoxRef).boxRef!
					const boxRect = canvas.toCanvasRect(boxRef.getBoundingClientRect())

					return [cardRect, boxRect]
				}

				return [cardRect]
			})
			.filter(cardRect => rectsOverlap(canvasRect, cardRect))

		for (const cardRect of cardRects) {
			const [primaryEdge, secondaryEdge] = direction === 'x'
				? [cardRect.left - marginStart, cardRect.right + marginEnd]
				: [cardRect.top - marginStart, cardRect.bottom + marginEnd]

			if (Math.abs(value + offset - primaryEdge) < snap) return primaryEdge - offset
			if (Math.abs(value + offset - secondaryEdge) < snap) return secondaryEdge - offset

			if (mode === 'drag') {
				if (Math.abs(value + size + offset - primaryEdge) < snap) return primaryEdge - size - marginEnd - offset
				if (Math.abs(value + size + offset - secondaryEdge) < snap) return secondaryEdge - size - marginEnd - offset
			}

			// Box label
			if (card.type === 'box' && mode === 'drag') {
				if (direction === 'x') {
					const labelWidth = boxLabelRect.width

					if (Math.abs(value + labelWidth + offset - primaryEdge) < snap) return primaryEdge - labelWidth - marginEnd - offset
					if (Math.abs(value + labelWidth + offset - secondaryEdge) < snap) return secondaryEdge - labelWidth - marginEnd - offset
				}

				if (direction === 'y') {
					if (Math.abs(value - primaryEdge) < snap) return primaryEdge
					if (Math.abs(value - secondaryEdge) < snap) return secondaryEdge
				}
			}
		}
	}

	// Snap to grid
	if (settings.snap === 'grid')
		return Math.round((value + offset) / gridSize) * gridSize - offset

	return value
}

function getContentComponent() {
	switch (card.type) {
		case 'box': return CardContentBox
		case 'text': return CardContentText
		case 'image': return CardContentImage
		case 'link': return CardContentLink
		case 'audio': return CardContentAudio
		case 'video': return CardContentVideo
	}
}

defineExpose({ card, ref: cardRef, contentRef, dragging: toRef(state, 'dragging') })
</script>

<template>
	<div
		ref="card-ref"
		class="card"
		:class="{ selected: state.selected }"
		:style="{
			translate: `${card.pos.x}px ${card.pos.y}px`,
			willChange: state.dragging && pointer.moved ? 'transform' : 'auto',
			cursor,
			zIndex
		}"
		@pointerdown.left="onPointerDown"
		@click.left.exact="selection.clear()"
		@click.left.ctrl.exact="state.selected = !state.selected"
		@click.left.meta.exact="state.selected = !state.selected"
	>
		<component
			ref="content-ref"
			:is="getContentComponent()"
			:card
		/>
	</div>
</template>

<style>
.card {
	position: absolute;
}

.resize-d,
.resize-h,
.resize-v {
	position: absolute;
	pointer-events: all;
}

.resize-d {
	right: -.5rem;
	bottom: -.5rem;
	width: .875rem;
	height: .875rem;
	cursor: se-resize;
}

.resize-h {
	top: 0;
	right: -.25rem;
	width: .375rem;
	height: 100%;
	cursor: ew-resize;
}

.resize-v {
	left: 0;
	bottom: -.25rem;
	width: 100%;
	height: .375rem;
	cursor: ns-resize;
}

@media (pointer: coarse) {
	.resize-d {
		scale: 2;
	}

	.resize-h {
		scale: 2 1;
	}

	.resize-v {
		scale: 1 2;
	}
}
</style>
