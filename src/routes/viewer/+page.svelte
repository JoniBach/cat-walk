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
			shadowMapType: THREE.PCFSoftShadowMap
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
				shadowMapSize: 2048
			}
		},

		// Ground settings
		ground: {
			size: 20,
			color: 0x555555 // Match scene background
		},

		// Model settings
		model: {
			scale: 2.0 // Default scale multiplier
		},

		// Controls settings
		controls: {
			enableDamping: true,
			dampingFactor: 0.05,
			screenSpacePanning: false,
			minDistance: 2,
			maxDistance: 10,
			maxPolarAngle: Math.PI / 2
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
		directionalLight.shadow.mapSize.width = CONFIG.lighting.directional.shadowMapSize;
		directionalLight.shadow.mapSize.height = CONFIG.lighting.directional.shadowMapSize;
		scene.add(directionalLight);

		// Ground
		const groundGeometry = new THREE.PlaneGeometry(CONFIG.ground.size, CONFIG.ground.size);
		const groundMaterial = new THREE.MeshLambertMaterial({ color: CONFIG.ground.color });
		const ground = new THREE.Mesh(groundGeometry, groundMaterial);
		ground.rotation.x = -Math.PI / 2;
		ground.receiveShadow = true;
		scene.add(ground);

		// Mouse controls
		controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = CONFIG.controls.enableDamping;
		controls.dampingFactor = CONFIG.controls.dampingFactor;
		controls.screenSpacePanning = CONFIG.controls.screenSpacePanning;
		controls.minDistance = CONFIG.controls.minDistance;
		controls.maxDistance = CONFIG.controls.maxDistance;
		controls.maxPolarAngle = CONFIG.controls.maxPolarAngle;

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

					// Set default animation
					if (animationNames.length > 0) {
						selectedAnimation = animationNames[0];
					}
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

	function playAnimation(animationName: string) {
		if (!mixer || !animations) return;

		// Stop current animation
		if (currentAction) {
			currentAction.stop();
		}

		// Find and play new animation
		const clip = animations.find((clip) => clip.name === animationName);
		if (clip) {
			currentAction = mixer.clipAction(clip);
			currentAction.reset();
			currentAction.play();
			isPlaying = true;
		}
	}

	function stopAnimation() {
		if (currentAction) {
			currentAction.stop();
			isPlaying = false;
		}
	}

	function updateModelScale() {
		if (model) {
			model.scale.setScalar(modelScale);
		}
	}

	function onAnimationChange() {
		if (selectedAnimation) {
			playAnimation(selectedAnimation);
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
		<h2>Model Controls</h2>

		<div class="control-group">
			<label for="scale-slider">Model Scale: {modelScale.toFixed(2)}</label>
			<input
				type="range"
				id="scale-slider"
				min="0.1"
				max="3.0"
				step="0.1"
				bind:value={modelScale}
				on:input={updateModelScale}
			/>
		</div>

		<h2>Animation Controls</h2>

		<div class="control-group">
			<label for="animation-select">Select Animation:</label>
			<select
				id="animation-select"
				bind:value={selectedAnimation}
				on:change={onAnimationChange}
				disabled={animationNames.length === 0}
			>
				{#each animationNames as animationName}
					<option value={animationName}>{animationName}</option>
				{/each}
			</select>
		</div>

		<div class="control-group">
			<button on:click={onAnimationChange} disabled={!selectedAnimation} class="play-btn">
				Play Animation
			</button>

			<button on:click={stopAnimation} disabled={!isPlaying} class="stop-btn">
				Stop Animation
			</button>
		</div>

		<div class="info">
			<p>Status: {isPlaying ? 'Playing' : 'Stopped'}</p>
			<p>Current: {selectedAnimation || 'None'}</p>
			<p>Available: {animationNames.length} animations</p>
		</div>
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
		right: 20px;
		background: rgba(0, 0, 0, 0.8);
		color: white;
		padding: 20px;
		border-radius: 8px;
		min-width: 250px;
		font-family: Arial, sans-serif;
	}

	.controls h2 {
		margin: 0 0 15px 0;
		font-size: 18px;
		color: #fff;
	}

	.control-group {
		margin-bottom: 15px;
	}

	.control-group label {
		display: block;
		margin-bottom: 5px;
		font-size: 14px;
		color: #ccc;
	}

	select {
		width: 100%;
		padding: 8px;
		border: 1px solid #555;
		border-radius: 4px;
		background: #333;
		color: white;
		font-size: 14px;
	}

	select:disabled {
		opacity: 0.5;
		cursor: not-allowed;
	}

	input[type='range'] {
		width: 100%;
		margin-top: 5px;
	}

	button {
		padding: 10px 15px;
		margin-right: 10px;
		border: none;
		border-radius: 4px;
		cursor: pointer;
		font-size: 14px;
		transition: background-color 0.2s;
	}

	.play-btn {
		background: #4caf50;
		color: white;
	}

	.play-btn:hover:not(:disabled) {
		background: #45a049;
	}

	.stop-btn {
		background: #f44336;
		color: white;
	}

	.stop-btn:hover:not(:disabled) {
		background: #da190b;
	}

	button:disabled {
		opacity: 0.5;
		cursor: not-allowed;
	}

	.info {
		margin-top: 15px;
		padding-top: 15px;
		border-top: 1px solid #555;
	}

	.info p {
		margin: 5px 0;
		font-size: 12px;
		color: #ccc;
	}

	:global(body) {
		margin: 0;
		padding: 0;
		overflow: hidden;
	}
</style>
