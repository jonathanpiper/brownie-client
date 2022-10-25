<script>
	import noUiSlider from 'noUiSlider';
    import wNumb from 'wNumb';
	import 'nouislider/dist/nouislider.css';
	import { onMount, createEventDispatcher } from 'svelte';
	export var register;

	const dispatch = createEventDispatcher();

	var slider;
	onMount(() => {
		noUiSlider.create(slider, {
			range: {
				min: 0,
				max: 255,
			},
			start: 0,
			orientation: 'horizontal',
			behaviour: 'tap-drag',
			format: wNumb({
				decimals: 0,
			}),
		});

		slider.noUiSlider.on('change', (values, handle, unencoded, tap, positions) => dispatch('change', { values, handle, unencoded, tap, positions }));
	});

	$: if (slider) {
		slider.noUiSlider.set(register);
	}
</script>

<div id="slider" bind:this={slider} />