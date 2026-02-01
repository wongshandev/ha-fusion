<script lang="ts">
	import { dashboard, motion, record, lang, ripple } from '$lib/Stores';
	import Ripple from 'svelte-ripple';
	import Icon from '@iconify/svelte';
	import { generateId } from '$lib/Utils';
	import { createEventDispatcher } from 'svelte';

	export let view: any;

	const dispatch = createEventDispatcher();

	$: noViewsOrSectionsOrStacks =
		!view ||
		!view.sections ||
		view.sections.length === 0 ||
		checkForHorizontalStackOnly(view.sections);

	function checkForHorizontalStackOnly(sections: any[]): boolean {
		return sections.every((section) => {
			if (section.type === 'horizontal-stack') {
				return section.sections ? checkForHorizontalStackOnly(section.sections) : true;
			}
			return false;
		});
	}

	/**
	 * Creates a new Tesla card in
	 * first section of current view
	 */
	function handleClick() {
		if (noViewsOrSectionsOrStacks) return;

		const section = findSection(view.sections);

		if (!section?.items) return;

		section.items.unshift({
			type: 'tesla',
			id: generateId($dashboard),
			prefix: 'san',
			model: 'y',
			color: 'white'
		});

		$dashboard = $dashboard;
		$record();

		dispatch('clicked');
	}

	function findSection(sections: any[]): any | undefined {
		for (const section of sections) {
			if (section.type !== 'horizontal-stack') return section;
			const found = section.sections && findSection(section.sections);
			if (found) return found;
		}
	}
</script>

<button
	class="button dropdown"
	on:click={handleClick}
	use:Ripple={{
		...$ripple,
		opacity: noViewsOrSectionsOrStacks ? '0' : $ripple.opacity
	}}
	style:cursor={noViewsOrSectionsOrStacks ? 'unset' : 'pointer'}
	style:opacity={noViewsOrSectionsOrStacks ? '0.5' : '1'}
	style:transition="opacity {$motion}ms ease"
>
	<figure>
		<Icon icon="mdi:car-electric" height="none" />
	</figure>

	Tesla
</button>
