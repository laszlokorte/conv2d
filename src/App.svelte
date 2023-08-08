<script>
	import Canvas from './Canvas.svelte'
	import Plot from './Plot.svelte'

	let numberFormatter = new Intl.NumberFormat('en-IN', { minimumFractionDigits: 1, maximumFractionDigits: 1, signDisplay: 'exceptZero', trailingZeroDisplay: 'auto' })

	let brushSticky = false
	let brush = {
		active: false,
		value: 1,
		size: 1,
		hardness: 0,
		element: null,
		prev:{x:null,y:null},
		current:{x:null,y:null},
	}

	let clipOutput = false
	let focus = null

	function focusInput(evt) {
		const newFocus = {
			type: 'input',
			x: Math.floor(evt.detail.x),
			y: Math.floor(evt.detail.y),
		}

		if(evt.detail.initial && focus && newFocus.type == focus.type && newFocus.x == focus.x && newFocus.y == focus.y) {
			focus = null
		} else {
			focus = newFocus
		}
	}

	function focusOutput(evt) {
		const newFocus = {
			type: 'output',
			x: Math.floor(evt.detail.x),
			y: Math.floor(evt.detail.y),
		}

		if(evt.detail.initial && focus && newFocus.type == focus.type && newFocus.x == focus.x && newFocus.y == focus.y) {
			focus = null
		} else {
			focus = newFocus
		}
	}

	function mod(a,b) {
		return ((a%b)+ b)%b
	}

	const fullExamples = {
		"Box Blur": {
			filter: {
				flip: false,
				function: 'identity',
				normalize: true,
				padding: {left: 2,right:2,top:2,bottom:2},
				kernel: {size: {x:5,y:5}, values: [[0,0,0,0,0],[0,1,1,1,0],[0,1,1,1,0],[0,1,1,1,0],[0,0,0,0,0]]},
				paddingType: 'zero',
			},
			input: 'Blobs',
			range: 0,
		},
		"Vertical Edges": {
			filter: {
				flip: false,
				function: 'identity',
				normalize: true,
				padding: {left: 2,right:2,top:2,bottom:2},
				kernel: {size: {x:5,y:5}, values: [[0,0,0,0,0],[0,-1,-1,-1,0],[0,0,0,0,0],[0,1,1,1,0],[0,0,0,0,0]]},
				paddingType: 'zero',
			},
			input: 'Blobs',
			range: -1,
		},
		erosion: {
			filter: {
				flip: false,
				function: 'floor',
				normalize: true,
				padding: {left: 2,right:2,top:2,bottom:2},
				kernel: {size: {x:5,y:5}, values: [[0,0,0,0,0],[0,1,1,1,0],[0,1,1,1,0],[0,1,1,1,0],[0,0,0,0,0]]},
				paddingType: 'zero',
			},
			input: 'Letter H',
			range: 0,
		},
		dilation: {
			filter: {
				flip: true,
				function: 'ceil',
				normalize: true,
				padding: {left: 2,right:2,top:2,bottom:2},
				kernel: {size: {x:5,y:5}, values: [[0,0,0,0,0],[0,1,1,1,0],[0,1,1,1,0],[0,1,1,1,0],[0,0,0,0,0]],},
				paddingType: 'zero',
			},
			input: 'Letter H',
			range: 0,
		}
	}

	function loadExample(name) {
		let ex = fullExamples[name]

		if(ex) {
			filter = {
				...JSON.parse(JSON.stringify(filter)),
				...JSON.parse(JSON.stringify(ex.filter)),
			}
			inputImage = JSON.parse(JSON.stringify(imageExamples[ex.input]))
			filterRange = ex.range 
			focus = null
		}
	}

	let imageExamples = {

		"Letter H": {
			size:{x:11,y:12},
			values:[
				[0,0,0,0,0,0,0,0,0,0,0],
				[0,0,0,0,0,0,0,0,0,0,0],
				[0,1,1,1,0,0,0,1,1,1,0],
				[0,1,1,1,0,0,0,1,1,1,0],
				[0,1,1,1,0,0,0,1,1,1,0],
				[0,1,1,1,1,1,1,1,1,1,0],
				[0,1,1,1,1,1,1,1,1,1,0],
				[0,1,1,1,0,0,0,1,1,1,0],
				[0,1,1,1,0,0,0,1,1,1,0],
				[0,1,1,1,0,0,0,1,1,1,0],
				[0,0,0,0,0,0,0,0,0,0,0],
				[0,0,0,0,0,0,0,0,0,0,0],
			]
		},
		"House": {"size":{"x":20,"y":20},"values":[[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,1,0,0,0,0],[0,0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,1,1,0,1,1,1,0,0,0,0,0],[0,0,0,0,0,0,0,0,1,0,0,1,1,0,1,0,0,0,0,0],[0,0,0,0,0,0,0,1,0,0,0,0,1,0,1,0,0,0,0,0],[0,0,0,0,0,0,1,0,0,0,0,0,0,1,1,0,0,0,0,0],[0,0,0,0,0,1,0,0,0,0,0,0,0,0,1,0,0,0,0,0],[0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0],[0,0,0,0,1,1,1,1,1,1,1,1,1,1,1,1,0,0,0,0],[0,0,0,0,1,0,0,0,0,0,0,0,0,0,0,1,0,0,0,0],[0,0,0,0,1,0,1,1,1,0,0,0,0,0,0,1,0,0,0,0],[0,0,0,0,1,0,1,0,1,0,1,1,1,0,0,1,0,0,0,0],[0,0,0,0,1,0,1,1,1,0,1,0,1,0,0,1,0,0,0,0],[0,0,0,0,1,0,0,0,0,0,1,0,1,0,0,1,0,0,0,0],[0,0,0,0,1,0,0,0,0,0,1,0,1,0,0,1,0,0,0,0],[1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1,1],[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0]]},
		"Blobs": {"size":{"x":20,"y":20},"values":[[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0],[0,0,0,0,0,0,0,0,0,0,0.06741105785966839,0.6830337183730698,0.9408962667007635,0.9408962667007635,0.6830337183730697,0.06741105785966839,0,0,0,0],[0,0,0,0,0.3459978405115438,0.8997983397082998,0.9563857920747757,0.7697912399916262,0.003046677513905486,0,0.91503054901673,0.9999844462893918,0.9999997991514642,0.9999967428506753,0.9981602186800108,0.6830337183730698,0,0,0,0],[0,0,0,0,0.9189665165233625,0.9999905766831179,0.999999907813869,0.9998175061862963,0.00019526039375020228,0,0.9999830958721263,1,1,1,0.9999988929672404,0.9448805118859208,0,0,0,0],[0,0,0,0,0.9899596272747867,0.9999999996142679,0.9999999999999899,0.9999999925298366,0.0002464185329440199,0,0.9939936836926341,1,1,1,0.9999999655402785,0.9812661094258712,0,0,0,0],[0,0,0,0,0.9899596272747868,0.9999999998321046,0.9999999999999996,0.9999999996142679,0.015728775477695665,0,0.7500633984234484,0.9999999999999858,1,1,0.9999999678632447,0.9825289808073007,0,0,0,0],[0,0,0,0,0.9189665165233625,0.9999966829924594,0.999999998324425,0.9999966829924594,0.9189665165233625,0,0.00025358935655285194,0.9999999998923039,1,1,0.9999999655402785,0.9825289808073007,0,0,0,0],[0,0,0,0,0.3459978405115438,0.9189665165233625,0.9846477988190947,0.9189665165233625,0.005510537805870657,0,0.7500633973391382,0.9999999998349105,1,1,0.9999999655402785,0.9825289808073007,0,0,0,0],[0,0,0,0,0,0,0,0,0.0000039754777486101065,0.00000403879323172909,0.9939936916803309,1,1,1,0.9999999655402785,0.9825289808073007,0,0,0,0],[0,0,0,0.5,0.9609375,0.987796015794419,0.7500010096983079,0.00000403879323172909,0.00025365267203597097,0.7539816112263549,0.9999999911876531,1,1,1,0.9999999678632447,0.9825289808073007,0,0,0,0],[0,0,0.75,0.9997615834948269,1,1,0.9999994180710364,0.9999633402701824,0.9999999693299602,0.9999999672971279,1,1,1,1,0.9999999678632447,0.9812661094258713,0,0,0,0],[0,0,0.9755859375,1,1,1,1,1,1,1,1,1,1,1,0.9999996237441368,0.9448805118859208,0,0,0,0],[0,0,0.75,0.9997837336611741,0.9999999999999959,1,1,1,1,1,1,1,1,0.9999999993338766,0.999680067532486,0.6830337183730697,0,0,0,0],[0,0,0,0.8986197139408392,0.9999983869144988,1,1,1,1,1,0.9999999999951656,0.9999999944115143,0.9999996237441368,0.999680067532486,0.8922701962798048,0.06741105785966839,0,0,0,0],[0,0,0,0.6717778134857498,0.9968442339240656,0.9999973453063101,0.9999999408914423,0.9999999689653283,0.9999999678632447,0.9999996491079426,0.9998985921954003,0.9936327664125425,0.9448805118859208,0.6830337183730698,0.06741105785966839,0.03429355281207136,0.41700960219478733,0.5829903978052127,0.41700960219478733,0.03429355281207136],[0,0,0,0.06741105785966839,0.6830337183730697,0.9448805118859208,0.9812661094258713,0.9825289808073007,0.9812661094258712,0.9448805118859208,0.6830337183730698,0.06741105785966839,0,0,0,0.41700960219478733,0.8737997256515775,0.9657064471879286,0.8737997256515775,0.41700960219478733],[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0.5829903978052127,0.9657064471879286,1,0.9657064471879286,0.5829903978052127],[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0.41700960219478733,0.8737997256515775,0.9657064471879286,0.8737997256515775,0.41700960219478733],[0,0,0,0,0,0,0,0,0,0,0,0,0,0,0,0.03429355281207136,0.41700960219478733,0.5829903978052127,0.41700960219478733,0.03429355281207136]]}
	}

	let kernelExamples = {
		"Identity": {
			size:{x:5,y:5},
			values:[[0,0,0,0,0],
					[0,0,0,0,0],
					[0,0,1,0,0],
					[0,0,0,0,0],
					[0,0,0,0,0],]
		},
		"PrewittVertical": {
			size:{x:5,y:5},
			values:[[0,0,0,0,0],
					[0,1,1,1,0],
					[0,0,0,0,0],
					[0,-1,-1,-1,0],
					[0,0,0,0,0],],
			range: -1
		},
		"PrewittHorizontal": {
			size:{x:5,y:5},
			values:[[0,0,0,0,0],
					[0,1,0,-1,0],
					[0,1,0,-1,0],
					[0,1,0,-1,0],
					[0,0,0,0,0],],
			range: -1
		},
		"SobelVertical": {
			size:{x:5,y:5},
			values:[[0,0,0,0,0],
					[0,1/2,2/2,1/2,0],
					[0,0,0,0,0],
					[0,-1/2,-2/2,-1/2,0],
					[0,0,0,0,0],],
			range: -1
		},
		"SobelHorizontal": {
			size:{x:5,y:5},
			values:[[0,0,0,0,0],
					[0,1/2,0,-1/2,0],
					[0,2/2,0,-2/2,0],
					[0,1/2,0,-1/2,0],
					[0,0,0,0,0],],
			range: -1
		},
		"3x3Box": {
			size:{x:5,y:5},
			values:[[0,0,0,0,0],
					[0,1,1,1,0],
					[0,1,1,1,0],
					[0,1,1,1,0],
					[0,0,0,0,0],]
		},
		"3x3Gauss": {
			size:{x:5,y:5},
			values:[[0,0,0,0,0],
					[0,1/4,2/4,1/4,0],
					[0,2/4,4/4,2/4,0],
					[0,1/4,2/4,1/4,0],
					[0,0,0,0,0],]
		},
		"3x3Sharpen": {
			size:{x:5,y:5},
			values:[[0,0,0,0,0],
					[0,0,-1/5,0,0],
					[0,-1/5,1,-1/5,0],
					[0,0,-1/5,0,0],
					[0,0,0,0,0],],
			range: -1
		},

		"3x3Cross": {
			size:{x:5,y:5},
			values:[[0,0,0,0,0],
					[0,0,1,0,0],
					[0,1,1,1,0],
					[0,0,1,0,0],
					[0,0,0,0,0],]
		},

		"3x3Diamond": {
			size:{x:5,y:5},
			values:[[0,0,1,0,0],
					[0,1,1,1,0],
					[1,1,1,1,1],
					[0,1,1,1,0],
					[0,0,1,0,0],]
		},
	}

	let inputImage = {
		size: {x:20,y:20},
		values: Array(20).fill(0).map(v=>Array(20).fill(v)),
	};

	let inputRange, filterRange;

	let filter = {
		padding: {left: 0,right:0,top:0,bottom:0},
		kernel: {size: {x:5,y:5}, values: Array(5).fill(0).map(v=>Array(5).fill(v)),},
		paddingType: 'zero',
		function: 'identity',
		flip: true,
		normalize: true,
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

			const doubleY = mod(y, modY)
			const doubleX = mod(x,modY)

			const yover = (doubleY/original.size.y)<<0
			const xover = (doubleX/original.size.x)<<0

			y = mod(original.size.y-yover+(yover?-1:1)*y, original.size.y)
			x = mod(original.size.x-xover+(xover?-1:1)*x, original.size.x)
			
			return original.values[y][x]
		},
		reflect: (original, x, y) => {
			const modY = (original.size.y)*2
			const modX = (original.size.x)*2

			const doubleY = mod(y, modY)
			const doubleX = mod(x, modX)

			const yover = (doubleY/original.size.y)<<0
			const xover = (doubleX/original.size.x)<<0

			y = ((original.size.y-2*yover+(yover?-1:1)*(y))%(original.size.y-yover)+original.size.y)%(original.size.y-yover)
			x = ((original.size.x-2*xover+(xover?-1:1)*(x))%(original.size.x-xover)+original.size.x)%(original.size.x-xover)
			
			return original.values[y][x]
		},
		cyclic: (original, x, y) => {
			x = mod(x,original.size.x)
			y = mod(y,original.size.y)
			
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

	$: filterNorm = Math.abs(filter.kernel.values.flat().reduce((a,b)=>a+b, 0)) || 1
	$: filterFlipped = filter.kernel.values.toReversed().map(r=>r.toReversed())

	$: filteredImage = {
		size: {
			x: Math.max(0, Math.abs(paddedImage.size.x - filter.kernel.size.x) + 1),
			y: Math.max(0, Math.abs(paddedImage.size.y - filter.kernel.size.y) + 1),
		},
		values: Array(Math.max(0, Math.abs(paddedImage.size.y - filter.kernel.size.y) + 1)).fill(0).map((v,y) => 
			Array(Math.max(0, Math.abs(paddedImage.size.x - filter.kernel.size.x) + 1)).fill(0).map((_,x) => 
				functions[filter.function](filter.kernel.values.flatMap((row, fy) => row.map((w, fx) => paddedImage.values[mod(y+fy, paddedImage.size.y)][mod(x+fx, paddedImage.size.x)]*
					filter.kernel.values[mod((filter.flip?-1:1)*fy+(filter.flip?-1:0), filter.kernel.size.y)][mod((filter.flip?-1:1)*fx+(filter.flip?-1:0), filter.kernel.size.x)]
					)).reduce((a,b)=>a+b, 0)/(filter.normalize?filterNorm:1))
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
		gap: 0.2em 1em;
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
		display: flex;
		gap: 0.2em;
		padding: 2px 0.5em;
		background: inherit;
		white-space: nowrap;
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
		padding: 1em;
	}

	.brush.sticky {
		position: sticky;
		top: 0;
		background: #fffa;

	}
	
	.image-array {
		max-width: 18em;
		max-height: 24em;
		width: 100%;
		min-width: 20em;
		height: auto;
		border: 1px solid white;
		display: block;
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

	.small-button {
		color: inherit;
		font: inherit;
		background: none;
		border: none;
		text-decoration: underline;
		cursor: pointer;
	}
</style>

<div class="container">

		<div style:margin="auto">
			<strong>Examples</strong>
			<button class="small-button" on:click={() => loadExample('Box Blur')}>Box Blur</button>
			<button class="small-button" on:click={() => loadExample('Vertical Edges')}>Vertical Edges</button>
			<button class="small-button" on:click={() => loadExample('erosion')}>Erosion</button>
			<button class="small-button" on:click={() => loadExample('dilation')}>Dilation</button>
		</div>
	
<section class="brush" class:sticky={brushSticky}>
	<fieldset class="option-container">
	<legend class="option-container-label">Brush / <label><input type="checkbox" bind:checked={brushSticky}> Pinned</label></legend>
	<dl class="options">
	<dt>Intensity </dt>
	<dd>	<svg style:cursor="pointer" style="border:1px solid black" viewBox="0 0 10 10" width={20} height={20}><rect x="0" y="0" width="10" height="10" style:--intensity={brush.value} on:click={() => {brush.value = 1-brush.value}} fill="magenta" class="intensity" /></svg>
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
	<Canvas examples={imageExamples}  bind:range={inputRange} bind:image={inputImage} brush={brush} />
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
		<Plot size={paddedImage.size} on:point={focusInput}>
		{#each paddedImage.values as row, y}
			{#each row as value, x}
				<rect pointer-events="none" data-x={x} data-y={y} cursor="pointer" class="intensity" style:--intensity={(value-inputRange)/(1-inputRange)} fill="magenta" image-rendering="crisp-edges" stroke="#abb3" {x} {y} width="1" height=1  vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
			{/each}
		{/each}
		{#if (focus && focus.type == 'input')}
		<rect pointer-events="none" fill="magenta" fill-opacity="0.7" image-rendering="crisp-edges" stroke="magenta" x={focus.x} y={focus.y} width="1" height="1"  vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
		{/if}

		{#if (focus && focus.type == 'output')}
		<rect pointer-events="none" fill-opacity="0.2" fill="magenta" image-rendering="crisp-edges" stroke="magenta" x={focus.x} y={focus.y} width="{filter.kernel.size.x}" height="{filter.kernel.size.y}"  vector-effect="non-scaling-stroke" stroke-width="1px"></rect>

		<svg pointer-events="none" x={focus.x} y={focus.y} width="{filter.kernel.size.x}" height="{filter.kernel.size.y}"  vector-effect="non-scaling-stroke">
			{#each (filter.flip?filterFlipped:filter.kernel.values) as row, y}
			{#each row as value, x}
				<rect fill="hsla(300deg 100% {(value-filterRange)/((1-filterRange)||1)*50}%)" fill-opacity="0.5" image-rendering="crisp-edges" {x} {y} width="1" height=1  vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
			{/each}
		{/each}
		</svg>

		{/if}
			<rect pointer-events="none" stroke="cyan" fill="none" x={filter.padding.left} y={filter.padding.top} width={inputImage.size.x} height={inputImage.size.y} vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
		</Plot>
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
			<dt>Normalize</dt>
			<dd>
				<input type="checkbox" bind:checked={filter.normalize} />
			</dd>
		</dl>
	</fieldset>
	<h2>Kernel</h2>
	<Canvas  bind:range={filterRange} maxSize={10} examples={kernelExamples} bind:image={filter.kernel} brush={brush} />
</div>
<div class="image-container">
		<h2>Filtered Image</h2>
	<div class="figure-with-legend">
		<Plot size={filteredImage.size} on:point={focusOutput}>
	{#each filteredImage.values as row, y}
		{#each row as value, x}
			<rect data-x={x} data-y={y} cursor="pointer" class="intensity" style:--intensity={!clipOutput?(value-filteredImageMin)/((filteredImageMax-filteredImageMin)||1):Math.max(0,Math.min(1, value))} fill="magenta" image-rendering="crisp-edges" stroke="#abb3" {x} {y} width="1" height=1  vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
		{/each}
	{/each}
		{#if (focus && focus.type == 'output')}
		<rect pointer-events="none" fill-opacity="0.7" fill="magenta" image-rendering="crisp-edges" stroke="magenta" x={focus.x} y={focus.y} width="1" height="1"  vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
		{/if}
		{#if (focus && focus.type == 'input')}
		<rect pointer-events="none" fill-opacity="0.7" fill="magenta" image-rendering="crisp-edges" stroke="magenta" x={focus.x-filter.kernel.size.x+1} y={focus.y-filter.kernel.size.y+1} width="{filter.kernel.size.x}" height="{filter.kernel.size.y}"  vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
		<svg image-rendering="crisp-edges" pointer-events="none" x={focus.x-filter.kernel.size.x+1} y={focus.y-filter.kernel.size.y+1} width="{filter.kernel.size.x}" height="{filter.kernel.size.y}"  vector-effect="non-scaling-stroke">
			{#each (!filter.flip?filterFlipped:filter.kernel.values) as row, y}
			{#each row as value, x}
				<rect fill="hsla(300deg 100% {(value-filterRange)/((1-filterRange)||1)*50}%)" fill-opacity="0.5" image-rendering="crisp-edges" {x} {y} width="1" height=1  vector-effect="non-scaling-stroke" stroke-width="1px"></rect>
			{/each}
		{/each}
		</svg>

		{/if}
	</Plot>
	<div class="legend">
		<span class="legend-label">{numberFormatter.format(!clipOutput?filteredImageMax:Math.max(0,Math.min(1, filteredImageMax)))}</span>
		<span class="legend-label">{numberFormatter.format(!clipOutput?filteredImageMin:Math.max(0,Math.min(1, filteredImageMin)))}</span>
	</div>
	</div>
	<label><input type="checkbox" bind:checked={clipOutput} /> Clip output to range 0.0 to 1.0</label>
</div>
	</div>
</div>
</div>
