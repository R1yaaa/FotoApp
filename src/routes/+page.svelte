<script lang="ts">
    import { onMount } from 'svelte';
    import { render } from 'svelte/server';
	import { base } from '$app/paths';

    let videoRef = $state<any>();
    let canvasRef = $state<any>();
    let originalPhotoUrl = $state<string | null>(null);
	let photoUrl = $state<string | null>(null);
	let stream = $state<MediaStream>();
	let facingMode = $state('user');
	let overlayText = $state('');
	let layoutIndex = $state(0);
	const TARGET_ASPECT_RATIO = 4 / 3; 
	const BAR_HEIGHT_RATIO = 0.15; 

	const layouts = [
		{
			name: 'layout1', 
			footer: `${base}/images/gelb.jpg`,
			textColor: 'black'
		},
		{
			name: 'layout2',
			footer: `${base}/images/gelb_text1.jpg`,
			textColor: 'black'
		},
		{
			name: 'layout3',
			footer: `${base}/images/gelb_text2.jpg`,
			textColor: 'black'
		},
		{
			name: 'layout4',
			footer: `${base}/images/gruen.jpg`,
			textColor: 'white'
		},
		{
			name: 'layout5',
			footer: `${base}/images/gruen_text1.jpg`,
			textColor: 'white'
		},
		{
			name: 'layout6',
			footer: `${base}/images/gruen_text2.jpg`,
			textColor: 'white'
		},
		{
			name: 'layout7',
			footer: `${base}/images/weltraum1.jpg`,
			textColor: 'white'
		},
		{
			name: 'layout8',
			footer: `${base}/images/weltraum2.jpg`,
			textColor: 'white'
		},
		{
			name: 'layout9',
			footer: `${base}/images/jubilaeum.jpg`,
			textColor: 'white'
		}
	]
	const currentLayout = () => layouts[layoutIndex];
	let logo: HTMLImageElement;

	

    async function startCamera() {
		try {
			stream = await navigator.mediaDevices.getUserMedia({
				audio: false,
				video: {
				facingMode: facingMode
				}
			});

			videoRef.srcObject = stream;

			console.log("Camera started.")
		} catch (err) {
			console.error("Could not start camera: ", err);
		}

    }

	function stopCamera() {
		stream?.getTracks().forEach((track) => track.stop());
		stream = undefined;

		if (videoRef) {
			videoRef.srcObject = null;
		}
	}

    function takePhoto() {
		const canvas = canvasRef;
		const video = videoRef;
		const context = canvas.getContext('2d');
		const videoRatio = video.videoWidth / video.videoHeight;

		let sourceX = 0;
		let sourceY = 0;
		let sourceWidth = video.videoWidth;
		let sourceHeight = video.videoHeight;

		if (videoRatio > TARGET_ASPECT_RATIO) {
			sourceWidth = video.videoHeight * TARGET_ASPECT_RATIO;
			sourceX = (video.videoWidth - sourceWidth) / 2;
		} else {
			sourceHeight = video.videoWidth / TARGET_ASPECT_RATIO;
			sourceY = (video.videoHeight - sourceHeight) / 2;
		}

		canvas.width = sourceWidth;
		canvas.height = sourceHeight;

		context.drawImage(
			video,
			sourceX,
			sourceY,
			sourceWidth,
			sourceHeight,
			0,
			0,
			sourceWidth,
			sourceHeight
		);

		originalPhotoUrl = canvas.toDataURL('image/jpeg', 0.95);

		renderFinalPhoto();
	}

	function renderFinalPhoto() {
		if (!originalPhotoUrl) return;

		if (!logo.complete) {
			logo.onload = renderFinalPhoto;
			return;
		}

		const canvas = canvasRef;
		const context = canvas.getContext('2d');

		const image = new Image();

		image.onload = () => {
			canvas.width = image.width;
			canvas.height = image.height;

			context.drawImage(image, 0, 0);

			const logoWidth = canvas.width * 0.1;
			const logoHeight = logoWidth * (logo.height / logo.width);
			const padding = canvas.width * 0.03;

			context.imageSmoothingEnabled = true;
			context.imageSmoothingQuality = 'high';

			context.drawImage(
				logo,
				canvas.width - logoWidth - padding,
				padding,
				logoWidth,
				logoHeight
			);

			const barHeight = canvas.height * BAR_HEIGHT_RATIO;
			const footer = new Image();

			footer.onload = () => {
				drawImageContain(
					context,
					footer,
					0,
					canvas.height - barHeight,
					canvas.width,
					barHeight,
					'#000000'
				);

				context.fillStyle = currentLayout().textColor;
				context.font = `${canvas.width * 0.04}px Arial`;
				context.textAlign = 'center';
				context.textBaseline = 'middle';

				context.fillText(
					overlayText,
					canvas.width / 2,
					canvas.height - barHeight / 2
				);

				photoUrl = canvas.toDataURL('image/jpeg', 0.95);
			};

			footer.src = currentLayout().footer;
		};

		image.src = originalPhotoUrl;
	}

    function removePhoto(){
        photoUrl = null;
		originalPhotoUrl = null;
        setTimeout(() => {
            startCamera();
        });
    }

	function switchCamera() {
		stopCamera();
		facingMode = facingMode === 'user' ? 'environment' : 'user';
		startCamera();
	}

	function savePhoto () {
		if (!photoUrl) {
			console.warn("No photo available to save.");
			return
		}

		const link = document.createElement('a');
		link.href = photoUrl;
		link.download = 'gruppenfoto.jpg';
		link.click();
	}

	function printPhoto() {
		if (!photoUrl) {
			console.warn("No photo available to print.");
			return;
		}
		window.print();
	}

	async function sharePhoto() {
		if (!photoUrl) {
			console.warn("No photo available to share.");
			return;
		}

		const response = await fetch(photoUrl);
		const blob = await response.blob();

		const file = new File([blob], "gruppenfoto.jpg", {
			type: "image/jpeg"
		});

		if (navigator.canShare && navigator.canShare({ files: [file] })) {
			await navigator.share({
				files: [file],
				title: "Gruppenfoto",
				text: "Hier ist das Gruppenfoto."
			});
		} else {
			console.warn("Sharing files is not supported on this device/browser.");
		}
	}

	function changeLayout() {
		layoutIndex = (layoutIndex + 1) % layouts.length;

		if(originalPhotoUrl) {
			renderFinalPhoto();
		}
	}

	function drawImageCover(
		context: CanvasRenderingContext2D,
		image: HTMLImageElement,
		x: number,
		y: number,
		width: number,
		height: number
	) {
		const imageRatio = image.width / image.height;
		const targetRatio = width / height;

		let sourceX = 0;
		let sourceY = 0;
		let sourceWidth = image.width;
		let sourceHeight = image.height;

		if (imageRatio > targetRatio) {
			sourceWidth = image.height * targetRatio;
			sourceX = (image.width - sourceWidth) / 2;
		} else {
			sourceHeight = image.width / targetRatio;
			sourceY = (image.height - sourceHeight) / 2;
		}

		context.drawImage(
			image,
			sourceX,
			sourceY,
			sourceWidth,
			sourceHeight,
			x,
			y,
			width,
			height
		);
	}

	function drawImageContain(
		context: CanvasRenderingContext2D,
		image: HTMLImageElement,
		x: number,
		y: number,
		width: number,
		height: number,
		bgColor: string = '#000000'
	) {
		context.save();
		context.fillStyle = bgColor;
		context.fillRect(x, y, width, height);

		const imageRatio = image.width / image.height;
		const targetRatio = width / height;

		let drawWidth: number;
		let drawHeight: number;

		if (imageRatio > targetRatio) {
			drawWidth = width;
			drawHeight = width / imageRatio;
		} else {
			drawHeight = height;
			drawWidth = height * imageRatio;
		}

		const drawX = x + (width - drawWidth) / 2;
		const drawY = y + (height - drawHeight) / 2;

		context.drawImage(image, drawX, drawY, drawWidth, drawHeight);
		context.restore();
	}
	
    onMount(() => {
		logo = new Image();
		logo.src = `${base}/images/logo-sk-jugend-symbol-mittel.png`; 
	});
</script>



<div class="min-h-screen bg-white p-4 print:hidden">
	<div class="mx-auto w-full max-w-5xl">
		<div class="flex justify-center">
			<div class="overflow-hidden rounded-2xl bg-black shadow-lg" style="max-width: 100%;">

				<!-- KAMERA -->
				<div
					class="relative overflow-hidden"
					style={`height: min(70vh, 700px); aspect-ratio: ${TARGET_ASPECT_RATIO}; width: auto; max-width: 95vw;`}
				>
				{#if photoUrl}
					<img
						src={photoUrl}
						alt="Aufgenommenes Foto"
						class="h-full w-full object-cover"
					/>
				{:else}
					<video
						bind:this={videoRef}
						autoplay
						playsinline
						muted
						class="h-full w-full object-cover"
					></video>
				{/if}
				<!-- FOOTER -->
				<div class="absolute bottom-0 left-0 flex w-full items-center justify-center bg-center bg-no-repeat text-black" 
				style={`
					height: 15%;
					background-image: url('${currentLayout().footer}');
					background-size: contain;
					background-color: black;
					color: ${currentLayout().textColor};
				`}>
					<input
					type="text"
					bind:value={overlayText}
					placeholder="Text eingeben"
					oninput={renderFinalPhoto}
					class="w-full bg-transparent text-center text-black outline-none"/>
				</div>
			</div>
		</div>
		</div>

		<!-- BUTTONS -->
		<div class="mt-3 flex justify-center gap-3 text-sm font-medium">
			<button class="rounded-2xl bg-green-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={startCamera}>
				Kamera starten
			</button>

			<button class="rounded-2xl bg-red-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={stopCamera}>
				Kamera stoppen
			</button> 

			<button class="rounded-2xl bg-gray-300 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={switchCamera}>
				Kamera wechseln
			</button>

			<button class="rounded-2xl bg-lime-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={takePhoto}>
				Foto aufnehmen
			</button>

			<button class="rounded-2xl bg-red-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={removePhoto}>
				Foto entfernen
			</button>

			<button class="rounded-2xl bg-indigo-300 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={changeLayout}>
				Layout wechseln
			</button>

			<button class="rounded-2xl bg-yellow-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={sharePhoto}>
				Foto teilen
			</button>

			<button class="rounded-2xl bg-blue-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={printPhoto}>
				Foto Drucken
			</button>
		</div>
	</div>
</div>

<canvas bind:this={canvasRef} class="hidden"></canvas>

	<!-- PRINT ONLY -->
<div class="print-only">
	{#if photoUrl}
		<img src={photoUrl} alt="Print Photo" />
	{/if}
</div>

<style>
	.print-only {
		display: none;
	}

	@page {
	size: A4 landscape;
	margin: 0;
	}

	@media print {
		html,
		body {
			margin: 0;
			padding: 0;
			overflow: hidden;
		}

		.print-only {
			display: flex;
			width: 297mm;
			height: 210mm;
			align-items: center;
			justify-content: center;
			background: white;
			overflow: hidden;
		}

		.print-only img {
			display: block;
			width: 100%;
			height: 100%;
			object-fit: contain;
		}
	}

</style>