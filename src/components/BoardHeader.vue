<script setup lang="ts">
import { useSettings } from '../composables/settings'
import { inject } from 'vue'
import { useGateway } from '../composables/gateway'
import { updateBoard } from '../composables/board'
import IconSnap from './Icons/IconSnap.vue'
import IconGrid from './Icons/IconGrid.vue'
import IconSettings from './Icons/IconSettings.vue'

const settings = useSettings()
const { board } = inject('board') as BoardContext
const { gateway } = useGateway()
</script>

<template>
	<header class="board-header">
		<div class="combobox">
			<div class="option">
				<IconSnap />
				<input
					type="radio"
					name="snap"
					value="cards"
					v-model="settings.snap"
				>
			</div>
			<div class="option">
				<IconGrid />
				<input
					type="radio"
					name="snap"
					value="grid"
					v-model="settings.snap"
				>
			</div>
		</div>
		<input v-model="board.name" @blur="() => updateBoard(board)">
		<div v-if="!gateway.connected.value">Gateway not connected</div>
		<RouterLink to="/">Board List</RouterLink>
		<div class="flex-spacer"></div>
		<RouterLink to="/settings" class="settings-button">
			<IconSettings />
		</RouterLink>
	</header>
</template>

<style>
.board-header {
	display: flex;
	padding: .25rem 1rem;
	gap: 1rem;
	align-items: center;
	user-select: none;
}

.settings-button {
	width: 2rem;
	height: 2rem;
	padding: .25rem;
	color: inherit;
	background-color: transparent;
	border-radius: 100%;

	.icon {
		width: 1.5rem;
		height: 1.5rem;
		opacity: .75;
		transition: rotate 200ms;
	}

	&:hover,
	&:focus-visible {
		background-color: #80808040;

		.icon {
			opacity: 1;
			rotate: 60deg;
		}
	}
}
</style>
