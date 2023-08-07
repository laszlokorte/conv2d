<script>
	import Canvas from './Canvas.svelte'


	let brush = {
		active: false,
		value: 1,
		size: 1,
		hardness: 0,
		element: null,
		prev:{x:null,y:null},
		current:{x:null,y:null},
	}

	let inputImage = {
		size: {x:20,y:20},
		values: Array(20).fill(0).map(v=>Array(20).fill(v)),
	};

	let inputRange;

	let filter = {
		padding: {left: 0,right:0,top:0,bottom:0},
		mask: {size: {x:5,y:5}, values: Array(5).fill(0).map(v=>Array(5).fill(v)),},
		paddingType: 'zero',
		function: 'identity',
		flip: true,
	};

	const paddingTypes = {
		zero: (original, x, y) => {
			if(original.values && y >= 0 && y < original.values.length) {
				if(x >= 0 && x < original.values[0].length) {
					return original.values[y][x]
				} else {
					return 0
				}
			} else {
				return 0
			}
		},
		replicate: (original, x, y) => {
			if(y<0) y = 0;
			if(x<0) x = 0;
			if(y>=original.values.length) y = original.values.length-1;
			if(x>=original.values[y].length) x = original.values[y].length-1;
			return original.values[y][x]
		},
		mirror: (original, x, y) => {
			const modY = (original.size.y)*2
			const modX = (original.size.x)*2

			const doubleY = ((y % modY)+modY)%modY
			const doubleX = ((x % modX)+modX)%modX

			const yover = (doubleY/original.size.y)<<0
			const xover = (doubleX/original.size.x)<<0

			y = ((original.size.y-yover+(yover?-1:1)*y)%original.size.y+original.size.y)%original.size.y
			x = ((original.size.x-xover+(xover?-1:1)*x)%original.size.x+original.size.x)%original.size.x
			
			return original.values[y][x]
		},
		reflect: (original, x, y) => {
			const modY = (original.size.y)*2
			const modX = (original.size.x)*2

			const doubleY = ((y % modY)+modY)%modY
			const doubleX = ((x % modX)+modX)%modX

			const yover = (doubleY/original.size.y)<<0
			const xover = (doubleX/original.size.x)<<0

			y = ((original.size.y-2*yover+(yover?-1:1)*(y))%(original.size.y-yover)+original.size.y)%(original.size.y-yover)
			x = ((original.size.x-2*xover+(xover?-1:1)*(x))%(original.size.x-xover)+original.size.x)%(original.size.x-xover)
			
			return original.values[y][x]
		},
		cyclic: (original, x, y) => {
			x = (x%original.size.x + original.size.x)%original.size.x
			y = (y%original.size.y + original.size.y)%original.size.y
			
			return original.values[y][x]
		},
	}

	const functions = {
		identity: (x) => x,
		ceil: (x) => Math.ceil(x),
		floor: (x) => Math.floor(x),
		abs: (x) => Math.abs(x),
		round: (x) => Math.round(x),
		relu: (x) => x>=0?x:0,
	}

	$: paddedImage = {
		size: {
			x: inputImage.size.x + filter.padding.left + filter.padding.right,
			y: inputImage.size.y + filter.padding.top + filter.padding.bottom,
		},
		values: Array(inputImage.size.y + filter.padding.top + filter.padding.bottom).fill(0).map((v,y) => 
			Array(inputImage.size.x + filter.padding.left + filter.padding.right).fill(0).map((_,x) => paddingTypes[filter.paddingType](inputImage, x-filter.padding.left,y-filter.padding.top))
			)
	}

	$: filterNorm = Math.abs(filter.mask.values.flat().reduce((a,b)=>a+b, 0)) || 1

	$: filteredImage = {
		size: {
			x: paddedImage.size.x - filter.mask.size.x + 1,
			y: paddedImage.size.y - filter.mask.size.y + 1,
		},
		values: Array(paddedImage.size.y - filter.mask.size.y + 1).fill(0).map((v,y) => 
			Array(paddedImage.size.x - filter.mask.size.x + 1).fill(0).map((_,x) => 
				functions[filter.function](filter.mask.values.flatMap((row, fy) => row.map((w, fx) => paddedImage.values[y+(filter.flip?(filter.mask.size.y-fy-1):fy)][x+(filter.flip?(filter.mask.size.x-fx-1):fx)]*w)).reduce((a,b)=>a+b, 0)/filterNorm)
			)
		)
	}

	$: filteredImageMin = Math.min.call(Math, 0, ...filteredImage.values.flat())
	$: filteredImageMax = Math.max.call(Math, 1, ...filteredImage.values.flat())
</script>

<style>
	.intensity {
		fill: hsl(0deg 0% calc(var(--intensity, 0.5)*100%));
		accent-color: hsl(0deg 0% calc(var(--intensity, 0.5)*100%));
	}

	.options {
		display: grid;
		grid-template-columns: auto 3em auto;
		justify-content: start;
		gap: 1em;
	}
	.options-list {
		display: grid;
		grid-template-columns: auto auto;
		justify-content: start;
		gap: 1em;
	}

  
   .option-container {
		display: flex;
		 flex-direction: column;
		gap: 1em;
		border: none;
		background: #333;
		color: #fff;
	}

	.option-container-label {
		display: inline-block;
		padding: 2px 0.5em;
		background: inherit;
	}

	dd, dt, dl {
		margin: 0;
		padding: 0;
	}

	dt, dd {
		font-size: 0.8em;
		align-self: center;
	}

	section {
		display: grid;
		grid-template-columns: auto auto;
		justify-content: start;
	}

	.container {
		display: flex;
		flex-direction: column;
		align-items: start;
	}
	.padding-options {
		display: grid;
		grid-template-rows: auto auto;
		gap: 0 0.2em;
	}

	.padding-options > dt {
		grid-row: 2;
		text-align: center;
		font-size: 0.8em;
	}
	.padding-options > dd {
		grid-row: 1;
		text-align: center;
	}

	.image-container {
		background: #000;
		color: #fff;
		padding: 0.8em;
		display: flex;
		flex-direction: column;
		gap: 0.3em;
	}

	h2 {
		margin: 0;
		text-align: center;
	}

	.brush {
		margin: 1em auto;
	}
	
	.image-array {
		max-width: 18em;
		max-height: 24em;
		width: 100%;
		min-width: 20em;
		height: auto;
		border: 1px solid white;
	}

	.filter-chain {
		display: flex;
		justify-content: center;
		flex-wrap: wrap;
		align-items: start;
		gap: 1em;
	}

	.padding-field {
		width: 100%;
		flex-grow: 1;
		flex-shrink: 1;
	}

	select, input {
		margin: 0;
	}

	h2 {
		font-size: 1.2em;
		margin-bottom: 0.5em;
	}
</style>

<div class="container">
	
<section class="brush">
	<fieldset class="option-container">
	<legend class="option-container-label">Brush</legend>
	<dl class="options">
	<dt>Brightness </dt>
	<dd>	<svg style="border:1px solid black" viewBox="0 0 10 10" width={10} height={10}><rect x="0" y="0" width="10" height="10" style:--intensity={brush.value} fill="magenta" class="intensity" /></svg>
</dd>
	<dd><input style:--intensity={brush.value} class="intensity" type="range" min="0" max="1" step="0.05" bind:value={brush.value}/></dd>
	<dt>Size</dt>
	<dd>{brush.size}</dd>
	<dd><input type="range" min="1" max="6" step="1" bind:value={brush.size}/></dd>
	<dt>Hardness</dt>
	<dd>{Math.round(brush.hardness*100)}%</dd>
	<dd><input type="range" min="0" max="1" step="0.05" bind:value={brush.hardness}/></dd>
</dl>
</fieldset>

</section>



<div>	
	<div class="filter-chain">

<div class="image-container">
	<h2>Input Image</h2>
	<Canvas bind:range={inputRange} bind:image={inputImage} brush={brush} />
</div>

<div class="image-container">
	<h2>Padding</h2>
	<fieldset class="option-container">
	<legend class="option-container-label">Options</legend>
		<dl class="options">
			<dt>Type</dt>
			<dd>
				<select bind:value={filter.paddingType}>
					{#each Object.keys(paddingTypes) as type}
						<option value={type}>{type.charAt(0).toUpperCase()
  + type.slice(1)}</option>
					{/each}
				</select>
			</dd>
		</dl>
	<dl class="padding-options">
	<dt>Top</dt>
	<dd><input class="padding-field" type="number" size="5" min="0" max="50" step="1" bind:value={filter.padding.top} /></dd>
	<dt>Right</dt>
	<dd><input class="padding-field" type="number" size="5" min="0" max="50" step="1" bind:value={filter.padding.right} /></dd>
	<dt>Bottom</dt>
	<dd><input class="padding-field" type="number" size="5" min="0" max="50" step="1" bind:value={filter.padding.bottom} /></dd>
	<dt>Left</dt>
	<dd><input class="padding-field" type="number" size="5" min="0" max="50" step="1" bind:value={filter.padding.left} /></dd>
	</dl>
	</fieldset>

	<div>
			<h2>Padded Image</h2>
		<svg role="presentation" class="image-array no-cursor" viewBox="0 0 {paddedImage.size.x} {paddedImage.size.y}" width="{paddedImage.size.x}" height="{paddedImage.size.y}" preserveAspectRatio="meet xMidYMid">
		{#each paddedImage.values as row, y}
			{#each row as value, x}
				<rect pointer-events="none" class="intensity" style:--intensity={(value-inputRange)/(1-inputRange)} fill="magenta" image-rendering="crisp-edges" stroke="#abb3" {x} {y} width="1" height=1  vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
			{/each}
		{/each}
			<rect stroke="cyan" fill="none" x={filter.padding.left} y={filter.padding.top} width={inputImage.size.x} height={inputImage.size.y} vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
		</svg>
	</div>

	
	
</div>
<div class="image-container">
	<h2>Filter</h2>
	<fieldset class="option-container">
	<legend class="option-container-label">Operation</legend>
		<dl class="options-list">
			<dt>Vector Operation</dt>
			<dd>
				<select bind:value={filter.flip}>
					<option value={true}>Convolution</option>
					<option value={false}>Correlation</option>
				</select>
			</dd>
			<dt>Element-wise Operation</dt>
			<dd>
				<select bind:value={filter.function}>
					{#each Object.keys(functions) as fun}
						<option value={fun}>{fun.charAt(0).toUpperCase()
  + fun.slice(1)}</option>
					{/each}
				</select>
			</dd>
		</dl>
	</fieldset>
	<h2>Kernel</h2>
	<Canvas bind:image={filter.mask} brush={brush} />
</div>
<div class="image-container">
		<h2>Filtered Image</h2>
	<svg style:flx-shrink="0" role="presentation" class="image-array no-cursor" viewBox="0 0 {filteredImage.size.x} {filteredImage.size.y}" width="{filteredImage.size.x}" height="{filteredImage.size.y}" preserveAspectRatio="meet xMidYMid">
	{#each filteredImage.values as row, y}
		{#each row as value, x}
			<rect pointer-events="none" class="intensity" style:--intensity={(value-filteredImageMin)/((filteredImageMax-filteredImageMin)||1)} fill="magenta" image-rendering="crisp-edges" stroke="#abb3" {x} {y} width="1" height=1  vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
		{/each}
	{/each}
	</svg>
</div>
	</div>
</div>
</div>
