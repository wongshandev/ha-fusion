<script lang="ts">
	import { dashboard, lang, editMode } from '$lib/Stores';
	import ConfigButtons from '$lib/Modal/ConfigButtons.svelte';
	import { closeModal } from 'svelte-modals';
	import Select from '$lib/Components/Select.svelte';
	import InputClear from '$lib/Components/InputClear.svelte';

	export let isOpen: boolean;
	export let sel: any;

	let prefix = sel?.prefix || 'san';
	let model = sel?.model || 'y';
	let color = sel?.color || 'white';

	const modelOptions = [
		{ id: '3', label: 'Model 3' },
		{ id: 'y', label: 'Model Y' },
		{ id: 's', label: 'Model S' },
		{ id: 'x', label: 'Model X' }
	];

	const colorOptions = [
		{ id: 'white', label: 'White' },
		{ id: 'black', label: 'Black' },
		{ id: 'red', label: 'Red' },
		{ id: 'blue', label: 'Blue' },
		{ id: 'gray', label: 'Gray' },
		{ id: 'silver', label: 'Silver' }
	];

	function handleChange() {
		// Update dashboard configuration
		$dashboard = {
			...$dashboard,
			views: $dashboard?.views?.map((view) => ({
				...view,
				sections: view?.sections?.map((section) => ({
					...section,
					items: section?.items?.map((item: any) => {
						if (item?.id === sel?.id) {
							return {
								...item,
								prefix,
								model,
								color
							};
						}
						return item;
					})
				}))
			}))
		};
	}

	function handleClose() {
		closeModal();
	}
</script>

{#if isOpen}
	<div class="modal-content">
		<h2>Tesla Card Configuration</h2>

		<div class="field">
			<label for="prefix">Entity Prefix</label>
			<InputClear
				condition={prefix}
				on:clear={() => {
					prefix = '';
					handleChange();
				}}
			>
				<input
					id="prefix"
					type="text"
					bind:value={prefix}
					on:change={handleChange}
					placeholder="e.g., san"
				/>
			</InputClear>
			<small>The prefix used in your Tesla entity IDs (e.g., sensor.san_battery)</small>
		</div>

		<div class="field">
			<label for="model">Car Model</label>
			<Select
				options={modelOptions}
				selected={model}
				on:change={(e) => {
					model = e.detail;
					handleChange();
				}}
			/>
		</div>

		<div class="field">
			<label for="color">Car Color</label>
			<Select
				options={colorOptions}
				selected={color}
				on:change={(e) => {
					color = e.detail;
					handleChange();
				}}
			/>
		</div>

		<ConfigButtons {sel} on:close={handleClose} />
	</div>
{/if}

<style>
	.modal-content {
		padding: 1.5rem;
		max-width: 400px;
	}

	h2 {
		margin: 0 0 1.5rem 0;
		font-size: 1.2rem;
		font-weight: 600;
	}

	.field {
		margin-bottom: 1.25rem;
	}

	.field label {
		display: block;
		margin-bottom: 0.5rem;
		font-size: 0.9rem;
		font-weight: 500;
		color: var(--theme-button-name-color-off);
	}

	.field input {
		width: 100%;
		padding: 0.6rem 0.75rem;
		border: 1px solid rgba(255, 255, 255, 0.15);
		border-radius: 0.4rem;
		background: rgba(0, 0, 0, 0.2);
		color: inherit;
		font-size: 0.95rem;
	}

	.field input:focus {
		outline: none;
		border-color: rgba(255, 255, 255, 0.3);
	}

	.field small {
		display: block;
		margin-top: 0.35rem;
		font-size: 0.75rem;
		opacity: 0.6;
	}
</style>
