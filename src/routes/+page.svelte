<script lang="ts">
	import { onMount } from 'svelte';

	let worker;
	let audioSource = $state(null);
	let voices: { id: string; name: string }[] = $state([]);
	let inputText = $state('');
	let isLoading = $state(false);
	let modelLoaded = $state(false);
	let selectedVoice = $state('af_heart');
	let audioElement: null | HTMLAudioElement = $state(null);

	onMount(async () => {
		worker = new Worker(new URL('../lib/worker.js', import.meta.url), { type: 'module' });

		worker.addEventListener('message', onMessageReceived);
		worker.addEventListener('error', onErrorReceived);
	});

	function onMessageReceived(event) {
		switch (event.data.status) {
			case 'device':
				console.log('Loading model');
				break;
			case 'ready':
				// Voices is actually an object, but we want an array
				voices = Object.entries(event.data.voices).map(([id, name]) => ({
					id,
					name
				}));

				// Only include a few good voices
				const goodVoices = ['af_heart', 'af_bella', 'bf_emma', 'am_michael'];
				voices = voices.filter((voice) => goodVoices.includes(voice.id));

				modelLoaded = true;
				break;
			case 'error':
				console.error('Error from worker:', event.data);
				break;
			case 'complete':
				console.log('Audio generated');
				const { audio, text } = event.data;
				audioSource = audio;
				isLoading = false;

				// Play audio once generated
				setTimeout(() => {
					if (audioElement) {
						audioElement.play().catch((err) => console.error('Failed to autoplay audio:', err));
					}
				}, 100);
				break;
		}
	}

	function onErrorReceived(event) {
		console.error('Error from worker:', event);
		isLoading = false;
	}

	function handleSubmit(event) {
		event.preventDefault();
		isLoading = true;

		worker.postMessage({
			type: 'generate',
			text: inputText,
			voice: selectedVoice
		});
	}

	let canSubmit = $derived(inputText.trim().length > 0 && !isLoading && modelLoaded);
</script>

<main class="mx-auto max-w-3xl p-8">
	<h1 class="mb-2 text-center text-4xl text-gray-800">Text-to-Speech</h1>
	<p class="text-center text-gray-600">
		Transform text into natural-sounding speech, right in your browser!
	</p>
	<p class="mb-8 text-center text-gray-600">Only Chrome is currently supported.</p>

	{#if !modelLoaded}
		<div class="flex flex-col items-center justify-center p-8">
			<div class="mb-4 flex items-center space-x-3">
				<div
					class="h-5 w-5 animate-spin rounded-full border-2 border-gray-300 border-t-blue-500"
				></div>
				<p class="text-lg font-medium text-gray-600">Loading speech model...</p>
			</div>
			<p class="text-center text-gray-500">This might take a moment</p>
		</div>
	{:else}
		<form onsubmit={handleSubmit} class="flex flex-col gap-6 rounded-lg bg-white p-8 shadow-sm">
			<div class="flex flex-col gap-2">
				<label for="text-input" class="font-medium text-gray-600">Enter your text</label>
				<textarea
					id="text-input"
					bind:value={inputText}
					placeholder="Type something you'd like to hear..."
					rows="4"
					class="w-full rounded-md border border-gray-200 p-3 text-base focus:border-blue-500 focus:ring-2 focus:ring-blue-500/20 focus:outline-none disabled:bg-gray-100"
				></textarea>
			</div>

			<div class="flex flex-col gap-2">
				<label for="voice-select" class="font-medium text-gray-600">Choose a voice</label>
				<select
					id="voice-select"
					bind:value={selectedVoice}
					class="w-full rounded-md border border-gray-200 p-3 text-base focus:border-blue-500 focus:ring-2 focus:ring-blue-500/20 focus:outline-none"
				>
					{#each voices as voice}
						<option value={voice.id}
							>{voice.name.name} ({voice.name.language === 'en-us' ? 'American' : 'British'}) - {voice
								.name.gender}</option
						>
					{/each}
				</select>
			</div>

			<button
				type="submit"
				disabled={!canSubmit}
				class="rounded-md bg-blue-500 px-6 py-3 text-base font-medium text-white transition-colors hover:bg-blue-600 disabled:cursor-not-allowed disabled:bg-gray-400"
			>
				{isLoading ? 'Generating...' : 'Generate Speech'}
			</button>
		</form>

		{#if audioSource}
			<div class="mt-8">
				<audio bind:this={audioElement} controls src={audioSource} class="w-full"></audio>
			</div>
		{/if}
	{/if}
</main>
<footer class="mx-auto mt-4 max-w-3xl p-8 text-center text-gray-600">
	<p class="mb-2 flex items-center justify-center gap-2">
		made by <a href="https://mikeesto.com" class="underline transition-colors hover:text-blue-500"
			>@mikeesto</a
		>
		<span>⚉</span>
		<a
			href="https://github.com/mikeesto/kokoro-web"
			class="underline transition-colors hover:text-blue-500">source code</a
		>
	</p>
</footer>
