<script lang="ts">
	import { onMount } from 'svelte';
	import logo from '$lib/assets/logo.png';

	// Type definitions
	interface MusicTrack {
		minBPM: number;
		maxBPM: number;
		name: string;
		artist: string;
		audioFile: string; // Path to audio file
		originalBPM: number; // Original BPM of the song for pitch adjustment
	}

	interface DeviceMotionEventWithPermission extends DeviceMotionEvent {
		requestPermission?: () => Promise<'granted' | 'denied'>;
	}

	interface DeviceMotionEventConstructorWithPermission {
		new (type: string, eventInitDict?: DeviceMotionEventInit): DeviceMotionEvent;
		prototype: DeviceMotionEvent;
		requestPermission?: () => Promise<'granted' | 'denied'>;
	}

	// State variables
	let permissionGranted: boolean = false;
	let isMonitoring: boolean = false;
	let steps: number = 0;
	let lastStepTime: number = 0;
	let isPlaying: boolean = false;
	let currentBPM: number = 0;
	let speed: number = 0.0;
	let permissionStatus: string = 'Waiting for permission';
	let isDemoMode: boolean = false;
	let currentAudio: HTMLAudioElement | null = null;
	let audioContext: AudioContext | null = null;
	let audioSource: AudioBufferSourceNode | null = null;
	let gainNode: GainNode | null = null;
	let isLoading: boolean = false;

	// Music tracks by BPM range - Update these paths to match your downloaded songs
	const musicTracks: MusicTrack[] = [
		{
			minBPM: 0,
			maxBPM: 90,
			name: 'Relaxing Ambient',
			artist: 'Soundscape',
			audioFile: '/songs/relaxing-ambient.mp3',
			originalBPM: 75
		},
		{
			minBPM: 90,
			maxBPM: 110,
			name: 'Chill Vibes',
			artist: 'Lo-Fi Beats',
			audioFile: '/songs/chill-vibes.mp3',
			originalBPM: 95
		},
		{
			minBPM: 110,
			maxBPM: 130,
			name: 'Walking Rhythm',
			artist: 'Indie Pop',
			audioFile: '/songs/walking-rhythm.mp3',
			originalBPM: 120
		},
		{
			minBPM: 130,
			maxBPM: 150,
			name: 'Energy Boost',
			artist: 'Pop Hits',
			audioFile: '/songs/energy-boost.mp3',
			originalBPM: 140
		},
		{
			minBPM: 150,
			maxBPM: 200,
			name: 'Power Walk',
			artist: 'Electronic Dance',
			audioFile: '/songs/power-walk.mp3',
			originalBPM: 175
		}
	];

	// Initialize Web Audio API
	async function initializeAudioContext(): Promise<void> {
		try {
			audioContext = new (window.AudioContext || (window as any).webkitAudioContext)();
			gainNode = audioContext.createGain();
			gainNode.connect(audioContext.destination);
		} catch (error) {
			console.error('Error initializing audio context:', error);
		}
	}

	// Load and play audio file
	async function loadAndPlayAudio(track: MusicTrack): Promise<void> {
		if (!audioContext || !gainNode) {
			await initializeAudioContext();
		}

		try {
			isLoading = true;

			// Stop current audio if playing
			if (audioSource) {
				audioSource.stop();
				audioSource.disconnect();
			}

			// Fetch audio file
			const response = await fetch(track.audioFile);
			const arrayBuffer = await response.arrayBuffer();
			const audioBuffer = await audioContext!.decodeAudioData(arrayBuffer);

			// Create new audio source
			audioSource = audioContext!.createBufferSource();
			audioSource.buffer = audioBuffer;
			audioSource.connect(gainNode!);
			audioSource.loop = true; // Loop the song

			// Calculate playback rate to match current BPM
			const playbackRate = currentBPM / track.originalBPM;
			audioSource.playbackRate.setValueAtTime(
				Math.max(0.5, Math.min(2.0, playbackRate)), // Limit between 0.5x and 2.0x speed
				audioContext!.currentTime
			);

			// Start playing
			audioSource.start(0);
			isLoading = false;
		} catch (error) {
			console.error('Error loading audio:', error);
			isLoading = false;
			// Fallback to simple HTML5 audio
			playWithHTMLAudio(track);
		}
	}

	// Fallback HTML5 audio player (without pitch adjustment)
	function playWithHTMLAudio(track: MusicTrack): void {
		try {
			if (currentAudio) {
				currentAudio.pause();
				currentAudio.currentTime = 0;
			}

			currentAudio = new Audio(track.audioFile);
			currentAudio.loop = true;
			currentAudio.volume = 0.7;

			// Basic playback rate adjustment (may affect pitch)
			const playbackRate = currentBPM / track.originalBPM;
			currentAudio.playbackRate = Math.max(0.5, Math.min(2.0, playbackRate));

			currentAudio.play().catch((error) => {
				console.error('Error playing audio:', error);
			});
		} catch (error) {
			console.error('Error with HTML5 audio:', error);
		}
	}

	// Stop audio playback
	function stopAudio(): void {
		if (audioSource) {
			audioSource.stop();
			audioSource.disconnect();
			audioSource = null;
		}

		if (currentAudio) {
			currentAudio.pause();
			currentAudio.currentTime = 0;
			currentAudio = null;
		}
	}

	// Adjust playback speed based on current BPM
	function adjustPlaybackSpeed(newBPM: number): void {
		const currentTrack = getMatchingTrack(newBPM);
		const playbackRate = newBPM / currentTrack.originalBPM;
		const clampedRate = Math.max(0.5, Math.min(2.0, playbackRate));

		if (audioSource && audioContext) {
			// Web Audio API adjustment (preserves pitch)
			audioSource.playbackRate.setValueAtTime(clampedRate, audioContext.currentTime);
		} else if (currentAudio) {
			// HTML5 Audio adjustment (affects pitch)
			currentAudio.playbackRate = clampedRate;
		}
	}

	// Get music track matching current BPM
	function getMatchingTrack(bpm: number): MusicTrack {
		return musicTracks.find((track) => bpm >= track.minBPM && bpm < track.maxBPM) || musicTracks[0];
	}

	// Calculate walking speed and adjust music
	function calculateSpeed(): void {
		const stepLength: number = 0.7;
		if (steps > 1) {
			const timeElapsed: number = (Date.now() - lastStepTime) / 1000;
			const stepRate: number = 1 / timeElapsed;
			speed = stepRate * stepLength;
			const newBPM = Math.min(180, Math.max(60, stepRate * 60));

			// Only adjust if BPM changed significantly (to avoid constant adjustments)
			if (Math.abs(newBPM - currentBPM) > 2) {
				const previousTrack = getMatchingTrack(currentBPM);
				currentBPM = Math.round(newBPM);
				const newTrack = getMatchingTrack(currentBPM);

				// Switch songs if we moved to a different BPM range
				if (isPlaying && newTrack.audioFile !== previousTrack.audioFile) {
					loadAndPlayAudio(newTrack);
				} else if (isPlaying) {
					// Just adjust speed of current song
					adjustPlaybackSpeed(currentBPM);
				}
			}
		}
	}

	// Step detection algorithm
	function detectSteps(magnitude: number): void {
		const threshold: number = 10.5;
		if (magnitude > threshold) {
			const now: number = Date.now();
			if (now - lastStepTime > 300) {
				steps++;
				lastStepTime = now;
				calculateSpeed();
			}
		}
	}

	// Process motion data
	function handleMotionEvent(event: DeviceMotionEvent): void {
		if (!event.accelerationIncludingGravity) return;

		const acceleration = event.accelerationIncludingGravity;
		const magnitude: number = Math.sqrt(
			(acceleration.x || 0) * (acceleration.x || 0) +
				(acceleration.y || 0) * (acceleration.y || 0) +
				(acceleration.z || 0) * (acceleration.z || 0)
		);
		detectSteps(magnitude);
	}

	// Start monitoring motion
	function startMonitoring(): void {
		if (!permissionGranted || isMonitoring) return;

		isMonitoring = true;
		window.addEventListener('devicemotion', handleMotionEvent);

		// Start demo mode after 3 seconds if no real motion detected
		setTimeout(() => {
			if (!isDemoMode && speed === 0) {
				startDemoMode();
			}
		}, 3000);
	}

	// Demo mode for testing without motion sensors
	function startDemoMode(): void {
		isDemoMode = true;
		permissionStatus = 'Demo mode (simulated motion)';

		let demoValue: number = 0.5;
		let increasing: boolean = true;

		setInterval(() => {
			if (increasing) {
				demoValue += 0.05;
				if (demoValue > 1.8) increasing = false;
			} else {
				demoValue -= 0.05;
				if (demoValue < 0.5) increasing = true;
			}

			speed = demoValue;
			currentBPM = Math.round(80 + demoValue * 40);
		}, 1000);
	}

	// Request motion permissions
	async function requestPermission(): Promise<void> {
		try {
			const DeviceMotionEventTyped =
				DeviceMotionEvent as DeviceMotionEventConstructorWithPermission;

			if (
				typeof DeviceMotionEventTyped !== 'undefined' &&
				typeof DeviceMotionEventTyped.requestPermission === 'function'
			) {
				const permission = await DeviceMotionEventTyped.requestPermission();
				if (permission === 'granted') {
					permissionGranted = true;
					permissionStatus = 'Motion access granted';
					startMonitoring();
				} else {
					permissionStatus = 'Permission denied';
				}
			} else {
				// Handle non-iOS devices
				permissionGranted = true;
				permissionStatus = 'Motion access granted';
				startMonitoring();
			}
		} catch (error) {
			console.error('Error requesting permission:', error);
			permissionStatus = 'Error requesting permission';
		}
	}

	// Toggle music playback
	async function toggleMusic(): Promise<void> {
		if (isPlaying) {
			// Stop playing
			stopAudio();
			isPlaying = false;
		} else {
			// Start playing
			isPlaying = true;
			const track = getMatchingTrack(currentBPM);
			await loadAndPlayAudio(track);
		}
	}

	// Cleanup on component destroy
	onMount(() => {
		return () => {
			stopAudio();
			if (audioContext) {
				audioContext.close();
			}
		};
	});
	$: currentTrack = getMatchingTrack(currentBPM);
	$: permissionButtonClass =
		permissionGranted && !isDemoMode
			? 'bg-green-500 hover:bg-green-600'
			: isDemoMode
				? 'bg-blue-500 hover:bg-blue-600'
				: 'bg-orange-400 hover:bg-orange-500';
	$: permissionButtonText =
		permissionGranted && !isDemoMode
			? 'Sensor Active'
			: isDemoMode
				? 'Demo Active'
				: 'Enable Motion Sensor';
</script>

<svelte:head>
	<title>PulsePlay - Walk to your rhythm</title>
	<meta name="description" content="Adaptive music player that matches your walking pace" />
	<link
		rel="stylesheet"
		href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css"
	/>
</svelte:head>

<div
	class="flex min-h-screen items-center justify-center bg-gradient-to-br from-slate-800 via-slate-700 to-slate-900 p-4"
>
	<div
		class="w-full max-w-md rounded-2xl border border-slate-600/50 bg-slate-700/90 p-6 shadow-2xl backdrop-blur-sm"
	>
		<!-- Logo and Header -->
		<div class="mb-8 text-center">
			<div class="mb-4 inline-flex h-20 w-20 items-center justify-center rounded-full shadow-lg">
				<img src={logo} alt="logo" class="h-full w-full rounded-full object-contain" />
			</div>
			<h1 class="mb-2 text-3xl font-bold text-orange-400">PulsePlay</h1>
			<p class="text-sm text-slate-300">Walk to your rhythm</p>
		</div>

		<!-- Permission Section -->
		<div class="mb-6 rounded-xl border border-slate-500/30 bg-slate-600/50 p-5">
			<div class="mb-4 flex items-center gap-3">
				<i class="fas fa-mobile-alt text-orange-400"></i>
				<p class="text-sm text-slate-200">Enable motion access to detect your walking speed</p>
			</div>
			<button
				on:click={requestPermission}
				disabled={permissionGranted}
				class="w-full transform rounded-full px-4 py-3 font-semibold text-slate-800 shadow-lg transition-all duration-200 hover:scale-105 active:scale-95 disabled:cursor-not-allowed disabled:opacity-75 {permissionButtonClass}"
			>
				{permissionButtonText}
			</button>
			<p class="mt-3 text-center text-sm text-orange-400">{permissionStatus}</p>
		</div>

		<!-- Speed Display -->
		<div class="mb-6 rounded-xl border border-slate-500/30 bg-slate-600/50 p-6 text-center">
			<div class="mb-2 flex items-center justify-center gap-2">
				<i class="fas fa-walking text-orange-400"></i>
				<span class="text-sm font-medium text-slate-300">Current Speed</span>
			</div>
			<div class="mb-1 text-4xl font-bold text-orange-400">
				{speed.toFixed(2)}
			</div>
			<div class="text-sm text-slate-400">meters per second</div>
		</div>

		<!-- Music Player -->
		<div class="rounded-xl border border-slate-500/30 bg-slate-600/50 p-6">
			<div class="mb-4 flex items-center gap-2">
				<i class="fas fa-music text-orange-400"></i>
				<h3 class="font-semibold text-orange-400">Music Player</h3>
			</div>

			<div class="mb-4 text-center">
				<button
					on:click={toggleMusic}
					disabled={isLoading}
					class="inline-flex h-16 w-16 transform items-center justify-center rounded-full bg-orange-400 text-slate-800 shadow-lg transition-all duration-200 hover:scale-110 hover:bg-orange-500 active:scale-95 disabled:cursor-not-allowed disabled:opacity-50"
				>
					{#if isLoading}
						<i class="fas fa-spinner fa-spin text-2xl"></i>
					{:else if isPlaying}
						<i class="fas fa-pause text-2xl"></i>
					{:else}
						<i class="fas fa-play ml-1 text-2xl"></i>
					{/if}
				</button>
			</div>

			<div class="text-center">
				<div class="mb-1 text-sm text-slate-200">
					{isLoading ? 'Loading...' : isPlaying ? `Playing: ${currentTrack.name}` : 'Ready to play'}
				</div>
				{#if isPlaying && !isLoading}
					<div class="mb-3 text-xs text-slate-400">
						by {currentTrack.artist}
					</div>
					<div class="mb-3 text-xs text-slate-500">
						Speed: {currentBPM > 0 ? (currentBPM / currentTrack.originalBPM).toFixed(2) : '1.00'}x
					</div>
				{/if}
				<div class="inline-flex items-center gap-2 rounded-full bg-slate-700/70 px-3 py-1">
					<div class="h-2 w-2 rounded-full bg-orange-400 {isPlaying ? 'animate-pulse' : ''}"></div>
					<span class="text-sm font-medium text-orange-400">
						{Math.round(currentBPM)} BPM
					</span>
				</div>
			</div>
		</div>

		<!-- Instructions -->
		<p class="mt-6 text-center text-xs leading-relaxed text-slate-400">
			Hold your phone securely while walking. Music will automatically adapt to your pace for the
			perfect workout rhythm.
		</p>
	</div>
</div>

<style>
	/* Custom animations for elements not covered by Tailwind */
	@keyframes pulse {
		0%,
		100% {
			opacity: 1;
		}
		50% {
			opacity: 0.5;
		}
	}

	.animate-pulse {
		animation: pulse 2s cubic-bezier(0.4, 0, 0.6, 1) infinite;
	}
</style>
