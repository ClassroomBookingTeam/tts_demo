<!-- App.svelte -->
<script lang="ts">
	import { onMount } from 'svelte';
	import { writable } from 'svelte/store';
	let mediaRecorder: any;
	let audioChunks: any = [];
	let recording = false;
	let audioBlob: any;

	let question_str = 'Record and hit QA Text!';
	let answer_str = 'Record and hit QA Text!';
	let audioSrc = writable('');

	onMount(() => {
		navigator.mediaDevices.getUserMedia({ audio: true }).then((stream) => {
			mediaRecorder = new MediaRecorder(stream);
			mediaRecorder.addEventListener('dataavailable', (event: any) => {
				audioChunks.push(event.data);
			});
		});
	});

	function startRecording() {
		audioChunks = [];
		mediaRecorder.start();
		recording = true;
	}

	function stopRecording() {
		mediaRecorder.stop();
		recording = false;
		mediaRecorder.addEventListener('stop', () => {
			audioBlob = new Blob(audioChunks, { type: 'audio/mp3' });
		});
	}

	async function qaText() {
		const formData = new FormData();
		formData.append('file', audioBlob, 'recording.mp3');

		const response = await fetch('http://localhost:4011/qa_text', {
			method: 'POST',
			body: formData
		});
		const data = await response.json();
		console.log(data);
		question_str = data['question'];
		answer_str = data['answer'];
	}

	async function qaVoice() {
		const formData = new FormData();
		formData.append('file', audioBlob, 'recording.mp3');

		await fetch('http://localhost:4011/qa_voice', {
			method: 'POST',
			body: formData
		})
			.then((response) => response.blob())
			.then((blob) => {
				const url = URL.createObjectURL(blob);
				audioSrc.set(url);
			})
			.catch((error) => console.error('Error fetching audio:', error));
	}
</script>

<main>
	{#if !recording}
		<button on:click={startRecording}>Record</button>
	{:else}
		<button on:click={stopRecording}>Stop</button>
	{/if}
	<button on:click={qaText} disabled={!audioBlob}>QA Text</button>
	<button on:click={qaVoice} disabled={!audioBlob}>QA Voice</button>

	<p><b>Question:</b> {question_str}</p>
	<p><b>Answer: </b> {answer_str}</p>

	{#if $audioSrc}
		<audio src={$audioSrc} controls> Your browser does not support the audio element. </audio>
	{:else}
		<p>Say something</p>
	{/if}
</main>

<style>
	button {
		margin: 5px;
		padding: 8px 16px;
		font-size: 16px;
	}
</style>

