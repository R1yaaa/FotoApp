<script lang="ts">
    import { onMount } from 'svelte';
    let videoRef = $state<any>();
    let canvasRef = $state<any>();
    let photoUrl = $state<string | null>(null);
	let stream = $state<MediaStream>();
	let facingMode = $state('user');
	let overlayText = $state('');
	let layoutIndex = $state(0);
	const layouts = ['bar-bottom', 'bar-top', 'no-bar'];
	const currentLayout = layouts[layoutIndex];

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
		if (!videoRef || !canvasRef) {
			console.warn("Video or canvas is not ready.");
			return;
		}

        const canvas = canvasRef;
        const video = videoRef;
        const context = canvas.getContext('2d');

		if (!context) {
			console.error("Could not get canvas context.");
			return;
		}

        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;

        context.drawImage(video, 0, 0, canvas.width, canvas.height);

		const barHeight = canvas.height * 0.15;
		context.fillStyle = '#fef3c7';
		context.fillRect(0, canvas.height - barHeight, canvas.width, barHeight);
		context.fillStyle = 'black';
		context.font = `${canvas.width * 0.04}px Arial`;
		context.textAlign = 'center';
		context.textBaseline = 'middle';
		context.fillText(overlayText, canvas.width / 2, canvas.height - barHeight / 2);

        photoUrl = canvas.toDataURL('image/jpeg');
    }

    function removePhoto(){
        photoUrl = null;
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
	}
	
    
</script>



<div class="min-h-screen bg-white p-4 print:hidden">
	<div class="mx-auto w-full max-w-5xl">
		<div class="overflow-hidden rounded-2xl bg-black shadow-lg">

			<!-- KAMERA -->
			<div class="relative aspect-video overflow-hidden">
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
				<!-- BALKEN -->
				<div class="absolute bottom-0 left-0 flex h-16 w-full items-center justify-center bg-yellow-100 text-black">
					<input
					type="text"
					bind:value={overlayText}
					placeholder="Text eingeben"
					class="w-full bg-transparent text-center text-black outline-none"/>
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
				Foto speichern
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
			max-width: 100vw;
			max-height: 100vh;
			align-items: center;
			justify-content: center;
			background: white;
			overflow: hidden;
		}

		.print-only img {
			display: block;
			width: auto;
			height: 80vh;
			object-fit: contain;
		}
	}

</style>

