<script lang="ts">
    import { onMount } from 'svelte';
    let videoRef = $state<any>();
    let canvasRef = $state<any>();
    let photoUrl = $state<string | null>(null);
	let stream = $state<MediaStream>();
	let facingMode = $state('user');

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
		context.fillText('LOGO / TEXT', canvas.width / 2, canvas.height - barHeight / 2);

        photoUrl = canvas.toDataURL('image/png');
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
		link.download = 'gruppenfoto.png';
		link.click();
	}

	function printPhoto() {
		if (!photoUrl) {
			console.warn("No photo available to print.");
			return;
		}
		window.print();
	}

    onMount(() => {
	startCamera();
    });
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
					LOGO / TEXT
				</div>
			</div>
		</div>

		<!-- BUTTONS -->
		<div class="mt-3 flex justify-center gap-3 text-sm font-medium">
			<button class="rounded-2xl bg-gray-300 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={switchCamera}>
				Kamera wechseln
			</button>

			<button class="rounded-2xl bg-green-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={takePhoto}>
				Foto aufnehmen
			</button>

			<button class="rounded-2xl bg-red-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={removePhoto}>
				Foto entfernen
			</button>

			<button class="rounded-2xl bg-yellow-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={savePhoto}>
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
		body {
			margin: 0;
		}

		.print-only {
			display: block;
			max-width: 95vw;
			max-height: 95vh;
			align-items: center;
			justify-content: center;
			background: white;
			object-fit: contain;
		}

		.print-only img {
			width: 95%;
			height: auto;
			object-fit: contain;
		}
	}
</style>

