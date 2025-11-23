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

	// Walking state
	let isWalking = false;

	// Transition state
	let isTransitioning = false;
	let pendingTransition: string | null = null;

	// Walk stop transition state
	let shouldStopWalking = false;

	// Model scale control
	let modelScale = CONFIG.model.scale;

	function handleKeyDown(event: KeyboardEvent) {
		if (event.code === 'Space' || event.code === 'Enter') {
			event.preventDefault(); // Prevent default behavior (page scroll for space, form submit for enter)
			toggleWalk();
		}
	}

	onMount(() => {
		initThreeJS();
		loadGLTF();
		animate();

		// Add keyboard event listeners
		window.addEventListener('keydown', handleKeyDown);

		return () => {
			if (controls) {
				controls.dispose();
			}
			if (renderer) {
				renderer.dispose();
			}
			// Remove keyboard event listeners
			window.removeEventListener('keydown', handleKeyDown);
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

	function stopAnimation() {
		if (currentAction) {
			currentAction.stop();
			isPlaying = false;
		}
	}

	function toggleWalk() {
		const wasWalking = isWalking;
		isWalking = !isWalking;

		if (isWalking) {
			// Start walking: stand-to-walk → walk (seamless with crossfade)
			playAnimation('stand-to-walk', false, 0.3); // Longer crossfade for smoother transition
			isTransitioning = true;
			pendingTransition = 'walk';
		} else {
			// Stop walking: set flag to transition at end of current walk cycle
			if (wasWalking && selectedAnimation === 'walk') {
				// Currently in walk loop, wait for cycle to finish
				shouldStopWalking = true;
			} else {
				// Not in walk loop, directly transition
				playAnimation('Walk to stand', false, 0.3); // Longer crossfade for smoother transition
				isTransitioning = true;
				pendingTransition = 'stand';
			}
		}
	}

	function getAnimationDuration(animationName: string): number {
		const clip = animations.find((clip) => clip.name === animationName);
		return clip ? clip.duration : 1.0;
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

			// Check for walk cycle end to stop walking
			if (shouldStopWalking && selectedAnimation === 'walk' && currentAction) {
				const walkClip = animations.find((clip) => clip.name === 'walk');
				if (walkClip) {
					const cycleProgress = currentAction.time % walkClip.duration;
					const timeToEnd = walkClip.duration - cycleProgress;

					// Trigger transition when very close to end of cycle (within 0.1 seconds)
					if (timeToEnd <= 0.1) {
						playAnimation('Walk to stand', false, 0.4); // Even longer crossfade for walk-to-stand
						isTransitioning = true;
						pendingTransition = 'stand';
						shouldStopWalking = false;
					}
				}
			}

			// Check for seamless transitions
			if (isTransitioning && currentAction && selectedAnimation) {
				const currentClip = animations.find((clip) => clip.name === selectedAnimation);
				if (currentClip && currentAction.time >= currentClip.duration - 0.05) {
					// Near end of animation
					if (pendingTransition) {
						if (pendingTransition === 'walk') {
							playAnimation('walk', true, 0.3); // Crossfade to walk loop
						} else if (pendingTransition === 'stand') {
							playAnimation('stand', true, 0.3); // Crossfade to stand loop
						}
						isTransitioning = false;
						pendingTransition = null;
					}
				}
			}
		}

		renderer.render(scene, camera);
	}
</script>

<svelte:head>
	<title>GLTF Cat Viewer</title>
</svelte:head>

<div class="viewer-container">
	<canvas bind:this={canvas}></canvas>

	<div class="controls">
		<h1>Cat Walk</h1>
		<button on:click={toggleWalk} class="walk-btn">
			{isWalking ? 'Stop Walking' : 'Start Walking'}
		</button>
		<p class="keyboard-hint">Press <kbd>Space</kbd> or <kbd>Enter</kbd></p>
	</div>
</div>

<style>
	.viewer-container {
		position: relative;
		width: 100vw;
		height: 100vh;
		overflow: hidden;
	}

	canvas {
		display: block;
		width: 100%;
		height: 100%;
	}

	.controls {
		position: absolute;
		top: 20px;
		left: 20px;
		background: rgba(0, 0, 0, 0.8);
		color: white;
		padding: 12px;
		border-radius: 8px;
		min-width: 180px;
		font-family: Arial, sans-serif;
		backdrop-filter: blur(10px);
	}

	.controls h1 {
		margin: 0 0 10px 0;
		font-size: 18px;
		color: #fff;
		text-align: center;
	}

	.walk-btn {
		width: 100%;
		padding: 8px 16px;
		background: #2196f3;
		color: white;
		border: none;
		border-radius: 6px;
		cursor: pointer;
		font-size: 14px;
		font-weight: bold;
		transition: all 0.2s ease;
	}

	.walk-btn:hover {
		background: #1976d2;
		transform: translateY(-1px);
	}

	.walk-btn:active {
		transform: translateY(0);
	}

	.keyboard-hint {
		margin: 8px 0 0 0;
		font-size: 10px;
		color: #ccc;
		text-align: center;
		opacity: 0.8;
	}

	.keyboard-hint kbd {
		background: rgba(255, 255, 255, 0.1);
		border: 1px solid rgba(255, 255, 255, 0.2);
		border-radius: 3px;
		padding: 1px 3px;
		font-size: 9px;
		font-family: monospace;
		color: #fff;
	}

	/* Mobile responsiveness */
	@media (max-width: 768px) {
		.controls {
			top: 10px;
			left: 10px;
			right: 10px;
			min-width: auto;
			padding: 10px;
			border-radius: 6px;
		}

		.controls h1 {
			font-size: 16px;
			margin-bottom: 8px;
		}

		.walk-btn {
			padding: 10px 12px;
			font-size: 14px;
		}

		.keyboard-hint {
			font-size: 9px;
			margin-top: 6px;
		}
	}

	@media (max-width: 480px) {
		.controls {
			padding: 8px;
		}

		.controls h1 {
			font-size: 14px;
			margin-bottom: 6px;
		}

		.walk-btn {
			padding: 12px 10px;
			font-size: 13px;
		}

		.keyboard-hint {
			font-size: 8px;
			margin-top: 4px;
		}
	}

	:global(body) {
		margin: 0;
		padding: 0;
		overflow: hidden;
	}
</style>
