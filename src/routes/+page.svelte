<script lang="ts">
	import { browser } from '$app/environment';
	import { onMount, onDestroy } from 'svelte';
	import type { Pipeline, RawImage } from '@huggingface/transformers';
	import * as THREE from 'three';
	import { OrbitControls } from 'three/examples/jsm/controls/OrbitControls.js';

	// State management using Svelte 5 runes
	let selectedImage = $state<File | null>(null);
	let imagePreviewUrl = $state<string | null>(null);
	let depthMapUrl = $state<string | null>(null);
	let isProcessing = $state(false);
	let isModelLoading = $state(false);
	let error = $state<string | null>(null);
	let progress = $state<string>('');

	// Depth map adjustment controls
	let brightness = $state(1.0);
	let contrast = $state(1.0);
	let colorize = $state(true);

	// 3D preview controls
	let show3DPreview = $state(false);
	let depthStrength = $state(0.3);
	let autoRotate = $state(false);

	// Store the raw depth data for adjustments
	let rawDepthData = $state<ImageData | null>(null);
	let depthEstimator = $state<Pipeline | null>(null);
	let rawDepthImage: RawImage | null = null;

	// Three.js references
	let canvasContainer: HTMLDivElement;
	let scene: THREE.Scene | null = null;
	let camera: THREE.PerspectiveCamera | null = null;
	let renderer: THREE.WebGLRenderer | null = null;
	let controls: OrbitControls | null = null;
	let mesh: THREE.Mesh | null = null;
	let animationFrameId: number | null = null;

	// File input reference
	let fileInput: HTMLInputElement;

	// Initialize the depth estimation model
	onMount(async () => {
		if (browser) {
			try {
				isModelLoading = true;
				progress = 'Loading Depth Anything model...';

				const { pipeline } = await import('@huggingface/transformers');

				// Load the depth estimation model
				depthEstimator = await pipeline('depth-estimation', 'Xenova/depth-anything-small-hf', {
					progress_callback: (progressData: { status: string; progress?: number }) => {
						if (progressData.progress) {
							progress = `${progressData.status}: ${Math.round(progressData.progress)}%`;
						} else {
							progress = progressData.status;
						}
					}
				});

				isModelLoading = false;
				progress = 'Model loaded successfully!';
				setTimeout(() => (progress = ''), 2000);
			} catch (err) {
				error = `Failed to load model: ${err instanceof Error ? err.message : String(err)}`;
				isModelLoading = false;
			}
		}
	});

	// Handle file selection
	function handleFileSelect(event: Event) {
		const target = event.target as HTMLInputElement;
		const file = target.files?.[0];
		if (file) {
			processFile(file);
		}
	}

	// Handle drag and drop
	function handleDrop(event: DragEvent) {
		event.preventDefault();
		const file = event.dataTransfer?.files[0];
		if (file && file.type.startsWith('image/')) {
			processFile(file);
		}
	}

	function handleDragOver(event: DragEvent) {
		event.preventDefault();
	}

	// Process the uploaded file
	function processFile(file: File) {
		selectedImage = file;
		error = null;
		depthMapUrl = null;
		rawDepthData = null;

		// Create preview URL
		if (imagePreviewUrl) {
			URL.revokeObjectURL(imagePreviewUrl);
		}
		imagePreviewUrl = URL.createObjectURL(file);
	}

	// Generate depth map
	async function generateDepthMap() {
		if (!selectedImage || !depthEstimator) return;

		try {
			isProcessing = true;
			error = null;
			progress = 'Processing image...';

			// Read the image
			const imageUrl = URL.createObjectURL(selectedImage);

			// Run depth estimation
			const output = await depthEstimator(imageUrl);

			// Get the depth map (output.depth is a RawImage)
			const depthImage = output.depth as RawImage;

			// Store raw depth image for 3D preview
			rawDepthImage = depthImage;

			// Convert to canvas for display and manipulation
			const canvas = document.createElement('canvas');
			canvas.width = depthImage.width;
			canvas.height = depthImage.height;
			const ctx = canvas.getContext('2d');

			if (ctx) {
				// Create ImageData from the depth map
				const imageData = ctx.createImageData(canvas.width, canvas.height);
				const depthData = depthImage.data;

				// Store raw depth data for adjustments
				rawDepthData = imageData;

				// Apply initial processing
				applyDepthMapAdjustments(depthData, imageData, brightness, contrast, colorize);

				ctx.putImageData(imageData, 0, 0);

				// Convert canvas to blob URL
				if (depthMapUrl) {
					URL.revokeObjectURL(depthMapUrl);
				}
				depthMapUrl = canvas.toDataURL('image/png');
			}

			URL.revokeObjectURL(imageUrl);
			progress = 'Depth map generated!';
			setTimeout(() => (progress = ''), 2000);
		} catch (err) {
			error = `Failed to generate depth map: ${err instanceof Error ? err.message : String(err)}`;
		} finally {
			isProcessing = false;
		}
	}

	// Apply adjustments to depth map
	function applyDepthMapAdjustments(
		depthData: Uint8Array | Uint8ClampedArray,
		imageData: ImageData,
		brightnessVal: number,
		contrastVal: number,
		colorizeVal: boolean
	) {
		const data = imageData.data;

		for (let i = 0; i < depthData.length; i++) {
			let value = depthData[i];

			// Apply contrast (before brightness)
			value = ((value - 128) * contrastVal + 128);

			// Apply brightness
			value = value * brightnessVal;

			// Clamp to 0-255
			value = Math.max(0, Math.min(255, value));

			if (colorizeVal) {
				// Apply color gradient (blue for near, red for far)
				const normalized = value / 255;
				data[i * 4] = Math.floor(normalized * 255); // R
				data[i * 4 + 1] = Math.floor((1 - Math.abs(normalized - 0.5) * 2) * 255); // G
				data[i * 4 + 2] = Math.floor((1 - normalized) * 255); // B
			} else {
				// Grayscale
				data[i * 4] = value; // R
				data[i * 4 + 1] = value; // G
				data[i * 4 + 2] = value; // B
			}
			data[i * 4 + 3] = 255; // A
		}
	}

	// Update depth map when sliders change
	function updateDepthMapVisualization() {
		if (!rawDepthImage) return;

		// Use cached depth data instead of re-running the model
		const canvas = document.createElement('canvas');
		canvas.width = rawDepthImage.width;
		canvas.height = rawDepthImage.height;
		const ctx = canvas.getContext('2d');

		if (ctx) {
			const imageData = ctx.createImageData(canvas.width, canvas.height);
			applyDepthMapAdjustments(rawDepthImage.data, imageData, brightness, contrast, colorize);
			ctx.putImageData(imageData, 0, 0);

			if (depthMapUrl) {
				URL.revokeObjectURL(depthMapUrl);
			}
			depthMapUrl = canvas.toDataURL('image/png');
		}
	}

	// Download depth map
	function downloadDepthMap() {
		if (!depthMapUrl) return;

		const link = document.createElement('a');
		link.download = `depth-map-${selectedImage?.name || 'image'}.png`;
		link.href = depthMapUrl;
		link.click();
	}

	// Reset state
	function reset() {
		if (imagePreviewUrl) {
			URL.revokeObjectURL(imagePreviewUrl);
		}
		if (depthMapUrl) {
			URL.revokeObjectURL(depthMapUrl);
		}
		selectedImage = null;
		imagePreviewUrl = null;
		depthMapUrl = null;
		rawDepthData = null;
		rawDepthImage = null;
		error = null;
		brightness = 1.0;
		contrast = 1.0;
		colorize = true;
		show3DPreview = false;
		cleanup3DScene();
	}

	// Initialize 3D scene
	async function init3DScene() {
		if (!browser || !canvasContainer || !imagePreviewUrl || !rawDepthImage) return;

		// Clean up existing scene
		cleanup3DScene();

		// Create scene
		scene = new THREE.Scene();
		scene.background = new THREE.Color(0x1a1a1a);

		// Create camera
		camera = new THREE.PerspectiveCamera(
			50,
			canvasContainer.clientWidth / canvasContainer.clientHeight,
			0.1,
			1000
		);
		camera.position.z = 2;

		// Create renderer
		renderer = new THREE.WebGLRenderer({ antialias: true });
		renderer.setSize(canvasContainer.clientWidth, canvasContainer.clientHeight);
		renderer.setPixelRatio(window.devicePixelRatio);
		canvasContainer.appendChild(renderer.domElement);

		// Add orbit controls
		controls = new OrbitControls(camera, renderer.domElement);
		controls.enableDamping = true;
		controls.dampingFactor = 0.05;
		controls.autoRotate = autoRotate;
		controls.autoRotateSpeed = 2.0;

		// Add lights
		const ambientLight = new THREE.AmbientLight(0xffffff, 0.6);
		scene.add(ambientLight);

		const directionalLight = new THREE.DirectionalLight(0xffffff, 0.8);
		directionalLight.position.set(5, 5, 5);
		scene.add(directionalLight);

		// Load textures
		const textureLoader = new THREE.TextureLoader();
		const imageTexture = await new Promise<THREE.Texture>((resolve, reject) => {
			textureLoader.load(imagePreviewUrl!, resolve, undefined, reject);
		});

		// Create depth texture from raw depth data
		const depthCanvas = document.createElement('canvas');
		depthCanvas.width = rawDepthImage.width;
		depthCanvas.height = rawDepthImage.height;
		const depthCtx = depthCanvas.getContext('2d');

		if (depthCtx) {
			const depthImageData = depthCtx.createImageData(depthCanvas.width, depthCanvas.height);
			for (let i = 0; i < rawDepthImage.data.length; i++) {
				const value = rawDepthImage.data[i];
				depthImageData.data[i * 4] = value;
				depthImageData.data[i * 4 + 1] = value;
				depthImageData.data[i * 4 + 2] = value;
				depthImageData.data[i * 4 + 3] = 255;
			}
			depthCtx.putImageData(depthImageData, 0, 0);
		}

		const depthTexture = new THREE.CanvasTexture(depthCanvas);

		// Create plane geometry with high resolution for displacement
		const geometry = new THREE.PlaneGeometry(2, 2, 256, 256);

		// Create material with displacement
		const material = new THREE.MeshStandardMaterial({
			map: imageTexture,
			displacementMap: depthTexture,
			displacementScale: depthStrength,
			roughness: 0.8,
			metalness: 0.2
		});

		// Create mesh
		mesh = new THREE.Mesh(geometry, material);
		scene.add(mesh);

		// Start animation loop
		animate();
	}

	// Animation loop
	function animate() {
		if (!scene || !camera || !renderer || !controls) return;

		animationFrameId = requestAnimationFrame(animate);

		controls.update();
		renderer.render(scene, camera);
	}

	// Update 3D scene when depth strength or auto-rotate changes
	function update3DSettings() {
		if (mesh && mesh.material instanceof THREE.MeshStandardMaterial) {
			mesh.material.displacementScale = depthStrength;
		}
		if (controls) {
			controls.autoRotate = autoRotate;
		}
	}

	// Clean up 3D scene
	function cleanup3DScene() {
		if (animationFrameId !== null) {
			cancelAnimationFrame(animationFrameId);
			animationFrameId = null;
		}

		if (mesh) {
			if (mesh.material instanceof THREE.Material) {
				mesh.material.dispose();
			}
			if (mesh.geometry) {
				mesh.geometry.dispose();
			}
			mesh = null;
		}

		if (controls) {
			controls.dispose();
			controls = null;
		}

		if (renderer) {
			renderer.dispose();
			if (renderer.domElement && canvasContainer) {
				canvasContainer.removeChild(renderer.domElement);
			}
			renderer = null;
		}

		scene = null;
		camera = null;
	}

	// Toggle 3D preview
	function toggle3DPreview() {
		show3DPreview = !show3DPreview;
		if (show3DPreview) {
			setTimeout(() => init3DScene(), 100);
		} else {
			cleanup3DScene();
		}
	}

	// Cleanup on component destroy
	onDestroy(() => {
		cleanup3DScene();
	});
</script>

<div class="container mx-auto px-4 py-8 max-w-7xl">
	<header class="mb-8 text-center">
		<h1 class="text-4xl font-bold mb-2">Depth Map Generator</h1>
		<p class="text-gray-600">
			Generate depth maps from images using AI-powered Depth Anything model
		</p>
	</header>

	{#if isModelLoading}
		<div class="bg-blue-50 border border-blue-200 rounded-lg p-6 mb-6">
			<div class="flex items-center gap-3">
				<div class="animate-spin rounded-full h-6 w-6 border-b-2 border-blue-600"></div>
				<p class="text-blue-800">{progress}</p>
			</div>
		</div>
	{/if}

	{#if error}
		<div class="bg-red-50 border border-red-200 rounded-lg p-4 mb-6">
			<p class="text-red-800">{error}</p>
		</div>
	{/if}

	{#if progress && !isModelLoading}
		<div class="bg-green-50 border border-green-200 rounded-lg p-4 mb-6">
			<p class="text-green-800">{progress}</p>
		</div>
	{/if}

	<!-- Upload Section -->
	<div class="mb-8">
		<div
			class="border-2 border-dashed border-gray-300 rounded-lg p-8 text-center hover:border-blue-500 transition-colors cursor-pointer"
			ondrop={handleDrop}
			ondragover={handleDragOver}
			onclick={() => fileInput.click()}
			onkeydown={(e) => {
				if (e.key === 'Enter' || e.key === ' ') {
					e.preventDefault();
					fileInput.click();
				}
			}}
			role="button"
			tabindex="0"
		>
			<svg
				class="mx-auto h-12 w-12 text-gray-400 mb-4"
				stroke="currentColor"
				fill="none"
				viewBox="0 0 48 48"
			>
				<path
					d="M28 8H12a4 4 0 00-4 4v20m32-12v8m0 0v8a4 4 0 01-4 4H12a4 4 0 01-4-4v-4m32-4l-3.172-3.172a4 4 0 00-5.656 0L28 28M8 32l9.172-9.172a4 4 0 015.656 0L28 28m0 0l4 4m4-24h8m-4-4v8m-12 4h.02"
					stroke-width="2"
					stroke-linecap="round"
					stroke-linejoin="round"
				/>
			</svg>
			<p class="text-lg mb-2">
				{selectedImage ? selectedImage.name : 'Drop an image here or click to select'}
			</p>
			<p class="text-sm text-gray-500">PNG, JPG, WEBP up to 10MB</p>
			<input
				bind:this={fileInput}
				type="file"
				accept="image/*"
				onchange={handleFileSelect}
				class="hidden"
			/>
		</div>

		{#if selectedImage && !depthMapUrl}
			<div class="mt-4 flex gap-3 justify-center">
				<button
					onclick={generateDepthMap}
					disabled={isProcessing || isModelLoading || !depthEstimator}
					class="bg-blue-600 text-white px-6 py-2 rounded-lg hover:bg-blue-700 disabled:bg-gray-400 disabled:cursor-not-allowed transition-colors"
				>
					{isProcessing ? 'Processing...' : 'Generate Depth Map'}
				</button>
				<button
					onclick={reset}
					class="bg-gray-600 text-white px-6 py-2 rounded-lg hover:bg-gray-700 transition-colors"
				>
					Reset
				</button>
			</div>
		{/if}
	</div>

	<!-- Image Preview and Depth Map Display -->
	{#if imagePreviewUrl || depthMapUrl}
		<div class="grid md:grid-cols-2 gap-6 mb-8">
			<!-- Original Image -->
			{#if imagePreviewUrl}
				<div class="bg-white rounded-lg shadow-lg p-4">
					<h3 class="text-lg font-semibold mb-3">Original Image</h3>
					<img src={imagePreviewUrl} alt="Original" class="w-full rounded-lg" />
				</div>
			{/if}

			<!-- Depth Map -->
			{#if depthMapUrl}
				<div class="bg-white rounded-lg shadow-lg p-4">
					<h3 class="text-lg font-semibold mb-3">Depth Map</h3>
					<img src={depthMapUrl} alt="Depth Map" class="w-full rounded-lg mb-3" />
					<button
						onclick={downloadDepthMap}
						class="w-full bg-green-600 text-white px-4 py-2 rounded-lg hover:bg-green-700 transition-colors"
					>
						Download Depth Map
					</button>
				</div>
			{/if}
		</div>
	{/if}

	<!-- Controls -->
	{#if depthMapUrl}
		<div class="bg-white rounded-lg shadow-lg p-6 mb-8">
			<h3 class="text-lg font-semibold mb-4">Depth Map Adjustments</h3>

			<div class="space-y-4">
				<!-- Brightness -->
				<div>
					<div class="flex items-center justify-between mb-2">
						<label for="brightness-slider" class="font-medium">Brightness</label>
						<span class="text-sm text-gray-600">{brightness.toFixed(2)}</span>
					</div>
					<input
						id="brightness-slider"
						type="range"
						min="0.1"
						max="2.0"
						step="0.1"
						bind:value={brightness}
						oninput={updateDepthMapVisualization}
						class="w-full"
					/>
				</div>

				<!-- Contrast -->
				<div>
					<div class="flex items-center justify-between mb-2">
						<label for="contrast-slider" class="font-medium">Contrast</label>
						<span class="text-sm text-gray-600">{contrast.toFixed(2)}</span>
					</div>
					<input
						id="contrast-slider"
						type="range"
						min="0.1"
						max="3.0"
						step="0.1"
						bind:value={contrast}
						oninput={updateDepthMapVisualization}
						class="w-full"
					/>
				</div>

				<!-- Colorize Toggle -->
				<div>
					<label class="flex items-center gap-2">
						<input
							type="checkbox"
							bind:checked={colorize}
							oninput={updateDepthMapVisualization}
							class="w-4 h-4"
						/>
						<span class="font-medium">Colorize (Blue = Near, Red = Far)</span>
					</label>
				</div>
			</div>

			<div class="mt-4 pt-4 border-t">
				<button
					onclick={reset}
					class="w-full bg-gray-600 text-white px-4 py-2 rounded-lg hover:bg-gray-700 transition-colors"
				>
					Process New Image
				</button>
			</div>
		</div>
	{/if}

	<!-- 3D Preview Section -->
	{#if depthMapUrl}
		<div class="bg-white rounded-lg shadow-lg p-6 mb-8">
			<div class="flex items-center justify-between mb-4">
				<h3 class="text-lg font-semibold">3D Preview</h3>
				<button
					onclick={toggle3DPreview}
					class={`px-4 py-2 rounded-lg transition-colors ${
						show3DPreview
							? 'bg-red-600 hover:bg-red-700 text-white'
							: 'bg-blue-600 hover:bg-blue-700 text-white'
					}`}
				>
					{show3DPreview ? 'Close 3D Preview' : 'View in 3D'}
				</button>
			</div>

			{#if show3DPreview}
				<div class="space-y-4">
					<!-- 3D Canvas Container -->
					<div
						bind:this={canvasContainer}
						class="w-full h-[500px] bg-gray-900 rounded-lg overflow-hidden"
					></div>

					<!-- 3D Controls -->
					<div class="space-y-4 bg-gray-50 p-4 rounded-lg">
						<!-- Depth Strength -->
						<div>
							<div class="flex items-center justify-between mb-2">
								<label for="depth-strength-slider" class="font-medium">Depth Strength</label>
								<span class="text-sm text-gray-600">{depthStrength.toFixed(2)}</span>
							</div>
							<input
								id="depth-strength-slider"
								type="range"
								min="0"
								max="1.0"
								step="0.05"
								bind:value={depthStrength}
								oninput={update3DSettings}
								class="w-full"
							/>
						</div>

						<!-- Auto Rotate -->
						<div>
							<label class="flex items-center gap-2">
								<input
									type="checkbox"
									bind:checked={autoRotate}
									onchange={update3DSettings}
									class="w-4 h-4"
								/>
								<span class="font-medium">Auto Rotate</span>
							</label>
						</div>

						<div class="text-sm text-gray-600 bg-blue-50 p-3 rounded">
							<strong>💡 Tip:</strong> Click and drag to rotate, scroll to zoom, right-click and drag
							to pan
						</div>
					</div>
				</div>
			{:else}
				<p class="text-gray-600 text-center py-8">
					Click "View in 3D" to see your image in 3D with depth displacement
				</p>
			{/if}
		</div>
	{/if}

	<!-- Info Section -->
	<div class="bg-blue-50 rounded-lg p-6">
		<h3 class="text-lg font-semibold mb-2">About Depth Estimation</h3>
		<p class="text-gray-700 mb-2">
			This application uses the <strong>Depth Anything</strong> model to generate monocular depth
			maps from single images. The model estimates the relative distance of objects in the scene.
		</p>
		<ul class="list-disc list-inside text-gray-700 space-y-1">
			<li>Brighter/Red areas indicate objects farther from the camera</li>
			<li>Darker/Blue areas indicate objects closer to the camera</li>
			<li>Adjust brightness and contrast to enhance depth visualization</li>
			<li>View your image in interactive 3D with depth displacement mapping</li>
			<li>Rotate, zoom, and pan the 3D preview to explore depth from all angles</li>
			<li>Download the depth map for use in 3D applications, visual effects, and more</li>
		</ul>
	</div>
</div>

<style>
	/* Custom styles for better slider appearance */
	input[type='range'] {
		-webkit-appearance: none;
		appearance: none;
		height: 8px;
		border-radius: 4px;
		background: #e5e7eb;
		outline: none;
	}

	input[type='range']::-webkit-slider-thumb {
		-webkit-appearance: none;
		appearance: none;
		width: 20px;
		height: 20px;
		border-radius: 50%;
		background: #2563eb;
		cursor: pointer;
	}

	input[type='range']::-moz-range-thumb {
		width: 20px;
		height: 20px;
		border-radius: 50%;
		background: #2563eb;
		cursor: pointer;
		border: none;
	}
</style>
