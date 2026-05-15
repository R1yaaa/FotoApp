<script lang="ts">
    import { onMount } from 'svelte';
    let videoRef = $state<any>();
    let canvasRef = $state<any>();
    let photoUrl = $state<string | null>(null);

    async function startCamera() {
        const stream = await navigator.mediaDevices.getUserMedia({
            video: {
            facingMode: 'user'
            }
        });

        videoRef.srcObject = stream;
    }

    function takePhoto() {
        const canvas = canvasRef;
        const video = videoRef;
        const context = canvas.getContext('2d');

        canvas.width = video.videoWidth;
        canvas.height = video.videoHeight;

        context.drawImage(video, 0, 0, canvas.width, canvas.height);

        photoUrl = canvas.toDataURL('image/png');
    }

    function removePhoto(){
        photoUrl = null;
        setTimeout(() => {
            startCamera();
        });
    }


    onMount(() => {
	startCamera();
    });
</script>



<div class="min-h-screen bg-white p-4">
	<div class="mx-auto w-full max-w-5xl">
		<div class="overflow-hidden rounded-2xl bg-black shadow-lg">

			<!-- KAMERA -->
			<div class="aspect-video overflow-hidden">
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
			</div>

			<!-- BALKEN -->
			<div class="flex h-16 items-center justify-center bg-yellow-100 text-black">
				LOGO / TEXT
			</div>
		</div>

		<!-- BUTTONS -->
		<div class="mt-3 flex justify-center gap-3 text-sm font-medium">
			<button class="rounded-2xl bg-gray-300 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer">
				Kamera wechseln
			</button>

			<button class="rounded-2xl bg-green-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={takePhoto}>
				Foto aufnehmen
			</button>

			<button class="rounded-2xl bg-red-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer" onclick={removePhoto}>
				Foto entfernen
			</button>

			<button class="rounded-2xl bg-blue-400 px-4 py-2 transition-all duration-200 hover:scale-105 cursor-pointer">
				Drucken
			</button>
		</div>
	</div>

	<canvas bind:this={canvasRef} class="hidden"></canvas>
</div>

