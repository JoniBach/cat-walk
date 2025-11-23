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

	// Animation sequences for dynamic transitions
	const animationSequences: Record<string, string[]> = {
		stand: ['stand'],
		stand_to_trot: ['stand_to_trot', 'trot'],
		stand_to_walk: ['stand_to_walk', 'walk'],
		stand_to_gallop_direct: ['stand_to_trot', 'trot_to_gallop', 'gallop'],
		stand_to_sit: ['stand_to_sit', 'sit'],
		trot: ['trot'],
		trot_to_stand: ['trot_to_stand', 'stand'],
		trot_to_walk: ['trot_to_walk', 'walk'],
		trot_to_gallop: ['trot_to_gallop', 'gallop'],
		gallop: ['gallop'],
		gallop_to_stand_direct: ['gallop_to_trot', 'trot_to_stand', 'stand'],
		gallop_to_walk_direct: ['gallop_to_trot', 'trot_to_walk', 'walk'],
		gallop_to_trot: ['gallop_to_trot', 'trot'],
		walk: ['walk'],
		walk_to_stand: ['walk_to_stand', 'stand'],
		walk_to_trot: ['walk_to_trot', 'trot'],
		walk_to_gallop_direct: ['walk_to_trot', 'trot_to_gallop', 'gallop'],
		sit: ['sit'],
		sit_to_stand: ['sit_to_stand', 'stand'],
		sit_to_walk_direct: ['sit_to_stand', 'stand_to_walk', 'walk'],
		sit_to_trot_direct: ['sit_to_stand', 'stand_to_trot', 'trot'],
		sit_to_gallop_direct: ['sit_to_stand', 'stand_to_gallop_direct'],
		sit_to_reach: ['sit_to_reach', 'reach'],
		reach: ['reach'],
		reach_to_sit: ['reach_to_sit', 'sit'],
		reach_to_stand_direct: ['reach_to_sit', 'sit_to_stand', 'stand'],
		reach_to_walk_direct: ['reach_to_sit', 'sit_to_walk_direct'],
		reach_to_trot_direct: ['reach_to_sit', 'sit_to_trot_direct'],
		reach_to_gallop_direct: ['reach_to_sit', 'sit_to_gallop_direct'],
	};

	// Transition paths for smart chaining between states
	const transitionPaths: Record<string, string[]> = {
		'stand-stand': ['stand'],
		'stand-trot': ['stand_to_trot'],
		'stand-walk': ['stand_to_walk'],
		'stand-gallop': ['stand_to_gallop_direct'],
		'stand-sit': ['stand_to_sit'],
		'stand-reach': ['stand_to_sit', 'sit_to_reach'],
		'trot-trot': ['trot'],
		'trot-stand': ['trot_to_stand'],
		'trot-walk': ['trot_to_walk'],
		'trot-gallop': ['trot_to_gallop'],
		'trot-sit': ['trot_to_stand', 'stand_to_sit'],
		'trot-reach': ['trot_to_stand', 'stand_to_sit', 'sit_to_reach'],
		'gallop-gallop': ['gallop'],
		'gallop-stand': ['gallop_to_stand_direct'],
		'gallop-walk': ['gallop_to_walk_direct'],
		'gallop-trot': ['gallop_to_trot'],
		'gallop-sit': ['gallop_to_stand_direct', 'stand_to_sit'],
		'gallop-reach': ['gallop_to_stand_direct', 'stand_to_sit', 'sit_to_reach'],
		'walk-walk': ['walk'],
		'walk-stand': ['walk_to_stand'],
		'walk-trot': ['walk_to_trot'],
		'walk-gallop': ['walk_to_gallop_direct'],
		'walk-sit': ['walk_to_stand', 'stand_to_sit'],
		'walk-reach': ['walk_to_stand', 'stand_to_sit', 'sit_to_reach'],
		'sit-sit': ['sit'],
		'sit-stand': ['sit_to_stand'],
		'sit-walk': ['sit_to_walk_direct'],
		'sit-trot': ['sit_to_trot_direct'],
		'sit-gallop': ['sit_to_gallop_direct'],
		'sit-reach': ['sit_to_reach'],
		'reach-reach': ['reach'],
		'reach-stand': ['reach_to_stand_direct'],
		'reach-walk': ['reach_to_walk_direct'],
		'reach-trot': ['reach_to_trot_direct'],
		'reach-gallop': ['reach_to_gallop_direct'],
		'reach-sit': ['reach_to_sit'],
	};

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

	// Transition state
	let isTransitioning = false;

	// Sequence state
	let currentSequence: string[] | null = null;
	let sequenceIndex = 0;
	let sequenceQueue: string[][] = [];
	let pendingTransitions: string[] = [];
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

	function playSequence(sequenceId: string) {
		const sequence = animationSequences[sequenceId];
		if (!sequence || sequence.length === 0) return;

		currentSequence = sequence;
		sequenceIndex = 0;
		isTransitioning = true;

		// Play the first animation in the sequence (transition, no loop)
		playAnimation(sequence[0], false, 0.3);
	}

	function playNextInQueue() {
		if (sequenceQueue.length > 0) {
			currentSequence = sequenceQueue.shift()!;
			sequenceIndex = 0;
			isTransitioning = true;
			playAnimation(currentSequence[0], false, 0.3);
		}
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

	function getCurrentGait(): string {
		if (selectedAnimation.includes('gallop')) return 'gallop';
		if (selectedAnimation.includes('trot')) return 'trot';
		if (selectedAnimation.includes('walk')) return 'walk';
		if (selectedAnimation.includes('sit')) return 'sit';
		if (selectedAnimation.includes('reach')) return 'reach';
		if (selectedAnimation.includes('stand')) return 'stand';
		return 'stand'; // default
	}

	function playTransitionTo(targetGait: string) {
		const currentGait = getCurrentGait();
		if (currentGait === targetGait) return; // already in target gait

		if (isTransitioning) {
			// Queue the transition request
			if (!pendingTransitions.includes(targetGait)) {
				pendingTransitions.push(targetGait);
			}
			return;
		}

		// Check if we need to wait for current cycle to end
		const currentClip = animations.find((clip) => clip.name === selectedAnimation);
		if (currentClip && currentAction && currentAction.loop === THREE.LoopRepeat) {
			// Currently in a looping animation, wait for cycle end
			if (!pendingTransitions.includes(targetGait)) {
				pendingTransitions.push(targetGait);
			}
			waitForCycleEnd = true;
			return;
		}

		// Start transition immediately
		startTransition(targetGait);
		waitForCycleEnd = false; // Reset flag
	}

	function startTransition(targetGait: string) {
		const currentGait = getCurrentGait();
		const pathKey = `${currentGait}-${targetGait}`;
		const path = transitionPaths[pathKey];
		if (!path || path.length === 0) {
			console.log(`No transition path defined for ${pathKey}`);
			return;
		}

		// Queue all sequences in the path
		sequenceQueue = path.map((seqId) => animationSequences[seqId]).filter((seq) => seq);

		if (sequenceQueue.length > 0) {
			playNextInQueue();
		}
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
			if (waitForCycleEnd && currentAction && selectedAnimation) {
				const currentClip = animations.find((clip) => clip.name === selectedAnimation);
				if (currentClip && currentAction.loop === THREE.LoopRepeat) {
					const cycleProgress = currentAction.time % currentClip.duration;
					const timeToEnd = currentClip.duration - cycleProgress;

					// Trigger transition when close to end of cycle (within crossfade duration)
					if (timeToEnd <= 0.3 && pendingTransitions.length > 0) {
						const nextTarget = pendingTransitions.shift()!;
						waitForCycleEnd = false;
						startTransition(nextTarget);
					}
				}
			}

			// Check for sequence transitions
			if (isTransitioning && currentAction && selectedAnimation && currentSequence) {
				const currentClip = animations.find((clip) => clip.name === selectedAnimation);
				if (currentClip && currentAction.time >= currentClip.duration - 0.05) {
					if (sequenceIndex < currentSequence.length - 1) {
						sequenceIndex++;
						const nextAnim = currentSequence[sequenceIndex];
						const isLast = sequenceIndex === currentSequence.length - 1;
						playAnimation(nextAnim, isLast, 0.3); // Last animation in sequence loops
					} else {
						// Sequence completed
						currentSequence = null;
						sequenceIndex = 0;
						isTransitioning = false;

						// Check if there are more sequences in queue
						if (sequenceQueue.length > 0) {
							playNextInQueue();
						} else if (pendingTransitions.length > 0) {
							// Process next pending transition
							const nextTarget = pendingTransitions.shift()!;
							startTransition(nextTarget);
						}
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
		<div class="button-group">
			<button on:click={() => playTransitionTo('sit')} class="gait-btn"> Sit </button>
			<button on:click={() => playTransitionTo('reach')} class="gait-btn"> Reach </button>

			<button on:click={() => playTransitionTo('stand')} class="gait-btn"> Stand </button>
			<button on:click={() => playTransitionTo('walk')} class="gait-btn"> Walk </button>
			<button on:click={() => playTransitionTo('trot')} class="gait-btn"> Trot </button>
			<button on:click={() => playTransitionTo('gallop')} class="gait-btn"> Gallop </button>
		</div>
		<h1 class="controls-title">Cat Walkies</h1>
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
		top: 10px;
		left: 10px;
		right: 10px;
		background: rgba(0, 0, 0, 0.8);
		color: white;
		padding: 8px;
		border-radius: 6px;
		font-family: Arial, sans-serif;
		backdrop-filter: blur(10px);

		display: flex;
		flex-wrap: wrap;
		justify-content: space-between;
		align-items: center;
	}

	.button-group {
		display: flex;
		gap: 8px;
		flex-wrap: wrap;
	}

	.controls-title {
		margin: 0;
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

	:global(body) {
		margin: 0;
		padding: 0;
		overflow: hidden;
	}
</style>
