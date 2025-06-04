<script>
	import { createEventDispatcher } from "svelte";

	export let size;

	let element;

	$: svgPoint = element ? element.createSVGPoint() : null;

	function svgCoords(event) {
		const rect = element.getBoundingClientRect();
		const viewBox = element.viewBox.baseVal;
		const clientX = event.touches
			? event.touches[0].clientX
			: event.clientX;
		const clientY = event.touches
			? event.touches[0].clientY
			: event.clientY;

		const x =
			((clientX - rect.left) / rect.width) * viewBox.width + viewBox.x;
		const y =
			((clientY - rect.top) / rect.height) * viewBox.height + viewBox.y;

		return { x, y };
	}

	const dispatch = createEventDispatcher();
	let active = false;

	function stopPoint() {
		active = false;
	}

	function startPoint(evt) {
		if (evt.pointerId !== undefined) {
			evt.currentTarget.setPointerCapture(evt.pointerId);
		}
		active = true;
		point(evt, true);
	}

	function point(evt, initial = false) {
		if (!element || !svgPoint) {
			return;
		}

		if (active) {
			let { x, y } = svgCoords(evt);

			dispatch("point", {
				x,
				y,
				initial,
			});
		}
	}
</script>

<svg
	on:pointerup={() => stopPoint()}
	on:pointerdown={startPoint}
	on:pointermove={point}
	on:touchend={() => stopPoint()}
	on:touchstart={startPoint}
	bind:this={element}
	role="presentation"
	class="image-array"
	viewBox="0 0 {size.x} {size.y}"
	width={size.x}
	height={size.y}
	preserveAspectRatio="xMidYMid meet"
>
	<slot></slot>
</svg>

<style>
	.image-array {
		max-width: 18em;
		max-height: 24em;
		width: 100%;
		min-width: 20em;
		height: auto;
		border: 1px solid white;
		display: block;
		background: gray;
		touch-action: none;
	}
</style>
