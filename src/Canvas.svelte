<script>
	export let image = {
		size: { x: 16, y: 16 },
		values: Array(16)
			.fill(0)
			.map((v) => Array(16).fill(v)),
	};

	let numberFormatter = new Intl.NumberFormat("en-IN", {
		minimumFractionDigits: 1,
		maximumFractionDigits: 1,
		signDisplay: "exceptZero",
		trailingZeroDisplay: "auto",
	});

	let config = false;

	export let maxSize = 50;
	export let minSize = 3;
	export let skipper = () => false;

	export let examples = {};

	function toggleConfig() {
		selectedRange = range;
		widthInput.value = image.size.x;
		heightInput.value = image.size.y;
		config = !config;
	}

	function loadExample(name) {
		const ex = examples[name];

		if (ex) {
			image = {
				size: { x: ex.size.x, y: ex.size.y },
				values: JSON.parse(JSON.stringify(ex.values)),
			};
			range = ex.range || 0;
		}
	}

	export let range = 0;
	let selectedRange = range;

	$: min = range;
	$: max = 1;

	$: if (
		image.values == null ||
		image.values.length != image.size.y ||
		image.values[0].length != image.size.x
	) {
		image.values = Array(image.size.y)
			.fill(null)
			.map(() => Array(image.size.x).fill(0));
	}

	$: realMin = min != null ? min : Math.min.apply(Math, image.values.flat());
	$: realMax = max != null ? max : Math.max.apply(Math, image.values.flat());

	let current = { x: null, y: null };
	let element;
	let svgPoint;
	let active = false;
	let outside = true;

	let widthInput, heightInput;

	export let brush = {
		value: 1,
		size: 3,
		hardness: 0,
		prev: { x: null, y: null },
	};

	$: svgPoint = element ? element.createSVGPoint() : null;

	function svgCoords(event) {
		var ctm = element.getScreenCTM();
		// Note: rest of method could work with another element,
		// if you don't want to listen to drags on the entire svg.
		// But createSVGPoint only exists on <svg> elements.
		svgPoint.x = event.clientX;
		svgPoint.y = event.clientY;
		let { x, y } = svgPoint.matrixTransform(ctm.inverse());

		return { x, y };
	}

	function clear() {
		config = false;
		image.size.x = widthInput.valueAsNumber << 0;
		image.size.y = heightInput.valueAsNumber << 0;
		range = selectedRange;
		image.values = Array(image.size.y)
			.fill(null)
			.map(() => Array(image.size.x).fill(0));
	}

	function clearColor() {
		config = false;
		image.values = Array(image.size.y)
			.fill(null)
			.map(() => Array(image.size.x).fill(0));
	}

	function startDrawing(evt) {
		if (evt.pointerId !== undefined) {
			evt.currentTarget.setPointerCapture(evt.pointerId);
		}
		active = true;
		moveDrawing(evt);
	}

	function stopDrawing() {
		active = false;
		brush.prev.x = null;
		brush.prev.y = null;
	}

	function moveDrawing(evt) {
		if (!element || !svgPoint) {
			return;
		}

		let { x, y } = svgCoords(evt);
		current.x = x;
		current.y = y;

		if (!active) {
			return;
		}
		const intX = Math.floor(x);
		const intY = Math.floor(y);
		const fracX = x - intX;
		const fracY = y - intY;
		if (brush.prev.x == intX && brush.prev.y == intY) {
			return;
		}

		if (brush.prev.x == null) {
			placeBrush(intX, intY, fracX, fracY);
		} else {
			let signX = 1;
			let signY = 1;

			const deltaX = Math.abs(intX - brush.prev.x);
			const deltaY = Math.abs(intY - brush.prev.y);
			const m = Math.abs(deltaY / deltaX);
			if (intX < brush.prev.x) signX = -1;
			if (intY < brush.prev.y) signY = -1;
			if (m <= 1)
				for (let i = 0; i <= deltaX; i++)
					placeBrush(
						Math.ceil(brush.prev.x + i * signX),
						Math.ceil(brush.prev.y + i * m * signY),
						fracX,
						fracY,
					);
			else
				for (let i = 0; i <= deltaY; i++)
					placeBrush(
						Math.ceil(brush.prev.x + (i / m) * signX),
						Math.ceil(brush.prev.y + i * signY),
						fracX,
						fracY,
					);
		}

		brush.prev.x = intX;
		brush.prev.y = intY;
	}

	function placeBrush(x, y, fracX, fracY) {
		const value = 1 * realMin + (realMax - realMin) * brush.value;
		const radius = brush.size / 2 + 0.5;
		let radius2 = radius * radius;
		for (let rx = -radius - 1; rx <= radius; rx++) {
			const xx = Math.floor(x + rx);
			for (let ry = -radius - 1; ry <= radius; ry++) {
				const yy = Math.floor(y + ry);
				let distance2 = rx * rx + ry * ry;
				if (
					yy < image.size.y &&
					yy >= 0 &&
					xx < image.size.x &&
					xx >= 0 &&
					radius2 > distance2
				) {
					const alpha =
						1 -
						smoothstep(
							distance2,
							radius2 * Math.pow(brush.hardness, 64),
							radius2,
						);
					image.values[yy][xx] =
						image.values[yy][xx] * (1 - alpha) + alpha * value;
				}
			}
		}
	}

	function smoothstep(x, edge0 = 0.0, edge1 = 1.0) {
		// Scale, and clamp x to 0..1 range
		x = clamp((x - edge0) / (edge1 - edge0));

		return x * x * (3.0 - 2.0 * x);
	}

	function clamp(x, lowerlimit = 0.0, upperlimit = 1.0) {
		if (x < lowerlimit) return lowerlimit;
		if (x > upperlimit) return upperlimit;
		return x;
	}

	function enter() {
		outside = false;
	}

	function exit() {
		outside = true;
	}
</script>

<section class="container">
	<div class:hidden={!config} class="options-overlay">
		<fieldset class="option-container">
			<legend class="option-container-label">Dimensions</legend>
			<dl class="options">
				<dt>Width</dt>
				<dd>
					<input
						type="number"
						size="5"
						min={minSize}
						max={maxSize}
						step="1"
						value={image.size.x}
						bind:this={widthInput}
					/>
				</dd>
				<dt>Height</dt>
				<dd>
					<input
						type="number"
						size="5"
						min={minSize}
						max={maxSize}
						step="1"
						value={image.size.y}
						bind:this={heightInput}
					/>
				</dd>
				<dt>Value Range</dt>
				<dd>
					<ul>
						<li>
							<label
								><input type="radio" bind:group={selectedRange} value={0} /> Positive
								(0.0 to 1.0)</label
							>
						</li>
						<li>
							<label
								><input type="radio" bind:group={selectedRange} value={-1} /> Symmetric
								(-1.0 to 1.0)</label
							>
						</li>
					</ul>
				</dd>
				<dt></dt>
				<dd>
					<button on:click={clear}>Resize+Clear</button><br />
					<button class="small-button" on:click={toggleConfig}>Cancel</button>
				</dd>
			</dl>
		</fieldset>
	</div>

	<div class="vstack">
		<div class="figure-with-legend">
			<svg
				bind:this={element}
				on:pointerenter={enter}
				on:pointerleave={exit}
				class:active
				role="presentation"
				on:pointerup={stopDrawing}
				on:pointerdown={startDrawing}
				on:pointermove={moveDrawing}
				on:touchstart={startDrawing}
				on:touchend={stopDrawing}
				class="image-array no-cursor"
				viewBox="0 0 {image.size.x} {image.size.y}"
				width={image.size.x}
				height={image.size.y}
				preserveAspectRatio="xMidYMid meet"
			>
				<g pointer-events="none">
					{#each image.values as row, y}
						<g>
							{#each row as value, x}
								<g>
									<rect
										pointer-events="none"
										class="intensity"
										style:--intensity={(value - realMin) / (realMax - realMin)}
										fill="magenta"
										image-rendering="crisp-edges"
										stroke="#abb3"
										{x}
										{y}
										width="1"
										height="1"
										vector-effect="non-scaling-stroke"
										stroke-width="1px"
									></rect>
								</g>
								{#if skipper && skipper(value, min, max)}
									<g opacity="0.5">
										<line
											stroke-width="0.05"
											x1={x + 0.1}
											y1={y + 0.1}
											x2={x + 1 - 0.1}
											y2={y + 1 - 0.1}
											stroke="#ff5555"
										/>
										<line
											stroke-width="0.05"
											x1={x + 1 - 0.1}
											y1={y + 0.1}
											x2={x + 0.1}
											y2={y + 1 - 0.1}
											stroke="#ff5555"
										/>
									</g>
								{/if}
							{/each}
						</g>
					{/each}
					<circle
						class:hidden={outside}
						vector-effect="non-scaling-stroke"
						r={brush.size / 2}
						cx={current.x}
						cy={current.y}
						fill="#fffa"
						stroke="#0005"
						stroke-width="1px"
					/>
					<circle
						class:hidden={outside}
						vector-effect="non-scaling-stroke"
						r={(brush.size / 2) * brush.hardness}
						cx={current.x}
						cy={current.y}
						fill="#fffa"
					/>
				</g>
			</svg>
			<div class="legend">
				<span class="legend-label">{numberFormatter.format(max)}</span>
				<span class="legend-label">{numberFormatter.format(min)}</span>
			</div>
		</div>
		<div class="row">
			<button class="small-button" on:click={clearColor}>Clear</button>
			<button class="small-button" on:click={toggleConfig}>Resize</button>
		</div>
		<strong>Examples</strong>
		<div
			class="row"
			style:max-width="15em"
			style:align-self="center"
			style:justify-content="center"
			style:flex-wrap="wrap"
		>
			{#each Object.keys(examples) as ex}
				<button class="small-button" on:click={() => loadExample(ex)}
					>{ex}</button
				>
			{/each}
		</div>
	</div>
</section>

<style>
	.image-array {
		max-width: 18em;
		max-height: 24em;
		min-width: 19em;
		width: 100%;
		height: auto;
		border: 1px solid white;
		display: block;
		touch-action: none;
	}

	.no-cursor {
		cursor: none;
	}

	.intensity {
		fill: hsl(0deg 0% calc(var(--intensity, 0.5) * 100%));
		accent-color: hsl(0deg 0% calc(var(--intensity, 0.5) * 100%));
	}

	.options {
		display: grid;
		grid-template-columns: auto auto;
		gap: 0.5em 1em;
	}

	.options-overlay {
		z-index: 100;
		padding: 1em;
		background: #000d;
	}

	.options-overlay.hidden {
		visibility: hidden;
	}

	circle.hidden {
		display: none;
	}

	.option-container {
		display: flex;
		gap: 1em;
		border: none;
		background: #333;
	}

	.option-container-label {
		display: inline-block;
		padding: 2px 0.5em;
		background: inherit;
	}

	dd,
	dt,
	dl {
		margin: 0;
	}

	.container {
		display: grid;
		grid-template-columns: max-content;
		gap: 0.4em;
		align-items: stretch;
		align-content: stretch;
		background: #111;
		color: #fff;
	}

	.container > * {
		grid-column: 1;
		grid-row: 1;
	}

	dt {
		align-self: center;
	}

	dd {
		align-self: center;
	}

	input,
	button {
		margin: 0;
	}

	ul {
		display: flex;
		list-style: none;
		flex-direction: column;
		margin: 0;
		padding: 0;
		gap: 0.2em;
	}

	li {
		margin: 0;
		padding: 0;
	}

	.small-button {
		color: inherit;
		font: inherit;
		background: none;
		border: none;
		text-decoration: underline;
		cursor: pointer;
	}

	.vstack {
		display: flex;
		flex-direction: column;
		align-items: center;
	}

	.row {
		display: flex;
		align-self: stretch;
		justify-content: space-around;
	}

	.legend {
		width: 100%;
		height: 100%;
		border: 1px solid currentColor;
		box-sizing: border-box;
		background-image: linear-gradient(0deg, black 0%, white 100%);
		display: flex;
		flex-direction: column;
		justify-content: space-between;
		align-items: center;
		font-weight: bold;
		font-size: 0.8em;
	}

	.legend > :first-child {
		color: black;
	}
	.legend > :last-child {
		color: white;
	}

	.figure-with-legend {
		display: grid;
		grid-template-columns: auto 2em;
		gap: 0.5em;
	}
</style>
