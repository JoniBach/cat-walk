<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import { GLTFLoader } from 'three/examples/jsm/loaders/GLTFLoader.js';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

	// Configuration object for all static values
	const CONFIG = {
		// Scene settings
		scene: {
			backgroundColor: 0x222222
		},

		// Camera settings
		camera: {
			fov: 75,
			near: 0.1,
			far: 1000,
			position: { x: 0, y: 2, z: 5 },
			lookAt: { x: 0, y: 1, z: 0 }
		},

		// Renderer settings
		renderer: {
			antialias: true,
			shadowMapType: THREE.VSMShadowMap // Variance Shadow Mapping for ultra soft shadows
		},

		// Lighting settings
		lighting: {
			ambient: {
				color: 0x404040,
				intensity: 0.6
			},
			directional: {
				color: 0xffffff,
				intensity: 0.8,
				position: { x: 10, y: 10, z: 5 },
				shadowMapSize: 4096, // Good balance for VSM
				shadowRadius: 10, // Much higher radius for very soft shadows
				shadowBias: 0, // VSM doesn't need bias
				shadowBlurSamples: 0, // VSM handles blur internally
				shadowCamera: {
					near: 0.1,
					far: 100,
					left: -25,
					right: 25,
					top: 25,
					bottom: -25
				}
			}
		},

		// Ground settings
		ground: {
			size: 20,
			color: 0x555555 // Match scene background
		},

		// Model settings
		model: {
			scale: 7 // Default scale multiplier
		},

		// Controls settings
		controls: {
			enableDamping: true,
			enableZoom: true,
			enableRotate: true,
			enablePan: false,
			dampingFactor: 0.05,
			minDistance: 2,
			maxDistance: 10,
			minPolarAngle: 0,
			maxPolarAngle: Math.PI / 2,
			minAzimuthAngle: -Infinity,
			maxAzimuthAngle: Infinity,
			screenSpacePanning: false
		},

		// Animation settings
		animation: {
			fps: 60,
			deltaTime: 0.016 // 1/60
		},

		// File paths
		paths: {
			gltfModel: '/gltf/lowpoly cat 5.glb'
		}
	} as const;

	// Type-safe gait state system
	type GaitState = 'stand' | 'walk' | 'trot' | 'gallop' | 'sit' | 'reach';

	// Graph-based transition system - defines only direct transitions
	const transitionGraph: Record<GaitState, Record<GaitState, string[]>> = {
		stand: {
			stand: ['stand'],
			walk: ['stand_to_walk', 'walk'],
			trot: ['stand_to_trot', 'trot'],
			gallop: ['stand_to_trot', 'trot_to_gallop', 'gallop'],
			sit: ['stand_to_sit', 'sit'],
			reach: [] // Auto-calculated: stand→sit→reach
		},
		walk: {
			stand: ['walk_to_stand', 'stand'],
			walk: ['walk'],
			trot: ['walk_to_trot', 'trot'],
			gallop: ['walk_to_trot', 'trot_to_gallop', 'gallop'],
			sit: [], // Auto-calculated: walk→stand→sit
			reach: [] // Auto-calculated: walk→stand→sit→reach
		},
		trot: {
			stand: ['trot_to_stand', 'stand'],
			walk: ['trot_to_walk', 'walk'],
			trot: ['trot'],
			gallop: ['trot_to_gallop', 'gallop'],
			sit: [], // Auto-calculated: trot→stand→sit
			reach: [] // Auto-calculated: trot→stand→sit→reach
		},
		gallop: {
			stand: ['gallop_to_trot', 'trot_to_stand', 'stand'],
			walk: ['gallop_to_trot', 'trot_to_walk', 'walk'],
			trot: ['gallop_to_trot', 'trot'],
			gallop: ['gallop'],
			sit: [], // Auto-calculated: gallop→stand→sit
			reach: [] // Auto-calculated: gallop→stand→sit→reach
		},
		sit: {
			stand: ['sit_to_stand', 'stand'],
			walk: ['sit_to_stand', 'stand_to_walk', 'walk'],
			trot: ['sit_to_stand', 'stand_to_trot', 'trot'],
			gallop: ['sit_to_stand', 'stand_to_trot', 'trot_to_gallop', 'gallop'],
			sit: ['sit'],
			reach: ['sit_to_reach', 'reach']
		},
		reach: {
			stand: ['reach_to_sit', 'sit_to_stand', 'stand'],
			walk: ['reach_to_sit', 'sit_to_stand', 'stand_to_walk', 'walk'],
			trot: ['reach_to_sit', 'sit_to_stand', 'stand_to_trot', 'trot'],
			gallop: ['reach_to_sit', 'sit_to_stand', 'stand_to_trot', 'trot_to_gallop', 'gallop'],
			sit: ['reach_to_sit', 'sit'],
			reach: ['reach']
		}
	};

	// BFS pathfinding algorithm for automatic transition routing
	function findTransitionPath(from: GaitState, to: GaitState): string[] {
		// Same state - return looping animation
		if (from === to) {
			return transitionGraph[from][to];
		}

		// Check for direct path first
		const directPath = transitionGraph[from][to];
		if (directPath && directPath.length > 0) {
			console.log(`Direct path ${from}→${to}:`, directPath);
			return directPath;
		}

		// BFS to find shortest path through intermediate states
		const queue: Array<{ state: GaitState; path: string[] }> = [{ state: from, path: [] }];
		const visited = new Set<GaitState>([from]);

		while (queue.length > 0) {
			const { state: currentState, path: currentPath } = queue.shift()!;

			// Check all possible next states from current state
			for (const nextState of Object.keys(transitionGraph[currentState]) as GaitState[]) {
				if (visited.has(nextState)) continue;

				const transition = transitionGraph[currentState][nextState];
				if (!transition || transition.length === 0) continue;

				// Build path by combining current path + this transition
				const newPath = [...currentPath];

				// Add transition animations, but skip intermediate loops
				for (let i = 0; i < transition.length; i++) {
					const anim = transition[i];
					// Skip intermediate loop animations (not the final target)
					if (nextState !== to && i === transition.length - 1 && anim === nextState) {
						continue; // Skip intermediate loop
					}
					newPath.push(anim);
				}

				// Found target state!
				if (nextState === to) {
					console.log(`BFS path ${from}→${to}:`, newPath);
					return newPath;
				}

				// Add to queue for further exploration
				visited.add(nextState);
				queue.push({ state: nextState, path: newPath });
			}
		}

		// No path found - fallback
		console.warn(`No transition path found from ${from} to ${to}`);
		return [];
	}

	let canvas: HTMLCanvasElement;
	let scene: THREE.Scene;
	let camera: THREE.PerspectiveCamera;
	let renderer: THREE.WebGLRenderer;
	let controls: OrbitControls;
	let mixer: THREE.AnimationMixer;
	let animations: THREE.AnimationClip[] = [];
	let currentAction: THREE.AnimationAction | null = null;
	let model: THREE.Group;

	// Animation states
	let animationNames: string[] = [];
	let selectedAnimation = '';
	let isPlaying = false;

	// New simplified state management
	let animationQueue: string[] = [];
	let currentAnimIndex = 0;
	let pendingTargetGait: GaitState | null = null;
	let currentGaitState: GaitState = 'stand';
	let isTransitioning = false;
	let waitForCycleEnd = false;

	// Model scale control
	let modelScale = CONFIG.model.scale;

	onMount(() => {
		initThreeJS();
		loadGLTF();
		animate();

		return () => {
			if (controls) {
				controls.dispose();
			}
			if (renderer) {
				renderer.dispose();
			}
		};
	});

	function initThreeJS() {
		// Scene
		scene = new THREE.Scene();
		scene.background = new THREE.Color(CONFIG.scene.backgroundColor);

		// Camera
		camera = new THREE.PerspectiveCamera(
			CONFIG.camera.fov,
			window.innerWidth / window.innerHeight,
			CONFIG.camera.near,
			CONFIG.camera.far
		);
		camera.position.set(
			CONFIG.camera.position.x,
			CONFIG.camera.position.y,
			CONFIG.camera.position.z
		);
		camera.lookAt(CONFIG.camera.lookAt.x, CONFIG.camera.lookAt.y, CONFIG.camera.lookAt.z);

		// Renderer
		renderer = new THREE.WebGLRenderer({
			canvas,
			antialias: CONFIG.renderer.antialias
		});
		renderer.setSize(window.innerWidth, window.innerHeight);
		renderer.shadowMap.enabled = true;
		renderer.shadowMap.type = CONFIG.renderer.shadowMapType;

		// Lights
		const ambientLight = new THREE.AmbientLight(
			CONFIG.lighting.ambient.color,
			CONFIG.lighting.ambient.intensity
		);
		scene.add(ambientLight);

		const directionalLight = new THREE.DirectionalLight(
			CONFIG.lighting.directional.color,
			CONFIG.lighting.directional.intensity
		);
		directionalLight.position.set(
			CONFIG.lighting.directional.position.x,
			CONFIG.lighting.directional.position.y,
			CONFIG.lighting.directional.position.z
		);
		directionalLight.castShadow = true;

		// Configure shadow properties for VSM ultra soft shadows
		directionalLight.shadow.mapSize.width = CONFIG.lighting.directional.shadowMapSize;
		directionalLight.shadow.mapSize.height = CONFIG.lighting.directional.shadowMapSize;
		directionalLight.shadow.radius = CONFIG.lighting.directional.shadowRadius;

		// VSM doesn't use bias, blurSamples, or normalBias
		// These settings are handled internally by VSM

		// Configure shadow camera for better coverage and softness
		directionalLight.shadow.camera.near = CONFIG.lighting.directional.shadowCamera.near;
		directionalLight.shadow.camera.far = CONFIG.lighting.directional.shadowCamera.far;
		directionalLight.shadow.camera.left = CONFIG.lighting.directional.shadowCamera.left;
		directionalLight.shadow.camera.right = CONFIG.lighting.directional.shadowCamera.right;
		directionalLight.shadow.camera.top = CONFIG.lighting.directional.shadowCamera.top;
		directionalLight.shadow.camera.bottom = CONFIG.lighting.directional.shadowCamera.bottom;

		directionalLight.shadow.camera.updateProjectionMatrix();
		renderer.shadowMap.needsUpdate = true; // Force shadow map regeneration

		// Debug logging for shadow settings
		console.log('Shadow settings applied:', {
			type: renderer.shadowMap.type === THREE.VSMShadowMap ? 'VSM' : 'PCF',
			mapSize: directionalLight.shadow.mapSize,
			radius: directionalLight.shadow.radius
		});

		scene.add(directionalLight);

		// Ground
		const groundGeometry = new THREE.PlaneGeometry(CONFIG.ground.size, CONFIG.ground.size);
		const groundMaterial = new THREE.MeshLambertMaterial({
			color: CONFIG.ground.color,
			transparent: false // Ensure no transparency interferes with shadows
		});
		const ground = new THREE.Mesh(groundGeometry, groundMaterial);
		ground.rotation.x = -Math.PI / 2;
		ground.receiveShadow = true;
		ground.name = 'ground'; // For debugging
		scene.add(ground);

		// Mouse controls
		controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = CONFIG.controls.enableDamping;
		controls.enableZoom = CONFIG.controls.enableZoom;
		controls.enableRotate = CONFIG.controls.enableRotate;
		controls.enablePan = CONFIG.controls.enablePan;
		controls.dampingFactor = CONFIG.controls.dampingFactor;
		controls.minDistance = CONFIG.controls.minDistance;
		controls.maxDistance = CONFIG.controls.maxDistance;
		controls.minPolarAngle = CONFIG.controls.minPolarAngle;
		controls.maxPolarAngle = CONFIG.controls.maxPolarAngle;
		controls.minAzimuthAngle = CONFIG.controls.minAzimuthAngle;
		controls.maxAzimuthAngle = CONFIG.controls.maxAzimuthAngle;
		controls.screenSpacePanning = CONFIG.controls.screenSpacePanning;

		// Handle window resize
		window.addEventListener('resize', onWindowResize);
	}

	function loadGLTF() {
		const loader = new GLTFLoader();
		loader.load(
			CONFIG.paths.gltfModel,
			(gltf) => {
				model = gltf.scene;
				model.scale.setScalar(CONFIG.model.scale);
				scene.add(model);

				// Enable shadows
				model.traverse((child) => {
					if (child instanceof THREE.Mesh) {
						child.castShadow = true;
						child.receiveShadow = true;
					}
				});

				// Setup animations
				if (gltf.animations && gltf.animations.length > 0) {
					mixer = new THREE.AnimationMixer(model);
					animations = gltf.animations;

					// Extract animation names
					animationNames = animations.map((clip) => clip.name);
					console.log('Available animations:', animationNames);

					// Set default animation to stand
					playAnimation('stand', true, 0); // No crossfade for initial animation
				}
			},
			(progress) => {
				console.log('Loading progress:', (progress.loaded / progress.total) * 100 + '%');
			},
			(error) => {
				console.error('Error loading GLTF:', error);
			}
		);
	}

	function playAnimation(
		animationName: string,
		loop: boolean = true,
		crossfadeDuration: number = 0.3
	) {
		if (!mixer || !animations) return;

		// Find the new animation
		const clip = animations.find((clip) => clip.name === animationName);
		if (!clip) return;

		// Create the new action
		const newAction = mixer.clipAction(clip);
		newAction.loop = loop ? THREE.LoopRepeat : THREE.LoopOnce;
		newAction.clampWhenFinished = !loop;

		// Handle crossfading for smooth transitions
		if (currentAction && crossfadeDuration > 0) {
			// Crossfade from current to new animation
			newAction.reset();
			newAction.play();
			newAction.crossFadeFrom(currentAction, crossfadeDuration, false);
		} else {
			// Stop current animation if no crossfade
			if (currentAction) {
				currentAction.stop();
			}
			newAction.reset();
			newAction.play();
		}

		// Update current action reference
		currentAction = newAction;
		isPlaying = true;
		selectedAnimation = animationName;
	}

	// New queue-based animation system
	function playNextInQueue() {
		// Check if queue is complete
		if (currentAnimIndex >= animationQueue.length) {
			// Transition complete - cleanup and check for pending
			isTransitioning = false;
			currentAnimIndex = 0;
			animationQueue = [];

			// Update current gait state
			currentGaitState = getCurrentGait();

			// Process pending transition if any
			if (pendingTargetGait) {
				const nextTarget = pendingTargetGait;
				pendingTargetGait = null;
				waitForCycleEnd = false;
				startTransitionToGait(nextTarget);
			}
			return;
		}

		// Play next animation in queue
		const animName = animationQueue[currentAnimIndex];
		const isLastInQueue = currentAnimIndex === animationQueue.length - 1;

		// Last animation should loop, others should not
		playAnimation(animName, isLastInQueue, 0.3);
		currentAnimIndex++;
	}

	function stopAnimation() {
		if (currentAction) {
			currentAction.stop();
			isPlaying = false;
		}
	}

	function getAnimationDuration(animationName: string): number {
		const clip = animations.find((clip) => clip.name === animationName);
		return clip ? clip.duration : 1.0;
	}

	function getCurrentGait(): GaitState {
		if (selectedAnimation.includes('gallop')) return 'gallop';
		if (selectedAnimation.includes('trot')) return 'trot';
		if (selectedAnimation.includes('walk')) return 'walk';
		if (selectedAnimation.includes('sit')) return 'sit';
		if (selectedAnimation.includes('reach')) return 'reach';
		if (selectedAnimation.includes('stand')) return 'stand';
		return currentGaitState; // fallback to tracked state
	}

	function playTransitionTo(targetGait: GaitState) {
		const currentGait = getCurrentGait();
		if (currentGait === targetGait && !isTransitioning) return; // already in target gait

		if (isTransitioning) {
			// Queue the transition request
			pendingTargetGait = targetGait;
			return;
		}

		// Check if we need to wait for current cycle to end
		const currentClip = animations.find((clip) => clip.name === selectedAnimation);
		if (currentClip && currentAction && currentAction.loop === THREE.LoopRepeat) {
			// Currently in a looping animation, wait for cycle end
			pendingTargetGait = targetGait;
			waitForCycleEnd = true;
			return;
		}

		// Start transition immediately
		startTransitionToGait(targetGait);
		waitForCycleEnd = false; // Reset flag
	}

	function startTransitionToGait(targetGait: GaitState) {
		const currentGait = getCurrentGait();
		const transitionPath = findTransitionPath(currentGait, targetGait);

		if (transitionPath.length === 0) {
			console.warn(`No transition path found from ${currentGait} to ${targetGait}`);
			return;
		}

		// Set up animation queue
		animationQueue = transitionPath;
		currentAnimIndex = 0;
		isTransitioning = true;

		// Start playing the queue
		playNextInQueue();
	}

	function updateModelScale() {
		if (model) {
			model.scale.setScalar(modelScale);
		}
	}

	function onWindowResize() {
		camera.aspect = window.innerWidth / window.innerHeight;
		camera.updateProjectionMatrix();
		renderer.setSize(window.innerWidth, window.innerHeight);
	}

	function animate() {
		requestAnimationFrame(animate);

		// Update controls
		if (controls) {
			controls.update();
		}

		// Update animation mixer
		if (mixer) {
			mixer.update(CONFIG.animation.deltaTime);

			// Check for cycle end to trigger queued transitions
			if (waitForCycleEnd && currentAction && selectedAnimation && pendingTargetGait) {
				const currentClip = animations.find((clip) => clip.name === selectedAnimation);
				if (currentClip && currentAction.loop === THREE.LoopRepeat) {
					const cycleProgress = currentAction.time % currentClip.duration;
					const timeToEnd = currentClip.duration - cycleProgress;

					// Trigger transition when close to end of cycle (within crossfade duration)
					if (timeToEnd <= 0.3) {
						const nextTarget = pendingTargetGait;
						pendingTargetGait = null;
						waitForCycleEnd = false;
						startTransitionToGait(nextTarget);
					}
				}
			}

			// Check for animation queue transitions
			if (isTransitioning && currentAction && selectedAnimation) {
				const currentClip = animations.find((clip) => clip.name === selectedAnimation);
				if (currentClip && currentAction.time >= currentClip.duration - 0.05) {
					playNextInQueue(); // Simple - handles everything
				}
			}
		}

		renderer.render(scene, camera);
	}
</script>

<svelte:head>
	<title>Cat Walkies - in Three.js by JoniBach</title>
</svelte:head>

<div class="viewer-container">
	<canvas bind:this={canvas}></canvas>

	<div class="controls">
		<div class="button-group">
			<button on:click={() => playTransitionTo('sit')} class="gait-btn"> Sit </button>
			<button on:click={() => playTransitionTo('reach')} class="gait-btn"> Reach </button>

			<button on:click={() => playTransitionTo('stand')} class="gait-btn"> Stand </button>
			<button on:click={() => playTransitionTo('walk')} class="gait-btn"> Walk </button>
			<button on:click={() => playTransitionTo('trot')} class="gait-btn"> Trot </button>
			<button on:click={() => playTransitionTo('gallop')} class="gait-btn"> Gallop </button>
		</div>
		<h1 class="controls-title">
			Cat Walkies by <a target="_blank" class="link" href="https://github.com/JoniBach">JoniBach</a>
		</h1>
	</div>
</div>

<style>
	.viewer-container {
		position: relative;
		width: 100vw;
		height: 100vh;
		overflow: hidden;
	}

	.link {
		color: rgb(101, 186, 225);
	}

	canvas {
		display: block;
		width: 100%;
		height: 100%;
	}

	.controls {
		position: absolute;
		bottom: 10px;
		left: 10px;
		right: 10px;
		background: rgba(0, 0, 0, 0.9);
		color: white;
		padding: 12px;
		border-radius: 8px;
		font-family: Arial, sans-serif;
		backdrop-filter: blur(10px);
		-webkit-backdrop-filter: blur(10px); /* Safari support */
		z-index: 1000; /* Ensure it appears above other elements */

		display: flex;
		flex-wrap: wrap;
		justify-content: space-between;
		align-items: center;
		box-shadow: 0 2px 10px rgba(0, 0, 0, 0.3);
	}

	.button-group {
		display: flex;
		gap: 8px;
		flex-wrap: wrap;
	}

	.controls-title {
		/* margin: 10px 0 0 0; */
		margin: 8px 0;
		font-size: 14px;
	}

	.gait-btn {
		padding: 6px 8px;
		background: #555;
		color: white;
		border: none;
		border-radius: 4px;
		cursor: pointer;
		font-size: 10px;
		font-weight: bold;
		transition: all 0.2s ease;
	}

	.gait-btn:hover {
		background: #777;
		transform: translateY(-1px);
	}

	.gait-btn:active {
		transform: translateY(0);
	}

	/* Mobile-specific improvements */
	@media (max-width: 768px) {
		.controls {
			bottom: 20px;
			left: 15px;
			right: 15px;
			padding: 16px;
			background: rgba(0, 0, 0, 0.95);
			backdrop-filter: blur(20px);
			-webkit-backdrop-filter: blur(20px);
			z-index: 10000; /* Extra high for mobile */
			box-shadow: 0 4px 20px rgba(0, 0, 0, 0.5);
		}

		.button-group {
			gap: 12px;
			margin-bottom: 8px;
		}

		.gait-btn {
			padding: 10px 12px;
			font-size: 12px;
			min-width: 60px;
			border: 1px solid rgba(255, 255, 255, 0.2);
		}

		.controls-title {
			font-size: 12px;
			margin: 4px 0 0 0;
			text-align: center;
			width: 100%;
		}
	}

	:global(body) {
		margin: 0;
		padding: 0;
		overflow: hidden;
	}
</style>
