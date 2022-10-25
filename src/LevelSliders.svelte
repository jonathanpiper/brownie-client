<script>
	import noUiSlider from 'noUiSlider';
    import wNumb from 'wNumb';
	import 'nouislider/dist/nouislider.css';
	import { onMount, createEventDispatcher } from 'svelte';
	export var levelMin, levelMax;

	const dispatch = createEventDispatcher();

	var levelsSlider;
	onMount(() => {
		noUiSlider.create(levelsSlider, {
			range: {
				min: 0,
				max: 100,
			},
			connect: true,
			start: [0, 100],
			direction: 'rtl',
			orientation: 'vertical',
			behaviour: 'tap-drag',
			format: wNumb({
				decimals: 0,
			}),
		});

		levelsSlider.noUiSlider.on('change', (values, handle, unencoded, tap, positions) => dispatch('change', { values, handle, unencoded, tap, positions }));
	});

	$: if (levelsSlider) {
		levelsSlider.noUiSlider.set([levelMin, levelMax]);
	}
</script>

<div id="levels-slider" bind:this={levelsSlider} />