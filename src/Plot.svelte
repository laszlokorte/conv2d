<script>
	import { createEventDispatcher } from 'svelte';

	export let size

	let element;

	$: svgPoint = element ? element.createSVGPoint() : null;

	function svgCoords(event) {
		var ctm = element.getScreenCTM();
	    // Note: rest of method could work with another element,
	    // if you don't want to listen to drags on the entire svg.
	    // But createSVGPoint only exists on <svg> elements.
		svgPoint.x = event.clientX;
		svgPoint.y = event.clientY;
		let {x,y} = svgPoint.matrixTransform(ctm.inverse());

		return {x,y}
	}

	const dispatch = createEventDispatcher();
	let active = false

	function stopPoint() {
		active = false
	}

	function startPoint(evt) {
		active = true
		point(evt,true)
	}
	

	function point(evt, initial=false) {
		
		if(!element || !svgPoint) {
			return
		}

		if(active) {
			let {x,y} = svgCoords(evt);

			dispatch('point', {
				x,y,initial
			});
		}

	}
</script>

<style>
	.image-array {
		max-width: 18em;
		max-height: 24em;
		width: 100%;
		min-width: 20em;
		height: auto;
		border: 1px solid white;
		display: block;
	}
</style>

<svelte:document on:mouseup={() => stopPoint()} on:mousemove={(evt) => point(evt)} />


<svg on:mousedown={startPoint} bind:this={element} role="presentation" class="image-array" viewBox="0 0 {size.x} {size.y}" width="{size.x}" height="{size.y}" preserveAspectRatio="meet xMidYMid">
	<slot></slot>
</svg>
