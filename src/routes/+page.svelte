<script lang="ts">
    import { onMount } from 'svelte';
    import logo from "$lib/assets/logo.png";

    // Type definitions
    interface MusicTrack {
        minBPM: number;
        maxBPM: number;
        name: string;
        artist: string;
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
    let speed: number = 0.00;
    let permissionStatus: string = 'Waiting for permission';
    let isDemoMode: boolean = false;

    // Music tracks by BPM range
    const musicTracks: MusicTrack[] = [
        { minBPM: 0, maxBPM: 90, name: "Relaxing Ambient", artist: "Soundscape" },
        { minBPM: 90, maxBPM: 110, name: "Chill Vibes", artist: "Lo-Fi Beats" },
        { minBPM: 110, maxBPM: 130, name: "Walking Rhythm", artist: "Indie Pop" },
        { minBPM: 130, maxBPM: 150, name: "Energy Boost", artist: "Pop Hits" },
        { minBPM: 150, maxBPM: 200, name: "Power Walk", artist: "Electronic Dance" }
    ];

    // Get music track matching current BPM
    function getMatchingTrack(bpm: number): MusicTrack {
        return musicTracks.find(track => bpm >= track.minBPM && bpm < track.maxBPM) || musicTracks[0];
    }

    // Calculate walking speed
    function calculateSpeed(): void {
        const stepLength: number = 0.7;
        if (steps > 1) {
            const timeElapsed: number = (Date.now() - lastStepTime) / 1000;
            const stepRate: number = 1 / timeElapsed;
            speed = stepRate * stepLength;
            currentBPM = Math.min(180, Math.max(60, stepRate * 60));
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
            currentBPM = Math.round(80 + (demoValue * 40));
        }, 1000);
    }

    // Request motion permissions
    async function requestPermission(): Promise<void> {
        try {
            const DeviceMotionEventTyped = DeviceMotionEvent as DeviceMotionEventConstructorWithPermission;

            if (typeof DeviceMotionEventTyped !== 'undefined' && typeof DeviceMotionEventTyped.requestPermission === 'function') {
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
    function toggleMusic(): void {
        isPlaying = !isPlaying;
    }

    // Reactive statements
    $: currentTrack = getMatchingTrack(currentBPM);
    $: permissionButtonClass = permissionGranted && !isDemoMode ?
        'bg-green-500 hover:bg-green-600' :
        isDemoMode ? 'bg-blue-500 hover:bg-blue-600' :
        'bg-orange-400 hover:bg-orange-500';
    $: permissionButtonText = permissionGranted && !isDemoMode ?
        'Sensor Active' :
        isDemoMode ? 'Demo Active' :
        'Enable Motion Sensor';
</script>

<svelte:head>
    <title>PulsePlay - Walk to your rhythm</title>
    <meta name="description" content="Adaptive music player that matches your walking pace" />
    <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
</svelte:head>

<div class="min-h-screen bg-gradient-to-br from-slate-800 via-slate-700 to-slate-900 flex items-center justify-center p-4">
    <div class="w-full max-w-md bg-slate-700/90 backdrop-blur-sm rounded-2xl p-6 shadow-2xl border border-slate-600/50">

        <!-- Logo and Header -->
        <div class="text-center mb-8">
            <div class="inline-flex items-center justify-center w-20 h-20rounded-full mb-4 shadow-lg">
                <img src="{logo}" alt="logo" class="w-full h-full rounded-full object-contain">
            </div>
            <h1 class="text-3xl font-bold text-orange-400 mb-2">PulsePlay</h1>
            <p class="text-slate-300 text-sm">Walk to your rhythm</p>
        </div>

        <!-- Permission Section -->
        <div class="bg-slate-600/50 rounded-xl p-5 mb-6 border border-slate-500/30">
            <div class="flex items-center gap-3 mb-4">
                <i class="fas fa-mobile-alt text-orange-400"></i>
                <p class="text-slate-200 text-sm">Enable motion access to detect your walking speed</p>
            </div>
            <button
                on:click={requestPermission}
                disabled={permissionGranted}
                class="w-full py-3 px-4 rounded-full font-semibold text-slate-800 transition-all duration-200 transform hover:scale-105 active:scale-95 shadow-lg disabled:opacity-75 disabled:cursor-not-allowed {permissionButtonClass}"
            >
                {permissionButtonText}
            </button>
            <p class="text-orange-400 text-sm mt-3 text-center">{permissionStatus}</p>
        </div>

        <!-- Speed Display -->
        <div class="bg-slate-600/50 rounded-xl p-6 mb-6 text-center border border-slate-500/30">
            <div class="flex items-center justify-center gap-2 mb-2">
                <i class="fas fa-walking text-orange-400"></i>
                <span class="text-slate-300 text-sm font-medium">Current Speed</span>
            </div>
            <div class="text-4xl font-bold text-orange-400 mb-1">
                {speed.toFixed(2)}
            </div>
            <div class="text-slate-400 text-sm">meters per second</div>
        </div>

        <!-- Music Player -->
        <div class="bg-slate-600/50 rounded-xl p-6 border border-slate-500/30">
            <div class="flex items-center gap-2 mb-4">
                <i class="fas fa-music text-orange-400"></i>
                <h3 class="text-orange-400 font-semibold">Music Player</h3>
            </div>

            <div class="text-center mb-4">
                <button
                    on:click={toggleMusic}
                    class="inline-flex items-center justify-center w-16 h-16 bg-orange-400 hover:bg-orange-500 rounded-full text-slate-800 transition-all duration-200 transform hover:scale-110 active:scale-95 shadow-lg"
                >
                    {#if isPlaying}
                        <i class="fas fa-pause text-2xl"></i>
                    {:else}
                        <i class="fas fa-play text-2xl ml-1"></i>
                    {/if}
                </button>
            </div>

            <div class="text-center">
                <div class="text-slate-200 text-sm mb-1">
                    {isPlaying ? `Playing: ${currentTrack.name}` : 'Ready to play'}
                </div>
                {#if isPlaying}
                    <div class="text-slate-400 text-xs mb-3">
                        by {currentTrack.artist}
                    </div>
                {/if}
                <div class="inline-flex items-center gap-2 bg-slate-700/70 rounded-full px-3 py-1">
                    <div class="w-2 h-2 bg-orange-400 rounded-full animate-pulse"></div>
                    <span class="text-orange-400 text-sm font-medium">
                        {Math.round(currentBPM)} BPM
                    </span>
                </div>
            </div>
        </div>

        <!-- Instructions -->
        <p class="text-slate-400 text-xs text-center mt-6 leading-relaxed">
            Hold your phone securely while walking. Music will automatically adapt to your pace for the perfect workout rhythm.
        </p>
    </div>
</div>

<style>
    /* Custom animations for elements not covered by Tailwind */
    @keyframes pulse {
        0%, 100% {
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
