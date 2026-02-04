<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import RAPIER from '@dimforge/rapier3d-compat';

	let container: HTMLDivElement | null = null;
	let joystickEl: HTMLDivElement | null = null;
	let joystickThumbEl: HTMLDivElement | null = null;
	let jumpEl: HTMLDivElement | null = null;
	let worldLabel = 'Verdant Expanse';
	let worldJump: ((id: 'earth' | 'mars' | 'moon') => void) | null = null;

	const jumpWorld = (id: 'earth' | 'mars' | 'moon') => worldJump?.(id);

	onMount(() => {
		let dispose = () => {};
		let cancelled = false;

		const init = async () => {
			if (!container) {
				return;
			}

			await RAPIER.init();
			if (cancelled || !container) {
				return;
			}

			const scene = new THREE.Scene();
			scene.background = new THREE.Color('#0b1216');
			scene.fog = new THREE.Fog(0x0b1216, 18, 90);

			const renderer = new THREE.WebGLRenderer({
				antialias: true,
				alpha: false,
				preserveDrawingBuffer: true
			});
			renderer.setPixelRatio(window.devicePixelRatio);
			renderer.outputColorSpace = THREE.SRGBColorSpace;
			container.appendChild(renderer.domElement);

			const camera = new THREE.PerspectiveCamera(65, 1, 0.1, 420);
			const cameraRig = new THREE.Group();
			cameraRig.rotation.order = 'YXZ';
			const baseCameraOffset = new THREE.Vector3(0, 4.6, 7.5);
			const defaultCameraDistance = baseCameraOffset.length();
			let cameraDistance = defaultCameraDistance;
			let cameraPitch = 0;
			const minPitch = -0.6;
			const maxPitch = 0.6;
			const collisionDampIn = 8;
			const collisionDampOut = 3.5;
			camera.position.copy(baseCameraOffset);
			cameraRig.add(camera);
			scene.add(cameraRig);

			const hemiLight = new THREE.HemisphereLight(0xeef6ff, 0x22303a, 0.9);
			scene.add(hemiLight);

			const dirLight = new THREE.DirectionalLight(0xffffff, 1.2);
			dirLight.position.set(6, 12, 4);
			scene.add(dirLight);

			const textureLoader = new THREE.TextureLoader();
			const loadTexture = (url: string) => {
				const texture = textureLoader.load(url);
				texture.colorSpace = THREE.SRGBColorSpace;
				texture.magFilter = THREE.NearestFilter;
				texture.minFilter = THREE.NearestMipMapNearestFilter;
				texture.wrapS = THREE.RepeatWrapping;
				texture.wrapT = THREE.RepeatWrapping;
				return texture;
			};
			const createCanvasTexture = (draw: (ctx: CanvasRenderingContext2D, size: number) => void) => {
				const size = 64;
				const canvas = document.createElement('canvas');
				canvas.width = size;
				canvas.height = size;
				const ctx = canvas.getContext('2d');
				if (ctx) {
					ctx.imageSmoothingEnabled = false;
					draw(ctx, size);
				}
				const texture = new THREE.CanvasTexture(canvas);
				texture.colorSpace = THREE.SRGBColorSpace;
				texture.magFilter = THREE.NearestFilter;
				texture.minFilter = THREE.NearestMipMapNearestFilter;
				texture.wrapS = THREE.RepeatWrapping;
				texture.wrapT = THREE.RepeatWrapping;
				return texture;
			};

			const grassTopTex = loadTexture('/textures/grass_top.png');
			const grassSideTex = loadTexture('/textures/grass_side.png');
			const dirtTex = loadTexture('/textures/dirt.png');
			const stoneTex = loadTexture('/textures/stone.png');
			const cobbleTex = loadTexture('/textures/cobble.png');
			const woodPlankTex = loadTexture('/textures/wood_plank.png');
			const redWoodPlankTex = loadTexture('/textures/red_wood_plank.png');
			const sandstoneTex = loadTexture('/textures/sandstone.png');
			const obsidianTex = loadTexture('/textures/obsidian.png');
			const marsSandTex = loadTexture('/textures/mars_sand.png');
			const marsRockTex = loadTexture('/textures/mars_rock.png');
			const moonDustTex = loadTexture('/textures/moon_dust.png');
			const sandTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#d9c58c';
				ctx.fillRect(0, 0, size, size);
				for (let i = 0; i < 140; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const shade = 0.9 + Math.random() * 0.15;
					ctx.fillStyle = `rgba(186, 164, 105, ${shade})`;
					ctx.fillRect(x, y, 1, 1);
				}
				for (let i = 0; i < 60; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					ctx.fillStyle = 'rgba(242, 232, 196, 0.9)';
					ctx.fillRect(x, y, 1, 1);
				}
			});
			const gravelTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#8a8e8f';
				ctx.fillRect(0, 0, size, size);
				for (let i = 0; i < 220; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const tone = 110 + Math.floor(Math.random() * 55);
					ctx.fillStyle = `rgb(${tone}, ${tone}, ${tone})`;
					ctx.fillRect(x, y, 2, 2);
				}
				for (let i = 0; i < 140; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const tone = 70 + Math.floor(Math.random() * 40);
					ctx.fillStyle = `rgba(${tone}, ${tone}, ${tone}, 0.95)`;
					ctx.fillRect(x, y, 1, 1);
				}
			});
			const waterTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#1f6fb6';
				ctx.fillRect(0, 0, size, size);
				for (let y = 0; y < size; y += 1) {
					const shade = 115 + Math.floor(60 * (y / size));
					ctx.fillStyle = `rgba(60, ${shade}, 215, 0.45)`;
					for (let x = 0; x < size; x += 4) {
						if ((x + y) % 9 === 0) ctx.fillRect(x, y, 2, 1);
					}
				}
				for (let i = 0; i < 120; i += 1) {
					ctx.fillStyle = `rgba(165, 230, 255, ${0.06 + Math.random() * 0.12})`;
					ctx.fillRect(Math.floor(Math.random() * size), Math.floor(Math.random() * size), 1, 1);
				}
			});
			const lavaTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#2a0b06';
				ctx.fillRect(0, 0, size, size);
				for (let i = 0; i < 520; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const hot = Math.random();
					const color =
						hot > 0.85
							? 'rgba(255, 233, 140, 0.95)'
							: hot > 0.55
								? 'rgba(255, 132, 28, 0.9)'
								: 'rgba(161, 52, 12, 0.85)';
					ctx.fillStyle = color;
					ctx.fillRect(x, y, hot > 0.7 ? 2 : 1, 1);
				}
				ctx.fillStyle = 'rgba(0,0,0,0.35)';
				for (let i = 0; i < 90; i += 1) {
					ctx.fillRect(Math.floor(Math.random() * size), Math.floor(Math.random() * size), 1, 1);
				}
			});
			const logTopTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#9b6a35';
				ctx.fillRect(0, 0, size, size);
				ctx.fillStyle = '#6d4525';
				for (let i = 0; i < 7; i += 1) {
					const inset = i * 4;
					ctx.strokeStyle = i % 2 === 0 ? '#6d4525' : '#83562e';
					ctx.strokeRect(inset, inset, size - inset * 2, size - inset * 2);
				}
			});
			const logSideTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#8a5b2e';
				ctx.fillRect(0, 0, size, size);
				for (let x = 0; x < size; x += 6) {
					ctx.fillStyle = x % 12 === 0 ? '#6f4424' : '#7b4f29';
					ctx.fillRect(x, 0, 2, size);
				}
				ctx.fillStyle = 'rgba(255, 232, 200, 0.08)';
				for (let i = 0; i < 80; i += 1) {
					ctx.fillRect(Math.floor(Math.random() * size), Math.floor(Math.random() * size), 1, 1);
				}
			});
			const leavesTex = createCanvasTexture((ctx, size) => {
				ctx.clearRect(0, 0, size, size);
				ctx.fillStyle = 'rgba(46, 140, 66, 0.85)';
				ctx.fillRect(0, 0, size, size);
				for (let i = 0; i < 320; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const alpha = Math.random() < 0.12 ? 0 : 0.55 + Math.random() * 0.4;
					const g = 120 + Math.floor(Math.random() * 90);
					ctx.fillStyle = `rgba(40, ${g}, 60, ${alpha})`;
					ctx.fillRect(x, y, 2, 2);
				}
				for (let i = 0; i < 120; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					ctx.clearRect(x, y, 1, 1);
				}
			});
			const glassTex = createCanvasTexture((ctx, size) => {
				ctx.clearRect(0, 0, size, size);
				ctx.fillStyle = 'rgba(170, 230, 255, 0.14)';
				ctx.fillRect(0, 0, size, size);
				ctx.strokeStyle = 'rgba(210, 246, 255, 0.55)';
				ctx.lineWidth = 2;
				ctx.strokeRect(2, 2, size - 4, size - 4);
				ctx.strokeStyle = 'rgba(210, 246, 255, 0.22)';
				for (let i = 0; i < 6; i += 1) {
					ctx.beginPath();
					ctx.moveTo(0, Math.floor((i / 6) * size));
					ctx.lineTo(size, Math.floor((i / 6) * size));
					ctx.stroke();
				}
			});
			const brickTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#9b3c30';
				ctx.fillRect(0, 0, size, size);
				const brickH = 10;
				const brickW = 16;
				for (let y = 0; y < size; y += brickH) {
					const offset = (y / brickH) % 2 === 0 ? 0 : brickW / 2;
					for (let x = -offset; x < size; x += brickW) {
						ctx.fillStyle = '#a54435';
						ctx.fillRect(Math.floor(x) + 1, y + 1, brickW - 2, brickH - 2);
					}
				}
				ctx.fillStyle = 'rgba(30, 12, 8, 0.38)';
				for (let y = 0; y < size; y += brickH) {
					ctx.fillRect(0, y, size, 1);
				}
				for (let x = 0; x < size; x += brickW / 2) {
					ctx.fillRect(x, 0, 1, size);
				}
			});
			const doorTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#9a6a36';
				ctx.fillRect(0, 0, size, size);
				for (let x = 0; x < size; x += 10) {
					ctx.fillStyle = x % 20 === 0 ? '#7a4d28' : '#83532b';
					ctx.fillRect(x, 0, 4, size);
				}
				ctx.fillStyle = 'rgba(32, 18, 10, 0.35)';
				ctx.fillRect(0, size * 0.48, size, 2);
				ctx.strokeStyle = 'rgba(255, 230, 190, 0.22)';
				ctx.lineWidth = 2;
				ctx.strokeRect(3, 3, size - 6, size - 6);
				ctx.fillStyle = '#d6b36a';
				ctx.fillRect(size * 0.72, size * 0.56, 4, 6);
				ctx.fillStyle = '#3b2818';
				ctx.fillRect(size * 0.72 + 1, size * 0.56 + 2, 2, 2);
			});
			const coalOreTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#7d8081';
				ctx.fillRect(0, 0, size, size);
				for (let i = 0; i < 260; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const tone = 92 + Math.floor(Math.random() * 80);
					ctx.fillStyle = `rgb(${tone}, ${tone}, ${tone})`;
					ctx.fillRect(x, y, 2, 2);
				}
				for (let i = 0; i < 90; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					ctx.fillStyle = i % 3 === 0 ? '#0b0f12' : '#1a232a';
					ctx.fillRect(x, y, 2, 2);
				}
			});
			const ironOreTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#7d8081';
				ctx.fillRect(0, 0, size, size);
				for (let i = 0; i < 260; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const tone = 92 + Math.floor(Math.random() * 80);
					ctx.fillStyle = `rgb(${tone}, ${tone}, ${tone})`;
					ctx.fillRect(x, y, 2, 2);
				}
				for (let i = 0; i < 85; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const tone = 165 + Math.floor(Math.random() * 60);
					ctx.fillStyle = `rgb(${tone}, ${tone - 35}, ${tone - 55})`;
					ctx.fillRect(x, y, 2, 2);
				}
			});
			const mossyCobbleTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#6f7375';
				ctx.fillRect(0, 0, size, size);
				for (let i = 0; i < 260; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const tone = 85 + Math.floor(Math.random() * 90);
					ctx.fillStyle = `rgb(${tone}, ${tone}, ${tone})`;
					ctx.fillRect(x, y, 2, 2);
				}
				for (let i = 0; i < 140; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const g = 90 + Math.floor(Math.random() * 80);
					ctx.fillStyle = `rgba(45, ${g}, 55, ${0.35 + Math.random() * 0.35})`;
					ctx.fillRect(x, y, 3, 3);
				}
				for (let i = 0; i < 120; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					ctx.clearRect(x, y, 1, 1);
				}
			});
			const clayTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#8098a8';
				ctx.fillRect(0, 0, size, size);
				for (let i = 0; i < 240; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const c = 120 + Math.floor(Math.random() * 70);
					ctx.fillStyle = `rgb(${c - 12}, ${c + 10}, ${c + 25})`;
					ctx.fillRect(x, y, 2, 2);
				}
				for (let i = 0; i < 110; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					ctx.fillStyle = 'rgba(35, 45, 55, 0.18)';
					ctx.fillRect(x, y, 1, 1);
				}
			});
			const snowTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#f7fbff';
				ctx.fillRect(0, 0, size, size);
				for (let i = 0; i < 240; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const b = 210 + Math.floor(Math.random() * 40);
					ctx.fillStyle = `rgba(${b}, ${b + 10}, 255, ${0.22 + Math.random() * 0.22})`;
					ctx.fillRect(x, y, 2, 2);
				}
				for (let i = 0; i < 160; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					ctx.fillStyle = 'rgba(130, 150, 175, 0.12)';
					ctx.fillRect(x, y, 1, 1);
				}
			});
			const iceTex = createCanvasTexture((ctx, size) => {
				ctx.clearRect(0, 0, size, size);
				ctx.fillStyle = 'rgba(160, 220, 255, 0.25)';
				ctx.fillRect(0, 0, size, size);
				ctx.strokeStyle = 'rgba(220, 250, 255, 0.4)';
				ctx.lineWidth = 2;
				for (let i = 0; i < 10; i += 1) {
					ctx.beginPath();
					ctx.moveTo(Math.random() * size, Math.random() * size);
					ctx.lineTo(Math.random() * size, Math.random() * size);
					ctx.stroke();
				}
				for (let i = 0; i < 100; i += 1) {
					ctx.fillStyle = `rgba(235, 255, 255, ${0.05 + Math.random() * 0.12})`;
					ctx.fillRect(Math.floor(Math.random() * size), Math.floor(Math.random() * size), 1, 1);
				}
			});
			const netherrackTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#5a1e1f';
				ctx.fillRect(0, 0, size, size);
				for (let i = 0; i < 420; i += 1) {
					const x = Math.floor(Math.random() * size);
					const y = Math.floor(Math.random() * size);
					const hot = Math.random();
					const color =
						hot > 0.82
							? 'rgba(154, 54, 40, 0.9)'
							: hot > 0.52
								? 'rgba(116, 38, 34, 0.85)'
								: 'rgba(72, 24, 26, 0.82)';
					ctx.fillStyle = color;
					ctx.fillRect(x, y, hot > 0.7 ? 2 : 1, 1);
				}
				for (let i = 0; i < 90; i += 1) {
					ctx.fillStyle = 'rgba(15, 6, 7, 0.28)';
					ctx.fillRect(Math.floor(Math.random() * size), Math.floor(Math.random() * size), 2, 2);
				}
			});
			const tntTopTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#c42d2d';
				ctx.fillRect(0, 0, size, size);
				ctx.fillStyle = '#f2e6d8';
				ctx.fillRect(0, size * 0.35, size, size * 0.3);
				ctx.fillStyle = '#111';
				ctx.font = `bold ${Math.floor(size * 0.3)}px sans-serif`;
				ctx.textAlign = 'center';
				ctx.textBaseline = 'middle';
				ctx.fillText('TNT', size / 2, size / 2);
			});
			const tntSideTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#b32727';
				ctx.fillRect(0, 0, size, size);
				ctx.fillStyle = '#f2e6d8';
				ctx.fillRect(0, size * 0.4, size, size * 0.2);
				ctx.fillStyle = '#111';
				ctx.font = `bold ${Math.floor(size * 0.22)}px sans-serif`;
				ctx.textAlign = 'center';
				ctx.textBaseline = 'middle';
				ctx.fillText('TNT', size / 2, size * 0.5);
			});
			const tntBottomTex = createCanvasTexture((ctx, size) => {
				ctx.fillStyle = '#7a1e1e';
				ctx.fillRect(0, 0, size, size);
			});

			const grassTopMat = new THREE.MeshStandardMaterial({ map: grassTopTex, roughness: 0.95 });
			const grassSideMat = new THREE.MeshStandardMaterial({ map: grassSideTex, roughness: 0.95 });
			const dirtMat = new THREE.MeshStandardMaterial({ map: dirtTex, roughness: 1 });
			const stoneMat = new THREE.MeshStandardMaterial({ map: stoneTex, roughness: 1 });
			const cobbleMat = new THREE.MeshStandardMaterial({ map: cobbleTex, roughness: 1 });
			const woodPlankMat = new THREE.MeshStandardMaterial({ map: woodPlankTex, roughness: 0.9 });
			const redWoodPlankMat = new THREE.MeshStandardMaterial({ map: redWoodPlankTex, roughness: 0.9 });
			const sandstoneMat = new THREE.MeshStandardMaterial({ map: sandstoneTex, roughness: 0.95 });
			const obsidianMat = new THREE.MeshStandardMaterial({
				map: obsidianTex,
				roughness: 0.45,
				metalness: 0.12
			});
			const portalFrameMat = new THREE.MeshStandardMaterial({
				map: obsidianTex,
				roughness: 0.35,
				metalness: 0.2,
				emissive: new THREE.Color(0x130a1b),
				emissiveIntensity: 0.7
			});
			const marsSandMat = new THREE.MeshStandardMaterial({ map: marsSandTex, roughness: 1 });
			const marsRockMat = new THREE.MeshStandardMaterial({ map: marsRockTex, roughness: 0.95 });
			const moonDustMat = new THREE.MeshStandardMaterial({ map: moonDustTex, roughness: 1 });
			const sandMat = new THREE.MeshStandardMaterial({ map: sandTex, roughness: 1 });
			const gravelMat = new THREE.MeshStandardMaterial({ map: gravelTex, roughness: 1 });
			const waterMat = new THREE.MeshStandardMaterial({
				map: waterTex,
				roughness: 0.15,
				metalness: 0.05,
				transparent: true,
				opacity: 0.62,
				depthWrite: false
			});
			const lavaMat = new THREE.MeshStandardMaterial({
				map: lavaTex,
				roughness: 0.7,
				metalness: 0.05,
				emissive: new THREE.Color('#ff5a1f'),
				emissiveIntensity: 0.85
			});
			const logTopMat = new THREE.MeshStandardMaterial({ map: logTopTex, roughness: 0.95 });
			const logSideMat = new THREE.MeshStandardMaterial({ map: logSideTex, roughness: 0.95 });
			const leavesMat = new THREE.MeshStandardMaterial({
				map: leavesTex,
				transparent: true,
				alphaTest: 0.2,
				roughness: 1
			});
			const glassMat = new THREE.MeshStandardMaterial({
				map: glassTex,
				transparent: true,
				opacity: 0.42,
				roughness: 0.05,
				metalness: 0.05,
				depthWrite: false
			});
			const brickMat = new THREE.MeshStandardMaterial({ map: brickTex, roughness: 0.95 });
			const doorMat = new THREE.MeshStandardMaterial({ map: doorTex, roughness: 0.9 });
			const coalOreMat = new THREE.MeshStandardMaterial({ map: coalOreTex, roughness: 1 });
			const ironOreMat = new THREE.MeshStandardMaterial({ map: ironOreTex, roughness: 1 });
			const mossyCobbleMat = new THREE.MeshStandardMaterial({ map: mossyCobbleTex, roughness: 1 });
			const clayMat = new THREE.MeshStandardMaterial({ map: clayTex, roughness: 1 });
			const snowMat = new THREE.MeshStandardMaterial({ map: snowTex, roughness: 0.95 });
			const iceMat = new THREE.MeshStandardMaterial({
				map: iceTex,
				transparent: true,
				opacity: 0.58,
				roughness: 0.1,
				metalness: 0.05,
				depthWrite: false
			});
			const netherrackMat = new THREE.MeshStandardMaterial({ map: netherrackTex, roughness: 1 });
			const tntTopMat = new THREE.MeshStandardMaterial({ map: tntTopTex, roughness: 0.85 });
			const tntSideMat = new THREE.MeshStandardMaterial({ map: tntSideTex, roughness: 0.85 });
			const tntBottomMat = new THREE.MeshStandardMaterial({ map: tntBottomTex, roughness: 0.9 });
			const particleMat = new THREE.MeshStandardMaterial({ color: 0xffc06b, roughness: 0.6 });
			const portalParticleMat = new THREE.MeshStandardMaterial({
				color: 0xffffff,
				emissive: 0xffffff,
				emissiveIntensity: 0.85,
				transparent: true,
				opacity: 0.9,
				roughness: 0.25
			});

			const blockGeo = new THREE.BoxGeometry(1, 1, 1);
			const fallingBlockGeo = new THREE.BoxGeometry(0.5, 0.5, 0.5);
			const particleGeo = new THREE.BoxGeometry(0.08, 0.08, 0.08);
			const portalParticleGeo = new THREE.IcosahedronGeometry(0.07, 0);
			const portalFrameGeo = new THREE.TorusGeometry(1, 0.12, 12, 40);
			const portalCoreGeo = new THREE.CircleGeometry(0.88, 32);
			const uniformMats = (mat: THREE.MeshStandardMaterial) => [mat, mat, mat, mat, mat, mat];
			const grassMats = [grassSideMat, grassSideMat, grassTopMat, dirtMat, grassSideMat, grassSideMat];
			const dirtMats = uniformMats(dirtMat);
			const stoneMats = uniformMats(stoneMat);
			const cobbleMats = uniformMats(cobbleMat);
			const woodMats = uniformMats(woodPlankMat);
			const redWoodMats = uniformMats(redWoodPlankMat);
			const sandstoneMats = uniformMats(sandstoneMat);
			const obsidianMats = uniformMats(obsidianMat);
			const marsSandMats = uniformMats(marsSandMat);
			const marsRockMats = uniformMats(marsRockMat);
			const moonDustMats = uniformMats(moonDustMat);
			const sandMats = uniformMats(sandMat);
			const gravelMats = uniformMats(gravelMat);
			const waterMats = uniformMats(waterMat);
			const lavaMats = uniformMats(lavaMat);
			const logMats = [logSideMat, logSideMat, logTopMat, logTopMat, logSideMat, logSideMat];
			const leavesMats = uniformMats(leavesMat);
			const glassMats = uniformMats(glassMat);
			const brickMats = uniformMats(brickMat);
			const doorMats = uniformMats(doorMat);
			const coalOreMats = uniformMats(coalOreMat);
			const ironOreMats = uniformMats(ironOreMat);
			const mossyCobbleMats = uniformMats(mossyCobbleMat);
			const clayMats = uniformMats(clayMat);
			const snowMats = uniformMats(snowMat);
			const iceMats = uniformMats(iceMat);
			const netherrackMats = uniformMats(netherrackMat);
			const tntMats = [tntSideMat, tntSideMat, tntTopMat, tntBottomMat, tntSideMat, tntSideMat];
			const blockMats = {
				grass: grassMats,
				dirt: dirtMats,
				stone: stoneMats,
				cobble: cobbleMats,
				wood: woodMats,
				redwood: redWoodMats,
				sandstone: sandstoneMats,
				obsidian: obsidianMats,
				marsSand: marsSandMats,
				marsRock: marsRockMats,
				moonDust: moonDustMats,
				sand: sandMats,
				gravel: gravelMats,
				water: waterMats,
				lava: lavaMats,
				log: logMats,
				leaves: leavesMats,
				glass: glassMats,
				brick: brickMats,
				door: doorMats,
				coalOre: coalOreMats,
				ironOre: ironOreMats,
				mossyCobble: mossyCobbleMats,
				clay: clayMats,
				snow: snowMats,
				ice: iceMats,
				netherrack: netherrackMats
			} as const;

			type BlockType = keyof typeof blockMats;

			type WorldDefinition = {
				id: 'earth' | 'mars' | 'moon';
				name: string;
				seed: number;
				skyColor: string;
				fogColor: string;
				fogNear: number;
				fogFar: number;
				gravity: number;
				music: { url: string; volume: number };
				portalColor: string;
				height: {
					base: number;
					amplitude: number;
					ridgeAmp: number;
					min: number;
					max: number;
					scale: number;
					ridgeScale: number;
					riverScale: number;
					riverWidth: number;
					riverDepth: number;
				};
				fluids: {
					waterLevel: number;
					lavaLevel: number;
					lavaScale: number;
					lavaThreshold: number;
					lavaCarveDepth: number;
				};
				vegetation: {
					treeDensity: number;
					treeScale: number;
					treeMinHeight: number;
					treeMaxHeight: number;
				};
				structures: {
					village: boolean;
					houseCount: number;
				};
				palette: {
					top: BlockType;
					sub: BlockType;
					deep: BlockType;
					topVariants?: BlockType[];
					deepVariants?: BlockType[];
					variantScale: number;
				};
				light: {
					hemiSky: number;
					hemiGround: number;
					hemiIntensity: number;
					dirColor: number;
					dirIntensity: number;
					dirPos: [number, number, number];
				};
			};

			const worlds: WorldDefinition[] = [
				{
					id: 'earth',
					name: 'Verdant Expanse',
					seed: 142857,
					skyColor: '#7ab3ff',
					fogColor: '#c7e4ff',
					fogNear: 38,
					fogFar: 140,
					gravity: -18,
					music: { url: '/audio/forest_ambience.mp3', volume: 0.48 },
					portalColor: '#6ef2c5',
					height: {
						base: 6.4,
						amplitude: 7.6,
						ridgeAmp: 2.4,
						min: 2,
						max: 19,
						scale: 0.055,
						ridgeScale: 0.19,
						riverScale: 0.03,
						riverWidth: 0.22,
						riverDepth: 4
					},
					fluids: {
						waterLevel: 7,
						lavaLevel: 4,
						lavaScale: 0.045,
						lavaThreshold: 0.89,
						lavaCarveDepth: 3
					},
					vegetation: {
						treeDensity: 0.055,
						treeScale: 0.09,
						treeMinHeight: 4,
						treeMaxHeight: 6
					},
					structures: {
						village: true,
						houseCount: 7
					},
					palette: {
						top: 'grass',
						sub: 'dirt',
						deep: 'stone',
						topVariants: ['cobble', 'wood', 'redwood', 'gravel'],
						deepVariants: ['cobble', 'coalOre', 'ironOre', 'obsidian'],
						variantScale: 0.18
					},
					light: {
						hemiSky: 0xeef6ff,
						hemiGround: 0x3f5b68,
						hemiIntensity: 0.95,
						dirColor: 0xffffff,
						dirIntensity: 1.15,
						dirPos: [7, 12, 4]
					}
				},
				{
					id: 'mars',
					name: 'Mars Rust Dunes',
					seed: 917331,
					skyColor: '#c86a48',
					fogColor: '#a84d37',
					fogNear: 32,
					fogFar: 125,
					gravity: -15,
					music: { url: '/audio/desert_travel.ogg', volume: 0.42 },
					portalColor: '#ff884c',
					height: {
						base: 5.1,
						amplitude: 4.4,
						ridgeAmp: 1.35,
						min: 2,
						max: 14,
						scale: 0.065,
						ridgeScale: 0.18,
						riverScale: 0.03,
						riverWidth: 0.26,
						riverDepth: 2
					},
					fluids: {
						waterLevel: 0,
						lavaLevel: 6,
						lavaScale: 0.04,
						lavaThreshold: 0.86,
						lavaCarveDepth: 4
					},
					vegetation: {
						treeDensity: 0,
						treeScale: 0,
						treeMinHeight: 0,
						treeMaxHeight: 0
					},
					structures: {
						village: true,
						houseCount: 5
					},
					palette: {
						top: 'marsSand',
						sub: 'sandstone',
						deep: 'marsRock',
						topVariants: ['sandstone', 'marsRock', 'gravel'],
						deepVariants: ['obsidian', 'marsRock', 'ironOre'],
						variantScale: 0.2
					},
					light: {
						hemiSky: 0xffd0b3,
						hemiGround: 0x5b2d25,
						hemiIntensity: 0.9,
						dirColor: 0xffc09a,
						dirIntensity: 1.05,
						dirPos: [8, 10, 3]
					}
				},
				{
					id: 'moon',
					name: 'Moon Dust Sea',
					seed: 424242,
					skyColor: '#1a2032',
					fogColor: '#111826',
					fogNear: 30,
					fogFar: 118,
					gravity: -9,
					music: { url: '/audio/outer_space.mp3', volume: 0.38 },
					portalColor: '#7bd1ff',
					height: {
						base: 4.8,
						amplitude: 4.2,
						ridgeAmp: 2.25,
						min: 2,
						max: 15,
						scale: 0.07,
						ridgeScale: 0.22,
						riverScale: 0.032,
						riverWidth: 0.25,
						riverDepth: 2
					},
					fluids: {
						waterLevel: 0,
						lavaLevel: 0,
						lavaScale: 0,
						lavaThreshold: 1,
						lavaCarveDepth: 0
					},
					vegetation: {
						treeDensity: 0,
						treeScale: 0,
						treeMinHeight: 0,
						treeMaxHeight: 0
					},
					structures: {
						village: true,
						houseCount: 4
					},
					palette: {
						top: 'moonDust',
						sub: 'moonDust',
						deep: 'obsidian',
						topVariants: ['cobble', 'gravel'],
						deepVariants: ['moonDust', 'obsidian', 'ironOre'],
						variantScale: 0.22
					},
					light: {
						hemiSky: 0xc9d8ff,
						hemiGround: 0x1a2235,
						hemiIntensity: 0.78,
						dirColor: 0xbad6ff,
						dirIntensity: 0.9,
						dirPos: [-4, 9, -6]
					}
				}
			];

				const worldById = new Map(worlds.map((world) => [world.id, world]));
				const spawnAnchors = {
					earth: { x: 28, z: 10 },
					mars: { x: 18, z: 14 },
					moon: { x: 14, z: 14 }
				} as const;
				let currentWorld = worlds[0];
				let gravity = currentWorld.gravity;
				worldLabel = currentWorld.name;

			const world = new RAPIER.World({ x: 0, y: gravity, z: 0 });

			const chunkSize = 16;
			const chunkRadius = 3;
			const cameraOccluders: THREE.Object3D[] = [];
			const blockTypeKeys = Object.keys(blockMats) as BlockType[];
			const chunkMatrix = new THREE.Matrix4();

			type Chunk = {
				key: string;
				x: number;
				z: number;
				meshes: THREE.InstancedMesh[];
				body: RAPIER.RigidBody;
			};

			const chunks = new Map<string, Chunk>();

			const hash2D = (x: number, z: number, seed: number) => {
				let h = Math.imul(x, 374761393) ^ Math.imul(z, 668265263) ^ seed;
				h = (h ^ (h >> 13)) * 1274126177;
				return ((h ^ (h >> 16)) >>> 0) / 4294967295;
			};

			const fade = (t: number) => t * t * (3 - 2 * t);
			const lerp = (a: number, b: number, t: number) => a + (b - a) * t;

			const valueNoise = (x: number, z: number, seed: number) => {
				const x0 = Math.floor(x);
				const z0 = Math.floor(z);
				const x1 = x0 + 1;
				const z1 = z0 + 1;
				const sx = fade(x - x0);
				const sz = fade(z - z0);
				const n00 = hash2D(x0, z0, seed);
				const n10 = hash2D(x1, z0, seed);
				const n01 = hash2D(x0, z1, seed);
				const n11 = hash2D(x1, z1, seed);
				const ix0 = lerp(n00, n10, sx);
				const ix1 = lerp(n01, n11, sx);
				return lerp(ix0, ix1, sz);
			};

			const fractalNoise01 = (x: number, z: number, seed: number) => {
				let total = 0;
				let amplitude = 1;
				let frequency = 1;
				let max = 0;
				for (let i = 0; i < 4; i += 1) {
					total += valueNoise(x * frequency, z * frequency, seed + i * 13) * amplitude;
					max += amplitude;
					amplitude *= 0.5;
					frequency *= 2;
				}
				return total / max;
			};

			const fractalNoise = (x: number, z: number, seed: number) => fractalNoise01(x, z, seed) * 2 - 1;

			const smoothstep = (edge0: number, edge1: number, x: number) => {
				const t = THREE.MathUtils.clamp((x - edge0) / (edge1 - edge0), 0, 1);
				return t * t * (3 - 2 * t);
			};

			const getLavaStrength = (x: number, z: number, worldDef: WorldDefinition) => {
				if (worldDef.fluids.lavaLevel <= 0 || worldDef.fluids.lavaScale <= 0) {
					return 0;
				}
				const field = valueNoise(
					x * worldDef.fluids.lavaScale,
					z * worldDef.fluids.lavaScale,
					worldDef.seed + 7077
				);
				if (field < worldDef.fluids.lavaThreshold) {
					return 0;
				}
				return (field - worldDef.fluids.lavaThreshold) / (1 - worldDef.fluids.lavaThreshold);
			};

			type BiomeId =
				| 'ocean'
				| 'beach'
				| 'plains'
				| 'forest'
				| 'swamp'
				| 'desert'
				| 'mountains'
				| 'volcanic'
				| 'craters';

			type BiomeContext = {
				id: BiomeId;
				name: string;
				heightBaseAdd: number;
				heightAmpMul: number;
				heightRidgeMul: number;
				palette: {
					top: BlockType;
					sub: BlockType;
					deep: BlockType;
					topVariants?: BlockType[];
					deepVariants?: BlockType[];
				};
				vegetation: {
					treeDensity: number;
					treeScale: number;
					treeMinHeight: number;
					treeMaxHeight: number;
				};
				structures: {
					allowVillage: boolean;
				};
			};

			const domainWarp = (x: number, z: number, seed: number) => {
				const warpScale = 0.035;
				const wx = fractalNoise(x * warpScale, z * warpScale, seed + 1337);
				const wz = fractalNoise((x + 71.3) * warpScale, (z - 12.9) * warpScale, seed + 1777);
				return {
					x: x + wx * 14,
					z: z + wz * 14,
					warpX: wx,
					warpZ: wz
				};
			};

			const getClimate = (x: number, z: number, worldDef: WorldDefinition) => {
				const warped = domainWarp(x, z, worldDef.seed);
				const dist = worldDef.id === 'earth' ? Math.hypot(x, z) : 0;
				const baseScale = 0.004;
				const localScale = 0.018;
				const temperature =
					fractalNoise(warped.x * baseScale, warped.z * baseScale, worldDef.seed + 11) * 0.72 +
					fractalNoise(warped.x * localScale, warped.z * localScale, worldDef.seed + 12) * 0.28;
				const humidity =
					fractalNoise(warped.x * baseScale, warped.z * baseScale, worldDef.seed + 21) * 0.72 +
					fractalNoise(warped.x * localScale, warped.z * localScale, worldDef.seed + 22) * 0.28;
				const continentalDetail = fractalNoise(warped.x * 0.006, warped.z * 0.006, worldDef.seed + 31);
				const continentalLarge = fractalNoise(warped.x * 0.0018, warped.z * 0.0018, worldDef.seed + 33);
				let continentalness = continentalDetail * 0.72 + continentalLarge * 0.28;
				if (worldDef.id === 'earth') {
					const spawnIsland = 1 - smoothstep(0, 95, dist);
					continentalness += spawnIsland * 0.65;
				}
				continentalness = THREE.MathUtils.clamp(continentalness, -1, 1);
				let erosion = fractalNoise(warped.x * 0.01, warped.z * 0.01, worldDef.seed + 41);
				let weirdness = fractalNoise(warped.x * 0.012, warped.z * 0.012, worldDef.seed + 51);
				if (worldDef.id === 'earth') {
					const spawnCalm = 1 - smoothstep(0, 75, dist);
					weirdness = lerp(weirdness, 0, spawnCalm * 0.85);
					erosion = lerp(erosion, 0.22, spawnCalm * 0.7);
				}
				erosion = THREE.MathUtils.clamp(erosion, -1, 1);
				weirdness = THREE.MathUtils.clamp(weirdness, -1, 1);

				const mineral1 = valueNoise(warped.x * 0.02, warped.z * 0.02, worldDef.seed + 610);
				const mineral2 = valueNoise(warped.x * 0.035, warped.z * 0.035, worldDef.seed + 915);
				const mineralArc = 2 * (0.5 - Math.abs(0.5 - mineral1)) * mineral2;

				return {
					temperature: THREE.MathUtils.clamp(temperature, -1, 1),
					humidity: THREE.MathUtils.clamp(humidity, -1, 1),
					continentalness: THREE.MathUtils.clamp(continentalness, -1, 1),
					erosion: THREE.MathUtils.clamp(erosion, -1, 1),
					weirdness: THREE.MathUtils.clamp(weirdness, -1, 1),
					mineralArc,
					warped
				};
			};

			const getBiomeFromClimate = (
				x: number,
				z: number,
				worldDef: WorldDefinition,
				climate: ReturnType<typeof getClimate>
			): BiomeContext => {
				const dist = Math.hypot(x, z);

				if (worldDef.id === 'moon') {
					const craterField = valueNoise(climate.warped.x * 0.035, climate.warped.z * 0.035, worldDef.seed + 880);
					const isCrater = craterField > 0.72 && dist > 18;
					return isCrater
						? {
								id: 'craters',
								name: 'Craters',
								heightBaseAdd: -1.1,
								heightAmpMul: 1.2,
								heightRidgeMul: 1.25,
								palette: {
									top: 'moonDust',
									sub: 'moonDust',
									deep: 'obsidian',
									topVariants: ['gravel', 'moonDust'],
									deepVariants: ['obsidian', 'ironOre']
								},
								vegetation: { treeDensity: 0, treeScale: 0, treeMinHeight: 0, treeMaxHeight: 0 },
								structures: { allowVillage: true }
							}
						: {
								id: 'plains',
								name: 'Dust Flats',
								heightBaseAdd: 0,
								heightAmpMul: 1,
								heightRidgeMul: 1,
								palette: {
									top: 'moonDust',
									sub: 'moonDust',
									deep: 'obsidian',
									topVariants: ['gravel', 'cobble'],
									deepVariants: ['obsidian', 'ironOre']
								},
								vegetation: { treeDensity: 0, treeScale: 0, treeMinHeight: 0, treeMaxHeight: 0 },
								structures: { allowVillage: true }
							};
				}

				if (worldDef.id === 'mars') {
					const volcanicBand = climate.mineralArc > 0.86 && dist > 20;
					if (volcanicBand) {
						return {
							id: 'volcanic',
							name: 'Basalt Wastes',
							heightBaseAdd: 1.2,
							heightAmpMul: 1.35,
							heightRidgeMul: 1.55,
							palette: {
								top: 'marsRock',
								sub: 'marsRock',
								deep: 'obsidian',
								topVariants: ['obsidian', 'netherrack', 'cobble', 'gravel'],
								deepVariants: ['obsidian', 'netherrack', 'ironOre']
							},
							vegetation: { treeDensity: 0, treeScale: 0, treeMinHeight: 0, treeMaxHeight: 0 },
							structures: { allowVillage: true }
						};
					}

					const dunes = climate.erosion > 0.25;
					return dunes
						? {
								id: 'desert',
								name: 'Dunes',
								heightBaseAdd: 0.3,
								heightAmpMul: 1.1,
								heightRidgeMul: 0.85,
								palette: {
									top: 'marsSand',
									sub: 'sandstone',
									deep: 'marsRock',
									topVariants: ['sandstone', 'marsRock', 'gravel'],
									deepVariants: ['marsRock', 'obsidian']
								},
								vegetation: { treeDensity: 0, treeScale: 0, treeMinHeight: 0, treeMaxHeight: 0 },
								structures: { allowVillage: true }
							}
						: {
								id: 'mountains',
								name: 'Canyons',
								heightBaseAdd: 0.8,
								heightAmpMul: 1.35,
								heightRidgeMul: 1.25,
								palette: {
									top: 'marsRock',
									sub: 'marsRock',
									deep: 'obsidian',
									topVariants: ['marsSand', 'gravel'],
									deepVariants: ['marsRock', 'obsidian', 'ironOre']
								},
								vegetation: { treeDensity: 0, treeScale: 0, treeMinHeight: 0, treeMaxHeight: 0 },
								structures: { allowVillage: true }
							};
				}

				const isOcean = climate.continentalness < -0.45;
				if (isOcean) {
					return {
						id: 'ocean',
						name: 'Ocean',
						heightBaseAdd: -5.2,
						heightAmpMul: 0.55,
						heightRidgeMul: 0.4,
						palette: {
							top: 'sand',
							sub: 'sand',
							deep: 'stone',
							topVariants: ['gravel', 'clay'],
							deepVariants: ['stone', 'coalOre']
						},
						vegetation: { treeDensity: 0, treeScale: 0, treeMinHeight: 0, treeMaxHeight: 0 },
						structures: { allowVillage: false }
					};
				}

				const isCoast = climate.continentalness < -0.19;
				if (isCoast) {
					return {
						id: 'beach',
						name: 'Beach',
						heightBaseAdd: -1.3,
						heightAmpMul: 0.8,
						heightRidgeMul: 0.6,
						palette: {
							top: 'sand',
							sub: 'sand',
							deep: 'stone',
							topVariants: ['gravel', 'clay'],
							deepVariants: ['stone', 'coalOre']
						},
						vegetation: { treeDensity: 0.01, treeScale: 0.12, treeMinHeight: 3, treeMaxHeight: 4 },
						structures: { allowVillage: true }
					};
				}

				const volcanicBand = climate.mineralArc > 0.88 && dist > 26;
				if (volcanicBand) {
					return {
						id: 'volcanic',
						name: 'Volcanic Ridge',
						heightBaseAdd: 2.4,
						heightAmpMul: 1.35,
						heightRidgeMul: 1.85,
						palette: {
							top: 'cobble',
							sub: 'stone',
							deep: 'obsidian',
							topVariants: ['obsidian', 'netherrack', 'gravel'],
							deepVariants: ['obsidian', 'netherrack', 'ironOre']
						},
						vegetation: { treeDensity: 0, treeScale: 0, treeMinHeight: 0, treeMaxHeight: 0 },
						structures: { allowVillage: false }
					};
				}

				const isMountain = climate.erosion < -0.32 || Math.abs(climate.weirdness) > 0.65;
				if (isMountain) {
					return {
						id: 'mountains',
						name: 'Mountains',
						heightBaseAdd: 1.8,
						heightAmpMul: 1.45,
						heightRidgeMul: 1.35,
						palette: {
							top: 'gravel',
							sub: 'stone',
							deep: 'stone',
							topVariants: ['stone', 'cobble', 'snow', 'ironOre'],
							deepVariants: ['coalOre', 'ironOre', 'obsidian']
						},
						vegetation: { treeDensity: 0.015, treeScale: 0.08, treeMinHeight: 4, treeMaxHeight: 6 },
						structures: { allowVillage: true }
					};
				}

				const hot = climate.temperature > 0.32;
				const cold = climate.temperature < -0.35;
				const wet = climate.humidity > 0.28;
				const dry = climate.humidity < -0.18;

				if (hot && dry) {
					return {
						id: 'desert',
						name: 'Desert',
						heightBaseAdd: 0.25,
						heightAmpMul: 0.95,
						heightRidgeMul: 0.75,
						palette: {
							top: 'sand',
							sub: 'sand',
							deep: 'sandstone',
							topVariants: ['sandstone', 'clay', 'gravel'],
							deepVariants: ['sandstone', 'stone', 'coalOre']
						},
						vegetation: { treeDensity: 0, treeScale: 0, treeMinHeight: 0, treeMaxHeight: 0 },
						structures: { allowVillage: true }
					};
				}

				if (wet && !cold) {
					const swampy = climate.continentalness < 0.18 && climate.erosion > 0.15;
					if (swampy) {
						return {
							id: 'swamp',
							name: 'Swamp',
							heightBaseAdd: -0.7,
							heightAmpMul: 0.85,
							heightRidgeMul: 0.65,
							palette: {
								top: 'grass',
								sub: 'dirt',
								deep: 'stone',
								topVariants: ['dirt', 'clay', 'mossyCobble', 'gravel'],
								deepVariants: ['stone', 'coalOre']
							},
							vegetation: {
								treeDensity: 0.09,
								treeScale: 0.12,
								treeMinHeight: 3,
								treeMaxHeight: 5
							},
							structures: { allowVillage: true }
						};
					}

					return {
						id: 'forest',
						name: 'Forest',
						heightBaseAdd: 0.35,
						heightAmpMul: 1.05,
						heightRidgeMul: 0.95,
						palette: {
							top: 'grass',
							sub: 'dirt',
							deep: 'stone',
							topVariants: ['dirt', 'mossyCobble', 'gravel'],
							deepVariants: ['stone', 'coalOre', 'ironOre']
						},
						vegetation: {
							treeDensity: 0.08,
							treeScale: 0.1,
							treeMinHeight: 4,
							treeMaxHeight: 7
						},
						structures: { allowVillage: true }
					};
				}

				if (cold && wet) {
					return {
						id: 'forest',
						name: 'Pine Woods',
						heightBaseAdd: 0.6,
						heightAmpMul: 1.1,
						heightRidgeMul: 1,
						palette: {
							top: 'grass',
							sub: 'dirt',
							deep: 'stone',
							topVariants: ['dirt', 'mossyCobble', 'gravel', 'snow'],
							deepVariants: ['stone', 'coalOre', 'ironOre']
						},
						vegetation: {
							treeDensity: 0.06,
							treeScale: 0.085,
							treeMinHeight: 5,
							treeMaxHeight: 7
						},
						structures: { allowVillage: true }
					};
				}

				return {
					id: 'plains',
					name: 'Plains',
					heightBaseAdd: 0,
					heightAmpMul: 1,
					heightRidgeMul: 0.9,
					palette: {
						top: 'grass',
						sub: 'dirt',
						deep: 'stone',
						topVariants: ['dirt', 'clay', 'gravel', 'cobble'],
						deepVariants: ['stone', 'coalOre', 'ironOre']
					},
					vegetation: {
						treeDensity: 0.03,
						treeScale: 0.09,
						treeMinHeight: 4,
						treeMaxHeight: 6
					},
					structures: { allowVillage: true }
				};
			};

			const getBiomeAt = (x: number, z: number, worldDef: WorldDefinition): BiomeContext => {
				const climate = getClimate(x, z, worldDef);
				return getBiomeFromClimate(x, z, worldDef, climate);
			};

			type TerrainInfo = {
				height: number;
				biome: BiomeContext;
				lavaStrength: number;
				riverStrength: number;
				warpedX: number;
				warpedZ: number;
				erosion: number;
				weirdness: number;
				continentalness: number;
				temperature: number;
				humidity: number;
			};

			const getTerrainInfo = (x: number, z: number, worldDef: WorldDefinition): TerrainInfo => {
				const climate = getClimate(x, z, worldDef);
				const biome = getBiomeFromClimate(x, z, worldDef, climate);
				const { x: warpedX, z: warpedZ } = climate.warped;
				const roughness = lerp(1.5, 0.6, (climate.erosion + 1) / 2);
				const continentLift = lerp(-2.8, 3.2, (climate.continentalness + 1) / 2);

				const detail = fractalNoise01(warpedX * worldDef.height.scale, warpedZ * worldDef.height.scale, worldDef.seed);
				const ridge = Math.abs(
					fractalNoise(warpedX * worldDef.height.ridgeScale, warpedZ * worldDef.height.ridgeScale, worldDef.seed + 91)
				);
				let height =
					worldDef.height.base +
					biome.heightBaseAdd +
					continentLift +
					detail * worldDef.height.amplitude * biome.heightAmpMul * roughness +
					ridge * worldDef.height.ridgeAmp * biome.heightRidgeMul * roughness;

				const riverField = Math.abs(
					fractalNoise(warpedX * worldDef.height.riverScale, warpedZ * worldDef.height.riverScale, worldDef.seed + 203)
				);
				const valleyFactor = 1 - smoothstep(0.15, 0.45, Math.abs(climate.weirdness));
				const riverWidth = worldDef.height.riverWidth * (biome.id === 'desert' ? 0.8 : 1);
				const riverStrength = (1 - smoothstep(0, riverWidth, riverField)) * (0.55 + valleyFactor * 0.7);
				height -= riverStrength * worldDef.height.riverDepth;

				let lavaStrength = getLavaStrength(x, z, worldDef);
				if (lavaStrength > 0) {
					const dist = Math.hypot(x, z);
					const spawnSafety = smoothstep(14, 40, dist);
					lavaStrength *= spawnSafety;
					const carve = Math.pow(lavaStrength, 1.6) * worldDef.fluids.lavaCarveDepth;
					height -= carve;
				}

				const rounded = THREE.MathUtils.clamp(Math.round(height), worldDef.height.min, worldDef.height.max);
				return {
					height: rounded,
					biome,
					lavaStrength,
					riverStrength,
					warpedX,
					warpedZ,
					erosion: climate.erosion,
					weirdness: climate.weirdness,
					continentalness: climate.continentalness,
					temperature: climate.temperature,
					humidity: climate.humidity
				};
			};

			const computeHeight = (x: number, z: number, worldDef: WorldDefinition) => {
				return getTerrainInfo(x, z, worldDef).height;
			};

			const pickVariant = (
				variants: BlockType[] | undefined,
				noise: number,
				fallback: BlockType
			) => {
				if (!variants || variants.length === 0) {
					return fallback;
				}
				if (noise < 0.65) {
					return fallback;
				}
				const idx = Math.min(
					Math.floor(((noise - 0.65) / 0.35) * variants.length),
					variants.length - 1
				);
				return variants[idx] ?? fallback;
			};

			const pickBlockType = (
				biome: BiomeContext,
				x: number,
				z: number,
				y: number,
				height: number,
				waterLevel: number,
				lavaStrength: number,
				riverStrength: number,
				temperature: number,
				humidity: number
			): BlockType => {
				const dryness = valueNoise(x * 0.018, z * 0.018, currentWorld.seed + 1009);
				const isUnderwater = waterLevel > 0 && height < waterLevel;
				const isBeach = waterLevel > 0 && height >= waterLevel && height <= waterLevel + 1;
				const isDesert = biome.id === 'desert' || (waterLevel > 0 && dryness > 0.78 && height > waterLevel + 1);
				const volcanic = lavaStrength > 0.35 || biome.id === 'volcanic';
				const inRiver = !isUnderwater && riverStrength > 0.55 && !volcanic;
				const snowy = temperature < -0.45 && !isUnderwater && !isDesert && !volcanic;

				const surfaceNoise = valueNoise(
					x * currentWorld.palette.variantScale,
					z * currentWorld.palette.variantScale,
					currentWorld.seed + 181
				);
				if (y === height - 1) {
					if (volcanic) {
						if (lavaStrength > 0.6 && surfaceNoise > 0.76) {
							return 'netherrack';
						}
						return surfaceNoise > 0.68 ? 'obsidian' : 'cobble';
					}
					if (snowy && (biome.id === 'mountains' || height > waterLevel + 6)) {
						return surfaceNoise > 0.82 ? 'ice' : 'snow';
					}
					if (inRiver) {
						if (humidity > 0.35 && surfaceNoise > 0.78) {
							return 'clay';
						}
						return dryness > 0.6 ? 'sand' : 'gravel';
					}
					if (isUnderwater) {
						return dryness > 0.7 ? 'gravel' : 'sand';
					}
					if (isBeach || isDesert) {
						return surfaceNoise > 0.84 ? 'clay' : 'sand';
					}
					if ((biome.id === 'forest' || biome.id === 'swamp') && humidity > 0.25 && surfaceNoise > 0.84) {
						return 'mossyCobble';
					}
					return pickVariant(biome.palette.topVariants, surfaceNoise, biome.palette.top);
				}
				if (y >= height - 3) {
					if (volcanic) {
						return 'stone';
					}
					if (snowy && y >= height - 2) {
						return 'snow';
					}
					if (inRiver) {
						return humidity > 0.35 && surfaceNoise > 0.76 ? 'clay' : dryness > 0.65 ? 'sand' : 'gravel';
					}
					if (isUnderwater || isDesert) {
						return 'sand';
					}
					return biome.palette.sub;
				}
				const deepNoise = valueNoise(
					x * currentWorld.palette.variantScale * 0.8,
					z * currentWorld.palette.variantScale * 0.8,
					currentWorld.seed + 419
				);
				if (volcanic && deepNoise > 0.72) {
					return 'obsidian';
				}
				return pickVariant(biome.palette.deepVariants, deepNoise, biome.palette.deep);
			};

			const getHeightAt = (x: number, z: number) => computeHeight(x, z, currentWorld);

				const isSolidBlockType = (type: BlockType) =>
					type !== 'water' && type !== 'lava' && type !== 'leaves' && type !== 'door';

				const findSafeSpawn = (worldDef: WorldDefinition) => {
					const waterLevel = worldDef.fluids.waterLevel;
					const start = spawnAnchors[worldDef.id];
					const avoidAabb = { minX: start.x - 2, maxX: start.x + 2, minZ: start.z - 2, maxZ: start.z + 2 };
					const maxRadius = 96;
					for (let r = 0; r <= maxRadius; r += 1) {
						for (let dx = -r; dx <= r; dx += 1) {
							for (let dz = -r; dz <= r; dz += 1) {
							if (Math.abs(dx) !== r && Math.abs(dz) !== r) {
								continue;
							}
							const x = start.x + dx;
							const z = start.z + dz;
							const info = getTerrainInfo(x, z, worldDef);
								if (x >= avoidAabb.minX && x <= avoidAabb.maxX && z >= avoidAabb.minZ && z <= avoidAabb.maxZ) {
									continue;
								}
								if (waterLevel > 0 && info.height < waterLevel) {
									continue;
								}
							if (waterLevel > 0 && info.riverStrength > 0.62) {
								continue;
							}
							if (info.lavaStrength > 0.22) {
								continue;
							}
							let minH = Number.POSITIVE_INFINITY;
							let maxH = Number.NEGATIVE_INFINITY;
							for (let ox = -1; ox <= 1; ox += 1) {
								for (let oz = -1; oz <= 1; oz += 1) {
									const h = computeHeight(x + ox, z + oz, worldDef);
									minH = Math.min(minH, h);
									maxH = Math.max(maxH, h);
								}
							}
							if (maxH - minH > 2) {
								continue;
							}
							return { x, z, y: info.height + 0.25 };
						}
					}
				}
				const fallback = getTerrainInfo(0, 0, worldDef);
				const surface = Math.max(fallback.height, worldDef.fluids.waterLevel, worldDef.fluids.lavaLevel);
				return { x: 0, z: 0, y: surface + 0.25 };
			};

			const buildChunk = (cx: number, cz: number) => {
				const key = `${cx},${cz}`;
				if (chunks.has(key)) {
					return;
				}
				const chunkMinX = cx * chunkSize;
				const chunkMinZ = cz * chunkSize;
				const positionsByType: Record<BlockType, number[]> = {} as Record<BlockType, number[]>;
				for (const type of blockTypeKeys) {
					positionsByType[type] = [];
				}

				const waterLevel = currentWorld.fluids.waterLevel;
				const lavaLevel = currentWorld.fluids.lavaLevel;
				const flatTarget = new Int16Array(chunkSize * chunkSize);
				flatTarget.fill(-1);
				const topOverride: (BlockType | null)[] = new Array(chunkSize * chunkSize).fill(null);
				const foundationStart = new Int16Array(chunkSize * chunkSize);
				foundationStart.fill(-1);
				const foundationMaterial: (BlockType | null)[] = new Array(chunkSize * chunkSize).fill(null);
				const noPlants = new Uint8Array(chunkSize * chunkSize);

				const solidBlockColliders: number[] = [];

				type HousePlan = {
					x: number;
					z: number;
					width: number;
					depth: number;
					wallHeight: number;
					roofHeight: number;
					facing: 0 | 1 | 2 | 3;
					style: 'oak' | 'spruce' | 'desert' | 'stone';
					baseY: number;
					doorX: number;
					doorZ: number;
				};

				const drawPath = (x0: number, z0: number, x1: number, z1: number, material: BlockType) => {
					let x = x0;
					let z = z0;
					const dx = Math.abs(x1 - x0);
					const dz = Math.abs(z1 - z0);
					const sx = x0 < x1 ? 1 : -1;
					const sz = z0 < z1 ? 1 : -1;
					let err = dx - dz;
					for (let i = 0; i < 420; i += 1) {
						if (x >= chunkMinX && x < chunkMinX + chunkSize && z >= chunkMinZ && z < chunkMinZ + chunkSize) {
							const ix = x - chunkMinX;
							const iz = z - chunkMinZ;
							const idx = ix * chunkSize + iz;
							topOverride[idx] = material;
							noPlants[idx] = 1;
						}
						if (x === x1 && z === z1) {
							break;
						}
						const e2 = err * 2;
						if (e2 > -dz) {
							err -= dz;
							x += sx;
						}
						if (e2 < dx) {
							err += dx;
							z += sz;
						}
					}
				};

				const emitSolidBlock = (x: number, y: number, z: number, type: BlockType, solid = true) => {
					if (x < chunkMinX || x >= chunkMinX + chunkSize || z < chunkMinZ || z >= chunkMinZ + chunkSize) {
						return;
					}
					positionsByType[type].push(x, y + 0.5, z);
					if (solid && isSolidBlockType(type)) {
						solidBlockColliders.push(x, y + 0.5, z);
					}
				};

				const emitHouse = (plan: HousePlan) => {
					const baseX = plan.x;
					const baseZ = plan.z;
					const houseSeed = currentWorld.seed + plan.x * 31 + plan.z * 17;
					const houseRand = (x: number, y: number, z: number, salt: number) =>
						hash2D(x + y * 11 + salt * 7, z - y * 13 - salt * 5, houseSeed + salt * 101);

					const floorMat: BlockType =
						plan.style === 'desert' ? 'sandstone' : plan.style === 'stone' ? 'cobble' : plan.style === 'spruce' ? 'redwood' : 'wood';
					const wallMat: BlockType =
						plan.style === 'desert' ? 'sandstone' : plan.style === 'stone' ? 'cobble' : 'wood';
					const beamMat: BlockType = plan.style === 'desert' ? 'sandstone' : 'log';
					const roofMat: BlockType = plan.style === 'desert' ? 'sandstone' : plan.style === 'stone' ? 'brick' : 'wood';
					const windowMat: BlockType = 'glass';
					const foundationMat: BlockType = plan.style === 'desert' ? 'sandstone' : 'cobble';

					for (let dx = 0; dx < plan.width; dx += 1) {
						for (let dz = 0; dz < plan.depth; dz += 1) {
							const x = baseX + dx;
							const z = baseZ + dz;
							if (x < chunkMinX || x >= chunkMinX + chunkSize || z < chunkMinZ || z >= chunkMinZ + chunkSize) {
								continue;
							}
							const idx = (x - chunkMinX) * chunkSize + (z - chunkMinZ);
							noPlants[idx] = 1;
							const targetSurface = plan.baseY;
							flatTarget[idx] = Math.max(flatTarget[idx], targetSurface);
							const surface = getHeightAt(x, z);
							if (surface < targetSurface) {
								foundationStart[idx] = foundationStart[idx] === -1 ? surface : Math.min(foundationStart[idx], surface);
								foundationMaterial[idx] = foundationMat;
							}
							topOverride[idx] = floorMat;
						}
					}

					const y0 = plan.baseY;
					const wallTop = y0 + plan.wallHeight;

					const isWall = (dx: number, dz: number) =>
						dx === 0 || dz === 0 || dx === plan.width - 1 || dz === plan.depth - 1;

					for (let dy = 0; dy < plan.wallHeight; dy += 1) {
						const y = y0 + dy;
						for (let dx = 0; dx < plan.width; dx += 1) {
							for (let dz = 0; dz < plan.depth; dz += 1) {
								if (!isWall(dx, dz)) continue;
								const x = baseX + dx;
								const z = baseZ + dz;

								const isDoor = x === plan.doorX && z === plan.doorZ;
								if (isDoor && dy < 2) {
									continue;
								}

								const isCorner = (dx === 0 || dx === plan.width - 1) && (dz === 0 || dz === plan.depth - 1);
								const onDoorWall =
									(plan.facing === 0 && dz === 0) ||
									(plan.facing === 2 && dz === plan.depth - 1) ||
									(plan.facing === 1 && dx === plan.width - 1) ||
									(plan.facing === 3 && dx === 0);
								const windowRow = dy === 1;
								const canWindow = windowRow && !isCorner && (dx === 0 || dz === 0 || dx === plan.width - 1 || dz === plan.depth - 1);
								const windowChance = plan.style === 'desert' ? 0.22 : 0.35;
								if (canWindow && (!onDoorWall || houseRand(x, y, z, 4) > 0.75) && houseRand(x, y, z, 2) < windowChance) {
									emitSolidBlock(x, y, z, windowMat, true);
									continue;
								}

								emitSolidBlock(x, y, z, isCorner ? beamMat : wallMat, true);
							}
						}
					}

					for (let dy = 0; dy < 2; dy += 1) {
						emitSolidBlock(plan.doorX, y0 + dy, plan.doorZ, 'door', false);
					}

					for (let i = 0; i < plan.roofHeight; i += 1) {
						const inset = i;
						const y = wallTop + i;
						for (let dx = inset; dx < plan.width - inset; dx += 1) {
							for (let dz = inset; dz < plan.depth - inset; dz += 1) {
								const edge =
									dx === inset ||
									dz === inset ||
									dx === plan.width - inset - 1 ||
									dz === plan.depth - inset - 1;
								if (!edge) continue;
								const x = baseX + dx;
								const z = baseZ + dz;
								emitSolidBlock(x, y, z, roofMat, true);
							}
						}
					}

					const chimneyChance = plan.style === 'stone' ? 0.65 : 0.28;
					if (houseRand(baseX, y0, baseZ, 9) < chimneyChance) {
						const cx = baseX + plan.width - 2;
						const cz = baseZ + 1;
						for (let dy = 0; dy < plan.wallHeight + plan.roofHeight; dy += 1) {
							emitSolidBlock(cx, wallTop - 1 + dy, cz, 'brick', true);
						}
						emitSolidBlock(cx, wallTop - 1, cz + 1, 'brick', true);
						emitSolidBlock(cx, wallTop - 1, cz - 1, 'brick', true);
					}
				};

				const emitVillageWell = (
					centerX: number,
					centerZ: number,
					centerHeight: number,
					centerBiome: BiomeContext
				) => {
					if (currentWorld.id === 'moon') {
						return;
					}
					let baseY = centerHeight;
					for (let dx = -2; dx <= 2; dx += 1) {
						for (let dz = -2; dz <= 2; dz += 1) {
							baseY = Math.max(baseY, getHeightAt(centerX + dx, centerZ + dz));
						}
					}

					const floorMat: BlockType =
						currentWorld.id === 'mars'
							? 'obsidian'
							: centerBiome.id === 'desert'
								? 'sandstone'
								: 'cobble';
					const pillarMat: BlockType =
						currentWorld.id === 'mars'
							? 'obsidian'
							: centerBiome.id === 'desert'
								? 'sandstone'
								: 'log';
					const roofMat: BlockType =
						currentWorld.id === 'mars'
							? 'brick'
							: centerBiome.id === 'desert'
								? 'sandstone'
								: 'wood';
					const fluidMat: BlockType = currentWorld.id === 'mars' ? 'lava' : 'water';

					for (let dx = -2; dx <= 2; dx += 1) {
						for (let dz = -2; dz <= 2; dz += 1) {
							const x = centerX + dx;
							const z = centerZ + dz;
							if (x < chunkMinX || x >= chunkMinX + chunkSize || z < chunkMinZ || z >= chunkMinZ + chunkSize) {
								continue;
							}
							const idx = (x - chunkMinX) * chunkSize + (z - chunkMinZ);
							noPlants[idx] = 1;
							flatTarget[idx] = Math.max(flatTarget[idx], baseY);
							topOverride[idx] = floorMat;
						}
					}

					for (let dx = -1; dx <= 1; dx += 1) {
						for (let dz = -1; dz <= 1; dz += 1) {
							const x = centerX + dx;
							const z = centerZ + dz;
							if (dx === 0 && dz === 0) {
								emitSolidBlock(x, baseY, z, fluidMat, false);
							} else {
								emitSolidBlock(x, baseY, z, floorMat, true);
							}
						}
					}

					const corners: Array<[number, number]> = [
						[-1, -1],
						[-1, 1],
						[1, -1],
						[1, 1]
					];
					for (const [dx, dz] of corners) {
						for (let dy = 1; dy <= 3; dy += 1) {
							emitSolidBlock(centerX + dx, baseY + dy, centerZ + dz, pillarMat, true);
						}
					}

					for (let dx = -1; dx <= 1; dx += 1) {
						for (let dz = -1; dz <= 1; dz += 1) {
							emitSolidBlock(centerX + dx, baseY + 4, centerZ + dz, roofMat, true);
						}
					}
				};

					const emitVillage = () => {
						if (!currentWorld.structures.village) {
							return;
						}
						const villageCellSize = 96;
					const minCellX = Math.floor((chunkMinX - 48) / villageCellSize);
					const maxCellX = Math.floor((chunkMinX + chunkSize + 48) / villageCellSize);
					const minCellZ = Math.floor((chunkMinZ - 48) / villageCellSize);
					const maxCellZ = Math.floor((chunkMinZ + chunkSize + 48) / villageCellSize);

						for (let vcx = minCellX; vcx <= maxCellX; vcx += 1) {
							for (let vcz = minCellZ; vcz <= maxCellZ; vcz += 1) {
								const roll = hash2D(vcx, vcz, currentWorld.seed + 9001);
								const spawnAnchor = spawnAnchors[currentWorld.id];
								const isSpawnVillage = vcx === 0 && vcz === 0;
								if (!isSpawnVillage && roll > 0.18) {
									continue;
								}
								const centerX = isSpawnVillage
									? spawnAnchor.x
									: vcx * villageCellSize + 16 + Math.floor(hash2D(vcx * 11 + 5, vcz * 11 - 7, currentWorld.seed + 9033) * (villageCellSize - 32));
								const centerZ = isSpawnVillage
									? spawnAnchor.z
									: vcz * villageCellSize + 16 + Math.floor(hash2D(vcx * 13 - 9, vcz * 13 + 3, currentWorld.seed + 9055) * (villageCellSize - 32));

							const centerBiome = getBiomeAt(centerX, centerZ, currentWorld);
							if (!centerBiome.structures.allowVillage) {
								continue;
							}
							const centerHeight = getHeightAt(centerX, centerZ);
							if (waterLevel > 0 && centerHeight < waterLevel) {
								continue;
							}

							const houseCount = isSpawnVillage ? Math.max(5, currentWorld.structures.houseCount) : currentWorld.structures.houseCount;
							const roadMat: BlockType =
								centerBiome.id === 'desert' || currentWorld.id === 'mars' ? 'sandstone' : 'gravel';

							emitVillageWell(centerX, centerZ, centerHeight, centerBiome);

							const computeHouseBaseY = (hx: number, hz: number, w: number, d: number) => {
								let minH = Number.POSITIVE_INFINITY;
								let maxH = Number.NEGATIVE_INFINITY;
								for (let dx = 0; dx < w; dx += 1) {
									for (let dz = 0; dz < d; dz += 1) {
										const x = hx + dx;
										const z = hz + dz;
										const h = getHeightAt(x, z);
										minH = Math.min(minH, h);
										maxH = Math.max(maxH, h);
									}
								}
								if (maxH - minH > 2) {
									return -1;
								}
								return maxH;
							};

							const pickHouseStyle = (biome: BiomeContext): HousePlan['style'] => {
								if (currentWorld.id === 'mars') return 'desert';
								if (biome.id === 'desert') return 'desert';
								if (biome.id === 'mountains') return 'stone';
								if (biome.id === 'forest' || biome.id === 'swamp') return 'spruce';
								return 'oak';
							};

							const placeHouse = (hx: number, hz: number, seedSalt: number) => {
								const biome = getBiomeAt(hx, hz, currentWorld);
								if (!biome.structures.allowVillage) return;

								const w = 6 + Math.floor(hash2D(hx + seedSalt, hz - seedSalt, currentWorld.seed + 110) * 4);
								const d = 6 + Math.floor(hash2D(hx - seedSalt, hz + seedSalt, currentWorld.seed + 111) * 4);
								const wallHeight =
									3 + Math.floor(hash2D(hx + seedSalt * 2, hz - seedSalt * 2, currentWorld.seed + 112) * 2);
								const roofHeight =
									2 + Math.floor(hash2D(hx - seedSalt * 3, hz + seedSalt * 3, currentWorld.seed + 113) * 2);

								const baseY = computeHouseBaseY(hx, hz, w, d);
								if (baseY < 0) return;
								if (waterLevel > 0 && baseY < waterLevel) return;

								const toCenterX = centerX - hx;
								const toCenterZ = centerZ - hz;
								const facing: 0 | 1 | 2 | 3 =
									Math.abs(toCenterX) > Math.abs(toCenterZ)
										? (toCenterX > 0 ? 3 : 1)
										: (toCenterZ > 0 ? 0 : 2);

								const doorX =
									facing === 0 ? hx + Math.floor(w / 2) : facing === 2 ? hx + Math.floor(w / 2) : facing === 1 ? hx + w - 1 : hx;
								const doorZ =
									facing === 1 ? hz + Math.floor(d / 2) : facing === 3 ? hz + Math.floor(d / 2) : facing === 0 ? hz : hz + d - 1;

								const plan: HousePlan = {
									x: hx,
									z: hz,
									width: w,
									depth: d,
									wallHeight,
									roofHeight,
									facing,
									style: pickHouseStyle(biome),
									baseY,
									doorX,
									doorZ
								};

								emitHouse(plan);

								const pathStartX = doorX + (facing === 1 ? 1 : facing === 3 ? -1 : 0);
								const pathStartZ = doorZ + (facing === 2 ? 1 : facing === 0 ? -1 : 0);
								drawPath(pathStartX, pathStartZ, centerX, centerZ, roadMat);
							};

							if (isSpawnVillage) {
								// Guarantee at least a couple houses close to spawn for immediate visual interest.
								placeHouse(centerX + 8, centerZ - 12, 777);
								placeHouse(centerX - 12, centerZ - 9, 778);
							}

							for (let i = 0; i < houseCount; i += 1) {
								const a = hash2D(vcx * 19 + i * 3, vcz * 23 - i * 2, currentWorld.seed + 9100) * Math.PI * 2;
								const r =
									(isSpawnVillage ? 8 : 10) +
									hash2D(vcx * 29 + i * 7, vcz * 31 + i * 5, currentWorld.seed + 9200) *
										(isSpawnVillage ? 18 : 22);
								const hx = Math.round(centerX + Math.cos(a) * r);
								const hz = Math.round(centerZ + Math.sin(a) * r);
								if (Math.hypot(hx, hz) < 14) {
									continue;
								}
								placeHouse(hx, hz, i + 1);
							}

							drawPath(centerX - 14, centerZ, centerX + 14, centerZ, roadMat);
							drawPath(centerX, centerZ - 14, centerX, centerZ + 14, roadMat);
						}
					}
				};

				emitVillage();

				const chunkBody = world.createRigidBody(
					RAPIER.RigidBodyDesc.fixed().setTranslation(chunkMinX, 0, chunkMinZ)
				);

				for (let ix = 0; ix < chunkSize; ix += 1) {
					for (let iz = 0; iz < chunkSize; iz += 1) {
						const worldX = chunkMinX + ix;
						const worldZ = chunkMinZ + iz;
						const info = getTerrainInfo(worldX, worldZ, currentWorld);
						let height = info.height;

						const idx = ix * chunkSize + iz;
						if (flatTarget[idx] >= 0) {
							height = Math.max(height, flatTarget[idx]);
						}
						const lavaStrength = info.lavaStrength;
						const isVolcanic = lavaStrength > 0.25 || info.biome.id === 'volcanic';
						const useLava = lavaLevel > 0 && isVolcanic && height < lavaLevel;

						const foundationFrom = foundationStart[idx] >= 0 ? foundationStart[idx] : -1;
						const foundationTo = flatTarget[idx] >= 0 ? flatTarget[idx] : -1;
						const foundationMat = foundationMaterial[idx];

						for (let y = 0; y < height; y += 1) {
							let blockType: BlockType;
							if (foundationFrom >= 0 && foundationTo >= 0 && foundationMat && y >= foundationFrom && y < foundationTo - 1) {
								blockType = foundationMat;
							} else {
								blockType = pickBlockType(
									info.biome,
									worldX,
									worldZ,
									y,
									height,
									waterLevel,
									lavaStrength,
									info.riverStrength,
									info.temperature,
									info.humidity
								);
							}
							if (y === height - 1 && topOverride[idx]) {
								blockType = topOverride[idx] as BlockType;
							}
							positionsByType[blockType].push(worldX, y + 0.5, worldZ);
						}

						if (useLava) {
							for (let y = height; y < lavaLevel; y += 1) {
								positionsByType.lava.push(worldX, y + 0.5, worldZ);
							}
						} else if (waterLevel > 0 && height < waterLevel) {
							const maxFill = info.biome.id === 'swamp' ? waterLevel + 1 : waterLevel;
							for (let y = height; y < maxFill; y += 1) {
								positionsByType.water.push(worldX, y + 0.5, worldZ);
							}
						} else if (
							waterLevel > 0 &&
							!useLava &&
							!noPlants[idx] &&
							!topOverride[idx] &&
							info.riverStrength > 0.62 &&
							info.biome.id !== 'ocean' &&
							info.biome.id !== 'beach' &&
							info.biome.id !== 'volcanic'
						) {
							const riverDepth = 1 + Math.floor((info.riverStrength - 0.62) * 3);
							for (let y = height; y < height + Math.min(2, riverDepth); y += 1) {
								positionsByType.water.push(worldX, y + 0.5, worldZ);
							}
						}

						if (
							!useLava &&
							!noPlants[idx] &&
							!topOverride[idx] &&
							(waterLevel <= 0 || height > waterLevel + 1) &&
							isVolcanic &&
							lavaLevel > 0 &&
							lavaStrength > 0.55
						) {
							const patch = valueNoise(worldX * 0.12, worldZ * 0.12, currentWorld.seed + 6211);
							if (patch > 0.82) {
								positionsByType.lava.push(worldX, height + 0.5, worldZ);
							}
						}

						if (!noPlants[idx]) {
							const veg = info.biome.vegetation;
							const density = Math.min(veg.treeDensity, currentWorld.vegetation.treeDensity);
							if (density > 0 && veg.treeScale > 0) {
								const treeNoise = valueNoise(
									worldX * veg.treeScale,
									worldZ * veg.treeScale,
									currentWorld.seed + 5001
								);
								const safeFromWater = waterLevel <= 0 || height > waterLevel + 1;
								const safeFromBounds = ix >= 2 && ix <= chunkSize - 3 && iz >= 2 && iz <= chunkSize - 3;
								if (safeFromWater && safeFromBounds && treeNoise < density) {
									const trunkHeight =
										veg.treeMinHeight +
										Math.floor(
											valueNoise(worldX * 0.12, worldZ * 0.12, currentWorld.seed + 5008) *
												(veg.treeMaxHeight - veg.treeMinHeight + 1)
										);
									for (let ty = 0; ty < trunkHeight; ty += 1) {
										emitSolidBlock(worldX, height + ty, worldZ, 'log', true);
									}
									const crownY = height + trunkHeight;
									for (let dy = -2; dy <= 1; dy += 1) {
										for (let dx = -2; dx <= 2; dx += 1) {
											for (let dz = -2; dz <= 2; dz += 1) {
												const distSq = dx * dx + dz * dz + dy * dy * 1.4;
												if (distSq > 8.8) continue;
												const lx = worldX + dx;
												const lz = worldZ + dz;
												const ly = crownY + dy;
												emitSolidBlock(lx, ly, lz, 'leaves', false);
											}
										}
									}
								}
							}
						}

						if (height > 0) {
							world.createCollider(
								RAPIER.ColliderDesc.cuboid(0.5, height / 2, 0.5).setTranslation(ix, height / 2, iz),
								chunkBody
							);
						}
					}
				}

				for (let i = 0; i < solidBlockColliders.length; i += 3) {
					const x = solidBlockColliders[i];
					const y = solidBlockColliders[i + 1];
					const z = solidBlockColliders[i + 2];
					world.createCollider(
						RAPIER.ColliderDesc.cuboid(0.5, 0.5, 0.5).setTranslation(x - chunkMinX, y, z - chunkMinZ),
						chunkBody
					);
				}

				const meshes: THREE.InstancedMesh[] = [];
				for (const type of blockTypeKeys) {
					const positions = positionsByType[type];
					if (!positions.length) {
						continue;
					}
					const mesh = new THREE.InstancedMesh(blockGeo, blockMats[type], positions.length / 3);
					if (type === 'water') {
						mesh.renderOrder = 2;
					} else if (type === 'lava') {
						mesh.renderOrder = 1;
					}
					for (let i = 0; i < positions.length; i += 3) {
						chunkMatrix.makeTranslation(positions[i], positions[i + 1], positions[i + 2]);
						mesh.setMatrixAt(i / 3, chunkMatrix);
					}
					mesh.instanceMatrix.needsUpdate = true;
					mesh.computeBoundingSphere();
					scene.add(mesh);
					if (type !== 'water' && type !== 'door') {
						cameraOccluders.push(mesh);
					}
					meshes.push(mesh);
				}

				chunks.set(key, { key, x: cx, z: cz, meshes, body: chunkBody });
			};

			const removeChunk = (chunk: Chunk) => {
				for (const mesh of chunk.meshes) {
					scene.remove(mesh);
					const occIndex = cameraOccluders.indexOf(mesh);
					if (occIndex >= 0) {
						cameraOccluders.splice(occIndex, 1);
					}
				}
				world.removeRigidBody(chunk.body);
			};

			const clearChunks = () => {
				for (const chunk of chunks.values()) {
					removeChunk(chunk);
				}
				chunks.clear();
			};

			let lastChunkX = Number.NaN;
			let lastChunkZ = Number.NaN;

			const syncChunks = (worldX: number, worldZ: number, force = false) => {
				const cx = Math.floor(worldX / chunkSize);
				const cz = Math.floor(worldZ / chunkSize);
				if (!force && cx === lastChunkX && cz === lastChunkZ) {
					return;
				}
				lastChunkX = cx;
				lastChunkZ = cz;
				for (let x = cx - chunkRadius; x <= cx + chunkRadius; x += 1) {
					for (let z = cz - chunkRadius; z <= cz + chunkRadius; z += 1) {
						buildChunk(x, z);
					}
				}
				for (const [key, chunk] of chunks.entries()) {
					if (
						Math.abs(chunk.x - cx) > chunkRadius ||
						Math.abs(chunk.z - cz) > chunkRadius
					) {
						removeChunk(chunk);
						chunks.delete(key);
					}
				}
			};

			const playerHeight = 1.8;
			const playerWidth = 0.6;
			const playerDepth = 0.6;
			const playerEyeHeight = playerHeight * 0.9;

			const player = new THREE.Group();
			const bodyMat = new THREE.MeshStandardMaterial({ color: 0xffd07a, roughness: 0.4 });
			const limbMat = new THREE.MeshStandardMaterial({ color: 0x334856, roughness: 0.6 });

			const body = new THREE.Mesh(new THREE.CapsuleGeometry(0.35, 0.9, 6, 12), bodyMat);
			body.position.y = 1.2;
			player.add(body);

			const head = new THREE.Mesh(new THREE.SphereGeometry(0.32, 16, 16), bodyMat);
			head.position.y = 2.0;
			player.add(head);

			const legLeft = new THREE.Mesh(new THREE.BoxGeometry(0.22, 0.7, 0.24), limbMat);
			legLeft.position.set(-0.2, 0.35, 0);
			player.add(legLeft);

			const legRight = new THREE.Mesh(new THREE.BoxGeometry(0.22, 0.7, 0.24), limbMat);
			legRight.position.set(0.2, 0.35, 0);
			player.add(legRight);

			const armLeft = new THREE.Mesh(new THREE.BoxGeometry(0.18, 0.6, 0.18), limbMat);
			armLeft.position.set(-0.52, 1.25, 0);
			player.add(armLeft);

			const armRight = new THREE.Mesh(new THREE.BoxGeometry(0.18, 0.6, 0.18), limbMat);
			armRight.position.set(0.52, 1.25, 0);
			player.add(armRight);

			const playerBounds = new THREE.Box3().setFromObject(player);
			const playerSize = new THREE.Vector3();
			playerBounds.getSize(playerSize);
			player.scale.set(
				playerWidth / playerSize.x,
				playerHeight / playerSize.y,
				playerDepth / playerSize.z
			);

			const initialSpawn = findSafeSpawn(currentWorld);
			player.position.set(initialSpawn.x, initialSpawn.y, initialSpawn.z);
			const playerBody = world.createRigidBody(
				RAPIER.RigidBodyDesc.kinematicPositionBased().setTranslation(
					initialSpawn.x,
					initialSpawn.y,
					initialSpawn.z
				)
			);
			const playerColliderDesc = RAPIER.ColliderDesc.cuboid(
				playerWidth / 2,
				playerHeight / 2,
				playerDepth / 2
			);
			playerColliderDesc.setTranslation(0, playerHeight / 2, 0);
			playerColliderDesc.setFriction(0.2);
			const playerCollider = world.createCollider(playerColliderDesc, playerBody);
			const controller = world.createCharacterController(0.05);
			controller.disableAutostep();
			controller.enableSnapToGround(0.35);
			controller.setMaxSlopeClimbAngle(Math.PI / 4);
			controller.setMinSlopeSlideAngle(Math.PI / 3);
			scene.add(player);

			type Bot = {
				group: THREE.Group;
				armLeft: THREE.Mesh;
				armRight: THREE.Mesh;
				legLeft: THREE.Mesh;
				legRight: THREE.Mesh;
				home: THREE.Vector2;
				target: THREE.Vector2;
				speed: number;
				phase: number;
				targetTimer: number;
			};

			const botHeadGeo = new THREE.BoxGeometry(0.55, 0.55, 0.55);
			const botBodyGeo = new THREE.BoxGeometry(0.55, 0.82, 0.32);
			const botLimbGeo = new THREE.BoxGeometry(0.18, 0.68, 0.18);
			const botEyeGeo = new THREE.BoxGeometry(0.07, 0.07, 0.02);
			const botNoseGeo = new THREE.BoxGeometry(0.12, 0.18, 0.18);
			const villagerSkinMat = new THREE.MeshStandardMaterial({ color: 0xffd2a1, roughness: 0.55 });
			const villagerEyeMat = new THREE.MeshStandardMaterial({ color: 0x121417, roughness: 0.9 });
			const villagerRobeMats = [
				new THREE.MeshStandardMaterial({ color: 0x567a55, roughness: 0.85 }),
				new THREE.MeshStandardMaterial({ color: 0x7a5656, roughness: 0.85 }),
				new THREE.MeshStandardMaterial({ color: 0x51657f, roughness: 0.85 }),
				new THREE.MeshStandardMaterial({ color: 0x7a6a3d, roughness: 0.85 }),
				new THREE.MeshStandardMaterial({ color: 0x3d6c7a, roughness: 0.85 })
			];

			const bots: Bot[] = [];

			const createVillagerBot = (seed: number, homeX: number, homeZ: number): Bot => {
				const robeMat = villagerRobeMats[seed % villagerRobeMats.length] ?? villagerRobeMats[0];
				const group = new THREE.Group();

				const body = new THREE.Mesh(botBodyGeo, robeMat);
				body.position.y = 0.95;
				group.add(body);

				const head = new THREE.Mesh(botHeadGeo, villagerSkinMat);
				head.position.y = 1.55;
				group.add(head);

				const nose = new THREE.Mesh(botNoseGeo, villagerSkinMat);
				nose.position.set(0, -0.05, 0.365);
				head.add(nose);

				const eyeLeft = new THREE.Mesh(botEyeGeo, villagerEyeMat);
				eyeLeft.position.set(-0.13, 0.06, 0.285);
				head.add(eyeLeft);
				const eyeRight = eyeLeft.clone();
				eyeRight.position.x = 0.13;
				head.add(eyeRight);

				const legLeft = new THREE.Mesh(botLimbGeo, robeMat);
				legLeft.position.set(-0.14, 0.34, 0);
				group.add(legLeft);
				const legRight = new THREE.Mesh(botLimbGeo, robeMat);
				legRight.position.set(0.14, 0.34, 0);
				group.add(legRight);

				const armLeft = new THREE.Mesh(botLimbGeo, robeMat);
				armLeft.position.set(-0.38, 0.98, 0);
				group.add(armLeft);
				const armRight = new THREE.Mesh(botLimbGeo, robeMat);
				armRight.position.set(0.38, 0.98, 0);
				group.add(armRight);

				const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
				const ground = Math.max(getHeightAt(homeX, homeZ), fluidSurface) + 0.25;
				group.position.set(homeX, ground, homeZ);

				return {
					group,
					armLeft,
					armRight,
					legLeft,
					legRight,
					home: new THREE.Vector2(homeX, homeZ),
					target: new THREE.Vector2(homeX, homeZ),
					speed: 2.2 + (hash2D(seed, seed * 7, currentWorld.seed + 8801) - 0.5) * 0.6,
					phase: hash2D(seed * 3, seed * 11, currentWorld.seed + 8811) * Math.PI * 2,
					targetTimer: 0
				};
			};

			const pickBotTarget = (bot: Bot, seed: number) => {
				const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
				for (let attempt = 0; attempt < 10; attempt += 1) {
					const a = hash2D(seed + attempt * 17, seed - attempt * 9, currentWorld.seed + 8200) * Math.PI * 2;
					const r = 4 + hash2D(seed + attempt * 5, seed + attempt * 13, currentWorld.seed + 8201) * 16;
					const tx = Math.round(bot.home.x + Math.cos(a) * r);
					const tz = Math.round(bot.home.y + Math.sin(a) * r);
					const info = getTerrainInfo(tx, tz, currentWorld);
					if (currentWorld.fluids.waterLevel > 0 && info.height < currentWorld.fluids.waterLevel) {
						continue;
					}
					if (info.lavaStrength > 0.22) {
						continue;
					}
					const surface = Math.max(info.height, fluidSurface);
					let minH = Number.POSITIVE_INFINITY;
					let maxH = Number.NEGATIVE_INFINITY;
					for (let ox = -1; ox <= 1; ox += 1) {
						for (let oz = -1; oz <= 1; oz += 1) {
							const h = computeHeight(tx + ox, tz + oz, currentWorld);
							minH = Math.min(minH, h);
							maxH = Math.max(maxH, h);
						}
					}
					if (maxH - minH > 3) {
						continue;
					}
					bot.target.set(tx, tz);
					bot.targetTimer = 2.2 + hash2D(tx, tz, currentWorld.seed + 8300) * 3.6;
					bot.group.position.y = Math.max(bot.group.position.y, surface + 0.25);
					return;
				}
				bot.target.copy(bot.home);
				bot.targetTimer = 2;
			};

			const clearBots = () => {
				for (const bot of bots) {
					scene.remove(bot.group);
				}
				bots.length = 0;
			};

				const spawnBotsForWorld = () => {
					clearBots();
					const defaultHome = spawnAnchors[currentWorld.id];
					const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
					const usedHomes = new Set<string>();

				const count = currentWorld.id === 'earth' ? 8 : 6;
				const preferredHomes: Array<{ x: number; z: number }> = [
					{ x: defaultHome.x + 2, z: defaultHome.z - 5 },
					{ x: defaultHome.x - 4, z: defaultHome.z - 4 },
					{ x: defaultHome.x + 5, z: defaultHome.z - 2 }
				];

				let botIndex = 0;
				for (const home of preferredHomes) {
					if (botIndex >= count) break;
					const info = getTerrainInfo(home.x, home.z, currentWorld);
					if (currentWorld.fluids.waterLevel > 0 && info.height < currentWorld.fluids.waterLevel) {
						continue;
					}
					if (info.lavaStrength > 0.22) {
						continue;
					}
					const key = `${home.x},${home.z}`;
					if (usedHomes.has(key)) {
						continue;
					}
					usedHomes.add(key);
					const bot = createVillagerBot(botIndex, home.x, home.z);
					bot.group.position.y = Math.max(bot.group.position.y, Math.max(info.height, fluidSurface) + 0.25);
					bots.push(bot);
					scene.add(bot.group);
					pickBotTarget(bot, botIndex + 31);
					botIndex += 1;
				}

				for (; botIndex < count; botIndex += 1) {
					const a = hash2D(botIndex * 7, botIndex * 13, currentWorld.seed + 8800) * Math.PI * 2;
					const r = 3 + hash2D(botIndex * 11, botIndex * 5, currentWorld.seed + 8809) * 10;
					const hx = Math.round(defaultHome.x + Math.cos(a) * r);
					const hz = Math.round(defaultHome.z + Math.sin(a) * r);
					const key = `${hx},${hz}`;
					if (usedHomes.has(key)) {
						continue;
					}
					const info = getTerrainInfo(hx, hz, currentWorld);
					if (currentWorld.fluids.waterLevel > 0 && info.height < currentWorld.fluids.waterLevel) {
						continue;
					}
					if (info.lavaStrength > 0.22) {
						continue;
					}
					usedHomes.add(key);
					const bot = createVillagerBot(botIndex, hx, hz);
					bot.group.position.y = Math.max(bot.group.position.y, Math.max(info.height, fluidSurface) + 0.25);
					bots.push(bot);
					scene.add(bot.group);
					pickBotTarget(bot, botIndex + 31);
				}
			};

			const bgmCache = new Map<string, HTMLAudioElement>();
			const fadingOut: HTMLAudioElement[] = [];
			let audioUnlocked = false;
			let currentBgm: HTMLAudioElement | null = null;
			let bgmTargetVolume = currentWorld.music.volume;
			let pendingBgm: { url: string; volume: number } | null = {
				url: currentWorld.music.url,
				volume: currentWorld.music.volume
			};
			const portalSfx = new Audio('/audio/portal.ogg');
			portalSfx.preload = 'auto';
			portalSfx.volume = 0.7;

			const getBgm = (url: string) => {
				const cached = bgmCache.get(url);
				if (cached) {
					return cached;
				}
				const audio = new Audio(url);
				audio.loop = true;
				audio.volume = 0;
				audio.preload = 'auto';
				bgmCache.set(url, audio);
				return audio;
			};

			const switchBgm = (url: string, volume: number) => {
				if (!audioUnlocked) {
					pendingBgm = { url, volume };
					return;
				}
				const next = getBgm(url);
				bgmTargetVolume = volume;
				if (currentBgm === next) {
					return;
				}
				if (currentBgm) {
					fadingOut.push(currentBgm);
				}
				currentBgm = next;
				currentBgm.currentTime = 0;
				currentBgm.volume = 0;
				currentBgm.play().catch(() => {});
			};

			const ensureAudio = () => {
				if (audioUnlocked) {
					return;
				}
				audioUnlocked = true;
				if (pendingBgm) {
					switchBgm(pendingBgm.url, pendingBgm.volume);
					pendingBgm = null;
				} else {
					switchBgm(currentWorld.music.url, currentWorld.music.volume);
				}
			};

			const updateAudio = (delta: number) => {
				if (!audioUnlocked) {
					return;
				}
				if (currentBgm) {
					currentBgm.volume += (bgmTargetVolume - currentBgm.volume) * Math.min(1, delta * 2.2);
				}
				for (let i = fadingOut.length - 1; i >= 0; i -= 1) {
					const fading = fadingOut[i];
					fading.volume = Math.max(0, fading.volume - delta * 0.6);
					if (fading.volume <= 0.001) {
						fading.pause();
						fading.currentTime = 0;
						fadingOut.splice(i, 1);
					}
				}
			};

			const playPortalSound = () => {
				if (!audioUnlocked) {
					return;
				}
				const instance = portalSfx.cloneNode(true) as HTMLAudioElement;
				instance.volume = portalSfx.volume;
				instance.play().catch(() => {});
			};

			const input = {
				forward: false,
				back: false,
				left: false,
				right: false,
				jump: false
			};

			const handleKeyDown = (event: KeyboardEvent) => {
				ensureAudio();
				switch (event.code) {
					case 'KeyW':
					case 'ArrowUp':
						input.forward = true;
						break;
					case 'KeyS':
					case 'ArrowDown':
						input.back = true;
						break;
					case 'KeyA':
					case 'ArrowLeft':
						input.left = true;
						break;
					case 'KeyD':
					case 'ArrowRight':
						input.right = true;
						break;
					case 'Space':
						input.jump = true;
						break;
					default:
						break;
				}
			};

			const handleKeyUp = (event: KeyboardEvent) => {
				switch (event.code) {
					case 'KeyW':
					case 'ArrowUp':
						input.forward = false;
						break;
					case 'KeyS':
					case 'ArrowDown':
						input.back = false;
						break;
					case 'KeyA':
					case 'ArrowLeft':
						input.left = false;
						break;
					case 'KeyD':
					case 'ArrowRight':
						input.right = false;
						break;
					case 'Space':
						input.jump = false;
						break;
					default:
						break;
				}
			};

			window.addEventListener('keydown', handleKeyDown);
			window.addEventListener('keyup', handleKeyUp);

			const moveAxis = new THREE.Vector2();
			const pointerState = {
				lookYaw: 0,
				lookPitch: 0,
				mouseDown: false,
				lastMouseX: 0,
				lastMouseY: 0
			};

			const handleMouseDown = (event: PointerEvent) => {
				if (event.pointerType !== 'mouse' || event.button !== 0) {
					return;
				}
				ensureAudio();
				pointerState.mouseDown = true;
				pointerState.lastMouseX = event.clientX;
				pointerState.lastMouseY = event.clientY;
				if (event.target instanceof HTMLElement) {
					event.target.setPointerCapture(event.pointerId);
				}
			};

			const handleMouseMove = (event: PointerEvent) => {
				if (event.pointerType !== 'mouse' || !pointerState.mouseDown) {
					return;
				}
				const deltaX = event.clientX - pointerState.lastMouseX;
				const deltaY = event.clientY - pointerState.lastMouseY;
				pointerState.lastMouseX = event.clientX;
				pointerState.lastMouseY = event.clientY;
				pointerState.lookYaw += deltaX * 0.003;
				pointerState.lookPitch += deltaY * 0.003;
			};

			const handleMouseUp = (event: PointerEvent) => {
				if (event.pointerType !== 'mouse') {
					return;
				}
				pointerState.mouseDown = false;
			};

			renderer.domElement.addEventListener('pointerdown', handleMouseDown);
			window.addEventListener('pointermove', handleMouseMove);
			window.addEventListener('pointerup', handleMouseUp);

			const joystickState = {
				pointerId: null as number | null,
				centerX: 0,
				centerY: 0,
				radius: 0
			};

			const lookState = {
				pointerId: null as number | null,
				lastX: 0,
				lastY: 0
			};

			const updateJoystickBounds = () => {
				if (!joystickEl) {
					return;
				}
				const rect = joystickEl.getBoundingClientRect();
				joystickState.centerX = rect.left + rect.width / 2;
				joystickState.centerY = rect.top + rect.height / 2;
				joystickState.radius = rect.width / 2;
			};

			const updateJoystickThumb = (dx: number, dy: number) => {
				if (!joystickThumbEl) {
					return;
				}
				joystickThumbEl.style.transform = `translate(${dx}px, ${dy}px)`;
			};

			const handleJoystickDown = (event: PointerEvent) => {
				if (event.pointerType != 'touch' || !joystickEl) {
					return;
				}
				ensureAudio();
				joystickState.pointerId = event.pointerId;
				joystickEl.setPointerCapture(event.pointerId);
				updateJoystickBounds();
				handleJoystickMove(event);
				event.preventDefault();
			};

			const handleJoystickMove = (event: PointerEvent) => {
				if (event.pointerId != joystickState.pointerId) {
					return;
				}
				const radius = joystickState.radius || 1;
				const dx = event.clientX - joystickState.centerX;
				const dy = event.clientY - joystickState.centerY;
				const distance = Math.min(Math.hypot(dx, dy), radius);
				const angle = Math.atan2(dy, dx);
				const nx = Math.cos(angle) * (distance / radius);
				const ny = Math.sin(angle) * (distance / radius);
				moveAxis.set(nx, ny);
				updateJoystickThumb(nx * radius * 0.55, ny * radius * 0.55);
				event.preventDefault();
			};

			const handleJoystickUp = (event: PointerEvent) => {
				if (event.pointerId != joystickState.pointerId) {
					return;
				}
				joystickState.pointerId = null;
				moveAxis.set(0, 0);
				updateJoystickThumb(0, 0);
				joystickEl?.releasePointerCapture(event.pointerId);
			};

			const handleJumpDown = (event: PointerEvent) => {
				if (event.pointerType === 'mouse') {
					return;
				}
				ensureAudio();
				input.jump = true;
				event.preventDefault();
				event.stopPropagation();
			};

			const handleJumpUp = (event: PointerEvent) => {
				if (event.pointerType === 'mouse') {
					return;
				}
				input.jump = false;
				event.stopPropagation();
			};

			const handleLookDown = (event: PointerEvent) => {
				if (event.pointerType === 'mouse') {
					return;
				}
				if (event.target instanceof HTMLElement && event.target.closest('.touch-pad')) {
					return;
				}
				ensureAudio();
				lookState.pointerId = event.pointerId;
				lookState.lastX = event.clientX;
				lookState.lastY = event.clientY;
				if (event.target instanceof HTMLElement) {
					event.target.setPointerCapture(event.pointerId);
				}
				event.preventDefault();
			};

			const handleLookMove = (event: PointerEvent) => {
				if (event.pointerId != lookState.pointerId) {
					return;
				}
				const dx = event.clientX - lookState.lastX;
				const dy = event.clientY - lookState.lastY;
				lookState.lastX = event.clientX;
				lookState.lastY = event.clientY;
				pointerState.lookYaw += dx * 0.004;
				pointerState.lookPitch += dy * 0.004;
				event.preventDefault();
			};

			const handleLookUp = (event: PointerEvent) => {
				if (event.pointerId != lookState.pointerId) {
					return;
				}
				lookState.pointerId = null;
				if (event.target instanceof HTMLElement) {
					event.target.releasePointerCapture(event.pointerId);
				}
			};

			joystickEl?.addEventListener('pointerdown', handleJoystickDown, { passive: false });
			joystickEl?.addEventListener('pointermove', handleJoystickMove, { passive: false });
			joystickEl?.addEventListener('pointerup', handleJoystickUp);
			joystickEl?.addEventListener('pointercancel', handleJoystickUp);
			jumpEl?.addEventListener('pointerdown', handleJumpDown, { passive: false });
			jumpEl?.addEventListener('pointerup', handleJumpUp);
			jumpEl?.addEventListener('pointercancel', handleJumpUp);
			renderer.domElement.addEventListener('pointerdown', handleLookDown, { passive: false });
			renderer.domElement.addEventListener('pointermove', handleLookMove, { passive: false });
			renderer.domElement.addEventListener('pointerup', handleLookUp);
			renderer.domElement.addEventListener('pointercancel', handleLookUp);

			type WorldId = WorldDefinition['id'];

			type Portal = {
				targetId: WorldId;
				group: THREE.Group;
				core: THREE.Mesh;
				frame: THREE.Mesh;
				label: THREE.Sprite;
				color: THREE.Color;
				pulseOffset: number;
			};

			type PortalParticle = {
				mesh: THREE.Mesh;
				velocity: THREE.Vector3;
				life: number;
			};

			type FallingBlock = {
				mesh: THREE.Mesh;
				body: RAPIER.RigidBody;
				collider: RAPIER.Collider;
				isTnt: boolean;
				detonationQueued: boolean;
				removed: boolean;
			};

			type ExplosionParticle = {
				mesh: THREE.Mesh;
				velocity: THREE.Vector3;
				life: number;
			};

			type PendingDetonation = {
				block: FallingBlock;
				time: number;
				chainLevel: number;
			};

			const fallingBlocks: FallingBlock[] = [];
			const blockByCollider = new Map<number, FallingBlock>();
			const particles: ExplosionParticle[] = [];
			const pushImpulse = new THREE.Vector3();
			const pendingDetonations: PendingDetonation[] = [];
			const explosionRadius = 4.5;
			const explosionSpeed = 6.0;
			const maxChainLevel = 4;
			const portals: Portal[] = [];
			const portalParticles: PortalParticle[] = [];
			let portalTransition: { targetId: WorldId; timer: number } | null = null;
			let portalCooldown = 0;

			const portalLinks: Record<WorldId, WorldId[]> = {
				earth: ['mars', 'moon'],
				mars: ['earth', 'moon'],
				moon: ['earth', 'mars']
			};

			const createLabelTexture = (text: string, color: string) => {
				const canvas = document.createElement('canvas');
				canvas.width = 256;
				canvas.height = 64;
				const ctx = canvas.getContext('2d');
				if (ctx) {
					ctx.imageSmoothingEnabled = false;
					ctx.fillStyle = 'rgba(8, 12, 16, 0.7)';
					ctx.fillRect(0, 0, canvas.width, canvas.height);
					ctx.strokeStyle = color;
					ctx.lineWidth = 4;
					ctx.strokeRect(6, 6, canvas.width - 12, canvas.height - 12);
					ctx.fillStyle = '#fef3dd';
					ctx.font = 'bold 26px "Space Grotesk", sans-serif';
					ctx.textAlign = 'center';
					ctx.textBaseline = 'middle';
					ctx.fillText(text.toUpperCase(), canvas.width / 2, canvas.height / 2);
				}
				const texture = new THREE.CanvasTexture(canvas);
				texture.colorSpace = THREE.SRGBColorSpace;
				texture.magFilter = THREE.NearestFilter;
				texture.minFilter = THREE.NearestMipMapNearestFilter;
				return texture;
			};

			const clearPortals = () => {
				for (const portal of portals) {
					scene.remove(portal.group);
					const occIndex = cameraOccluders.indexOf(portal.frame);
					if (occIndex >= 0) {
						cameraOccluders.splice(occIndex, 1);
					}
					(portal.core.material as THREE.Material).dispose();
					const labelMat = portal.label.material as THREE.SpriteMaterial;
					labelMat.map?.dispose();
					labelMat.dispose();
				}
				portals.length = 0;
			};

			const buildPortals = () => {
				clearPortals();
				const targets = portalLinks[currentWorld.id];
				const baseAngle = (currentWorld.seed % 10) * 0.32;
				targets.forEach((targetId, index) => {
					const target = worldById.get(targetId);
					if (!target) {
						return;
					}
					const angle = baseAngle + (index / targets.length) * Math.PI * 2;
					const radius = 12 + index * 6;
					const x = Math.round(Math.cos(angle) * radius);
					const z = Math.round(Math.sin(angle) * radius);
					const terrainY = getHeightAt(x, z);
					const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
					const y = Math.max(terrainY, fluidSurface) + 1.2;

					const group = new THREE.Group();
					const frame = new THREE.Mesh(portalFrameGeo, portalFrameMat);
					frame.castShadow = true;
					group.add(frame);

					const coreColor = new THREE.Color(target.portalColor);
					const coreMat = new THREE.MeshStandardMaterial({
						color: coreColor,
						emissive: coreColor,
						emissiveIntensity: 0.95,
						transparent: true,
						opacity: 0.75,
						roughness: 0.2,
						side: THREE.DoubleSide
					});
					const core = new THREE.Mesh(portalCoreGeo, coreMat);
					core.position.z = 0.02;
					group.add(core);

					const labelTexture = createLabelTexture(`To ${target.name}`, target.portalColor);
					const labelMat = new THREE.SpriteMaterial({
						map: labelTexture,
						transparent: true,
						depthTest: false
					});
					const label = new THREE.Sprite(labelMat);
					label.position.set(0, 1.5, 0);
					label.scale.set(3.6, 0.9, 1);
					group.add(label);

					group.position.set(x, y, z);
					group.rotation.y = Math.atan2(x, z);
					scene.add(group);
					cameraOccluders.push(frame);
					portals.push({
						targetId,
						group,
						core,
						frame,
						label,
						color: coreColor,
						pulseOffset: Math.random() * Math.PI * 2
					});
				});
			};

			const spawnPortalBurst = (position: THREE.Vector3, color: THREE.Color) => {
				portalParticleMat.color.copy(color);
				portalParticleMat.emissive.copy(color);
				const count = 80;
				for (let i = 0; i < count; i += 1) {
					const mesh = new THREE.Mesh(portalParticleGeo, portalParticleMat);
					mesh.position.copy(position);
					const velocity = new THREE.Vector3(
						THREE.MathUtils.randFloatSpread(2),
						THREE.MathUtils.randFloat(0.8, 2.8),
						THREE.MathUtils.randFloatSpread(2)
					)
						.normalize()
						.multiplyScalar(THREE.MathUtils.randFloat(2.8, 5.4));
					portalParticles.push({ mesh, velocity, life: THREE.MathUtils.randFloat(0.7, 1.4) });
					scene.add(mesh);
				}
			};

			const triggerPortal = (portal: Portal) => {
				if (portalTransition || portalCooldown > 0) {
					return;
				}
				portalTransition = { targetId: portal.targetId, timer: 0.7 };
				playPortalSound();
				spawnPortalBurst(player.position, portal.color);
				spawnPortalBurst(portal.group.position, portal.color);
			};

			const applyWorld = (worldDef: WorldDefinition, resetPlayer = true) => {
				currentWorld = worldDef;
				worldLabel = worldDef.name;
				gravity = worldDef.gravity;
				world.gravity = { x: 0, y: gravity, z: 0 };
				scene.background = new THREE.Color(worldDef.skyColor);
				scene.fog = new THREE.Fog(worldDef.fogColor, worldDef.fogNear, worldDef.fogFar);
				hemiLight.color.setHex(worldDef.light.hemiSky);
				hemiLight.groundColor.setHex(worldDef.light.hemiGround);
				hemiLight.intensity = worldDef.light.hemiIntensity;
				dirLight.color.setHex(worldDef.light.dirColor);
				dirLight.intensity = worldDef.light.dirIntensity;
				dirLight.position.set(...worldDef.light.dirPos);
				switchBgm(worldDef.music.url, worldDef.music.volume);
				clearChunks();
				for (const block of fallingBlocks) {
					removeBlock(block);
				}
				fallingBlocks.length = 0;
				blockByCollider.clear();
				for (const particle of particles) {
					scene.remove(particle.mesh);
				}
				particles.length = 0;
				for (const particle of portalParticles) {
					scene.remove(particle.mesh);
				}
				portalParticles.length = 0;
				pendingDetonations.length = 0;
				buildPortals();
				spawnBotsForWorld();
				if (resetPlayer) {
					const spawn = findSafeSpawn(worldDef);
					playerBody.setNextKinematicTranslation({ x: spawn.x, y: spawn.y, z: spawn.z });
					player.position.set(spawn.x, spawn.y, spawn.z);
					verticalVelocity = 0;
					grounded = false;
				}
				syncChunks(player.position.x, player.position.z, true);
				portalCooldown = 1.1;
			};

			worldJump = (id) => {
				const target = worldById.get(id);
				if (target) {
					applyWorld(target, true);
				}
			};

			const spawnFallingBlock = () => {
				const radius = 10;
				const spawnX = player.position.x + THREE.MathUtils.randFloatSpread(radius * 2);
				const spawnZ = player.position.z + THREE.MathUtils.randFloatSpread(radius * 2);
				const surface = getHeightAt(Math.round(spawnX), Math.round(spawnZ));
				const spawnY = surface + THREE.MathUtils.randFloat(8, 16);
				const isTnt = Math.random() < 0.24;
				const baseMats =
					currentWorld.id === 'mars'
						? marsRockMats
						: currentWorld.id === 'moon'
							? moonDustMats
							: stoneMats;
				const block = new THREE.Mesh(fallingBlockGeo, isTnt ? tntMats : baseMats);
				block.position.set(spawnX, spawnY, spawnZ);
				cameraOccluders.push(block);
				scene.add(block);

				const bodyDesc = RAPIER.RigidBodyDesc.dynamic().setTranslation(spawnX, spawnY, spawnZ);
				const bodyInstance = world.createRigidBody(bodyDesc);
				const colliderDesc = RAPIER.ColliderDesc.cuboid(0.25, 0.25, 0.25);
				colliderDesc.setFriction(0.9);
				colliderDesc.setRestitution(0.1);
				const collider = world.createCollider(colliderDesc, bodyInstance);

				const fallingBlock: FallingBlock = {
					mesh: block,
					body: bodyInstance,
					collider,
					isTnt,
					detonationQueued: false,
					removed: false
				};
				blockByCollider.set(collider.handle, fallingBlock);
				fallingBlocks.push(fallingBlock);
			};

			const spawnExplosion = (position: THREE.Vector3) => {
				const count = 120;
				for (let i = 0; i < count; i += 1) {
					const mesh = new THREE.Mesh(particleGeo, particleMat);
					mesh.position.copy(position);
					const velocity = new THREE.Vector3(
						THREE.MathUtils.randFloatSpread(2.2),
						THREE.MathUtils.randFloat(1.2, 3.0),
						THREE.MathUtils.randFloatSpread(2.2)
					)
						.normalize()
						.multiplyScalar(THREE.MathUtils.randFloat(3.6, 7.2));
					particles.push({ mesh, velocity, life: THREE.MathUtils.randFloat(0.9, 1.6) });
					scene.add(mesh);
				}
			};

			const removeBlock = (block: FallingBlock) => {
				if (block.removed) {
					return;
				}
				block.removed = true;
				scene.remove(block.mesh);
				const occluderIndex = cameraOccluders.indexOf(block.mesh);
				if (occluderIndex >= 0) {
					cameraOccluders.splice(occluderIndex, 1);
				}
				blockByCollider.delete(block.collider.handle);
				world.removeRigidBody(block.body);
			};

			const scheduleChainDetonations = (origin: THREE.Vector3, chainLevel: number) => {
				if (chainLevel >= maxChainLevel) {
					return;
				}
				for (const block of fallingBlocks) {
					if (!block.isTnt || block.removed || block.detonationQueued) {
						continue;
					}
					const distance = origin.distanceTo(block.mesh.position);
					if (distance > explosionRadius) {
						continue;
					}
					block.detonationQueued = true;
					const delay = Math.max(distance / explosionSpeed, 0.08);
					pendingDetonations.push({
						block,
						time: delay,
						chainLevel: chainLevel + 1
					});
				}
			};

			const detonateBlock = (block: FallingBlock, chainLevel = 0) => {
				if (block.removed) {
					return;
				}
				spawnExplosion(block.mesh.position);
				scheduleChainDetonations(block.mesh.position, chainLevel);
				removeBlock(block);
			};

			const forwardBase = new THREE.Vector3(0, 0, -1);
			const xAxis = new THREE.Vector3(1, 0, 0);
			const headOffset = new THREE.Vector3(0, playerEyeHeight, 0);
			const tempVec = new THREE.Vector3();
			const tempVec2 = new THREE.Vector3();
			const tempVec3 = new THREE.Vector3();
			const raycaster = new THREE.Raycaster();
			
			const clock = new THREE.Clock();
			let frame = 0;
			let spawnTimer = 0.6;
			let verticalVelocity = 0;
			let grounded = false;

			applyWorld(currentWorld, false);

			const tick = () => {
				const delta = Math.min(clock.getDelta(), 0.05);
				const time = clock.elapsedTime;
				updateAudio(delta);

				if (portalCooldown > 0) {
					portalCooldown = Math.max(0, portalCooldown - delta);
				}
				if (portalTransition) {
					portalTransition.timer -= delta;
					if (portalTransition.timer <= 0) {
						const target = worldById.get(portalTransition.targetId);
						if (target) {
							applyWorld(target, true);
						}
						portalTransition = null;
					}
				}
				const isTransitioning = portalTransition !== null;
				if (isTransitioning) {
					input.jump = false;
				}

				const analogTurn = moveAxis.x;
				const analogMove = -moveAxis.y;
				const turnInput = isTransitioning
					? 0
					: -((input.left ? -1 : 0) + (input.right ? 1 : 0) + analogTurn);

				let moveInput = analogMove * 0.9;
				if (isTransitioning) {
					moveInput = 0;
				} else if (input.forward) {
					moveInput = 1.2;
				} else if (input.back) {
					moveInput = -0.4;
				}

				moveInput = THREE.MathUtils.clamp(moveInput, -0.6, 1.2);
				player.rotation.y += turnInput * 1.6 * delta + pointerState.lookYaw;
				pointerState.lookYaw = 0;
				cameraPitch = THREE.MathUtils.clamp(
					cameraPitch + pointerState.lookPitch,
					minPitch,
					maxPitch
				);
				pointerState.lookPitch = 0;

				if (grounded && verticalVelocity < 0) {
					verticalVelocity = 0;
				}
				if (input.jump && grounded) {
					verticalVelocity = 7.2;
					grounded = false;
					input.jump = false;
				}
				verticalVelocity += gravity * delta;

				tempVec.copy(forwardBase).applyQuaternion(player.quaternion);
				const desiredMovement = tempVec2
					.copy(tempVec)
					.multiplyScalar(4.2 * moveInput * delta);
				desiredMovement.y = verticalVelocity * delta;

				controller.computeColliderMovement(playerCollider, desiredMovement);
				const tntHits = new Set<FallingBlock>();
				const pushHits = new Set<FallingBlock>();
				const collisionCount = controller.numComputedCollisions();
				for (let i = 0; i < collisionCount; i += 1) {
					const collision = controller.computedCollision(i);
					const collider = collision?.collider;
					if (!collider) {
						continue;
					}
					const block = blockByCollider.get(collider.handle);
					if (!block || block.removed) {
						continue;
					}
					if (block.isTnt) {
						tntHits.add(block);
					} else {
						pushHits.add(block);
					}
				}
				const actualMovement = controller.computedMovement();
				const currentPos = playerBody.translation();
				const nextPos = {
					x: currentPos.x + actualMovement.x,
					y: currentPos.y + actualMovement.y,
					z: currentPos.z + actualMovement.z
				};

				playerBody.setNextKinematicTranslation(nextPos);
				player.position.set(nextPos.x, nextPos.y, nextPos.z);
				syncChunks(player.position.x, player.position.z);

				grounded = controller.computedGrounded();
				if (grounded && verticalVelocity < 0) {
					verticalVelocity = 0;
				}

				const movementStrength = Math.min(Math.abs(moveInput), 1);
				const stride = Math.sin(time * 8) * 0.7 * movementStrength;
				legLeft.rotation.x = stride;
				legRight.rotation.x = -stride;
				armLeft.rotation.x = -stride * 0.7;
				armRight.rotation.x = stride * 0.7;
				const bob = Math.abs(Math.sin(time * 8)) * 0.05 * movementStrength;
				body.position.y = 1.2 + bob;

				if (!portalTransition) {
					for (let i = 0; i < bots.length; i += 1) {
						const bot = bots[i];
						bot.targetTimer -= delta;
						const dx = bot.target.x - bot.group.position.x;
						const dz = bot.target.y - bot.group.position.z;
						const distSq = dx * dx + dz * dz;
						if (bot.targetTimer <= 0 || distSq < 0.8 * 0.8) {
							pickBotTarget(bot, i * 101 + Math.floor(time * 10));
						}

						if (distSq > 0.0001) {
							const dist = Math.sqrt(distSq);
							const step = bot.speed * delta;
							const stepScale = Math.min(step / dist, 1);
							bot.group.position.x += dx * stepScale;
							bot.group.position.z += dz * stepScale;
							bot.group.rotation.y = Math.atan2(dx, dz);
							bot.phase += delta * 10 * (0.35 + bot.speed * 0.18);
							const gait = Math.sin(bot.phase) * 0.95 * Math.min(1, step * 2.6);
							bot.legLeft.rotation.x = gait;
							bot.legRight.rotation.x = -gait;
							bot.armLeft.rotation.x = -gait * 0.75;
							bot.armRight.rotation.x = gait * 0.75;
						} else {
							bot.legLeft.rotation.x *= 0.85;
							bot.legRight.rotation.x *= 0.85;
							bot.armLeft.rotation.x *= 0.85;
							bot.armRight.rotation.x *= 0.85;
						}

						const bx = Math.round(bot.group.position.x);
						const bz = Math.round(bot.group.position.z);
						const surface = getHeightAt(bx, bz);
						const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
						const desiredY = Math.max(surface, fluidSurface) + 0.25;
						bot.group.position.y += (desiredY - bot.group.position.y) * Math.min(1, delta * 8);
					}
				}

				cameraRig.position.copy(player.position);
				cameraRig.rotation.y = player.rotation.y;
				cameraRig.rotation.x = 0;
				cameraRig.updateMatrixWorld();

				const target = tempVec.copy(player.position).add(headOffset);
				const desiredWorld = tempVec2.copy(baseCameraOffset).applyAxisAngle(xAxis, cameraPitch);
				cameraRig.localToWorld(desiredWorld);

				const toCamera = tempVec3.copy(desiredWorld).sub(target);
				const desiredDistance = toCamera.length();
				if (desiredDistance > 0.001) {
					toCamera.divideScalar(desiredDistance);
					raycaster.set(target, toCamera);
					raycaster.far = desiredDistance;
					const hits = raycaster.intersectObjects(cameraOccluders, false);
					const collisionDistance = hits.length > 0
						? Math.max(hits[0].distance - 0.35, 1.4)
						: desiredDistance;
					const damp = collisionDistance < cameraDistance ? collisionDampIn : collisionDampOut;
					cameraDistance = THREE.MathUtils.damp(cameraDistance, collisionDistance, damp, delta);
					cameraDistance = Math.min(cameraDistance, desiredDistance);
					const adjustedWorld = tempVec2.copy(target).addScaledVector(toCamera, cameraDistance);
					camera.position.copy(cameraRig.worldToLocal(adjustedWorld));
				}

				camera.lookAt(target);

				for (const portal of portals) {
					const coreMat = portal.core.material as THREE.MeshStandardMaterial;
					coreMat.opacity = 0.6 + Math.sin(time * 2.4 + portal.pulseOffset) * 0.15;
					portal.core.rotation.z += delta * 0.6;
				}

				if (!portalTransition && portalCooldown <= 0) {
					const triggerRadius = 1.6;
					for (const portal of portals) {
						const dx = player.position.x - portal.group.position.x;
						const dz = player.position.z - portal.group.position.z;
						if (dx * dx + dz * dz <= triggerRadius * triggerRadius) {
							triggerPortal(portal);
							break;
						}
					}
				}

				spawnTimer -= delta;
				if (spawnTimer <= 0) {
					spawnTimer = THREE.MathUtils.randFloat(0.6, 1.4);
					spawnFallingBlock();
				}

				world.timestep = delta;
				world.step();
				for (const block of tntHits) {
					detonateBlock(block);
				}
				for (const block of pushHits) {
					const blockPos = block.body.translation();
					const playerPos = playerBody.translation();
					pushImpulse.set(
						blockPos.x - playerPos.x,
						0,
						blockPos.z - playerPos.z
					);
					if (pushImpulse.lengthSq() < 0.0001) {
						continue;
					}
					pushImpulse.normalize().multiplyScalar(0.25);
					block.body.applyImpulse(
						{ x: pushImpulse.x, y: 0.02, z: pushImpulse.z },
						true
					);
				}

				for (let i = fallingBlocks.length - 1; i >= 0; i -= 1) {
					const block = fallingBlocks[i];
					if (block.removed) {
						fallingBlocks.splice(i, 1);
						continue;
					}
					const position = block.body.translation();
					block.mesh.position.set(position.x, position.y, position.z);

					if (position.y < -10) {
						removeBlock(block);
						fallingBlocks.splice(i, 1);
					}
				}

				for (let i = particles.length - 1; i >= 0; i -= 1) {
					const particle = particles[i];
					particle.velocity.y += gravity * 0.35 * delta;
					particle.mesh.position.addScaledVector(particle.velocity, delta);
					particle.life -= delta;
					if (particle.life <= 0) {
						scene.remove(particle.mesh);
						particles.splice(i, 1);
					}
				}

				for (let i = portalParticles.length - 1; i >= 0; i -= 1) {
					const particle = portalParticles[i];
					particle.velocity.y += gravity * 0.15 * delta;
					particle.mesh.position.addScaledVector(particle.velocity, delta);
					particle.life -= delta;
					if (particle.life <= 0) {
						scene.remove(particle.mesh);
						portalParticles.splice(i, 1);
					}
				}

				for (let i = pendingDetonations.length - 1; i >= 0; i -= 1) {
					const pending = pendingDetonations[i];
					pending.time -= delta;
					if (pending.time > 0) {
						continue;
					}
					pendingDetonations.splice(i, 1);
					if (!pending.block.removed) {
						detonateBlock(pending.block, pending.chainLevel);
					}
				}

				renderer.render(scene, camera);
				frame = requestAnimationFrame(tick);
			};

			const handleResize = () => {
				if (!container) {
					return;
				}
				const { clientWidth, clientHeight } = container;
				camera.aspect = clientWidth / clientHeight;
				camera.updateProjectionMatrix();
				renderer.setSize(clientWidth, clientHeight);
				updateJoystickBounds();
			};

			window.addEventListener('resize', handleResize);
			handleResize();
			tick();

			dispose = () => {
				cancelAnimationFrame(frame);
				window.removeEventListener('resize', handleResize);
				window.removeEventListener('keydown', handleKeyDown);
				window.removeEventListener('keyup', handleKeyUp);
				window.removeEventListener('pointermove', handleMouseMove);
				window.removeEventListener('pointerup', handleMouseUp);
				renderer.domElement.removeEventListener('pointerdown', handleMouseDown);
				joystickEl?.removeEventListener('pointerdown', handleJoystickDown);
				joystickEl?.removeEventListener('pointermove', handleJoystickMove);
				joystickEl?.removeEventListener('pointerup', handleJoystickUp);
				joystickEl?.removeEventListener('pointercancel', handleJoystickUp);
				jumpEl?.removeEventListener('pointerdown', handleJumpDown);
				jumpEl?.removeEventListener('pointerup', handleJumpUp);
				jumpEl?.removeEventListener('pointercancel', handleJumpUp);
				renderer.domElement.removeEventListener('pointerdown', handleLookDown);
				renderer.domElement.removeEventListener('pointermove', handleLookMove);
				renderer.domElement.removeEventListener('pointerup', handleLookUp);
				renderer.domElement.removeEventListener('pointercancel', handleLookUp);
				container?.removeChild(renderer.domElement);
				worldJump = null;

				blockGeo.dispose();
				fallingBlockGeo.dispose();
				particleGeo.dispose();
				portalParticleGeo.dispose();
				portalFrameGeo.dispose();
				portalCoreGeo.dispose();
				bodyMat.dispose();
				limbMat.dispose();
				grassTopMat.dispose();
				grassSideMat.dispose();
				dirtMat.dispose();
				stoneMat.dispose();
				cobbleMat.dispose();
				woodPlankMat.dispose();
				redWoodPlankMat.dispose();
				sandstoneMat.dispose();
				obsidianMat.dispose();
				marsSandMat.dispose();
				marsRockMat.dispose();
				moonDustMat.dispose();
				sandMat.dispose();
				gravelMat.dispose();
				waterMat.dispose();
				lavaMat.dispose();
				logTopMat.dispose();
				logSideMat.dispose();
				leavesMat.dispose();
				glassMat.dispose();
				brickMat.dispose();
				doorMat.dispose();
				coalOreMat.dispose();
				ironOreMat.dispose();
				mossyCobbleMat.dispose();
				clayMat.dispose();
				snowMat.dispose();
				iceMat.dispose();
				netherrackMat.dispose();
				portalFrameMat.dispose();
				tntTopMat.dispose();
				tntSideMat.dispose();
				tntBottomMat.dispose();
				particleMat.dispose();
				portalParticleMat.dispose();
				grassTopTex.dispose();
				grassSideTex.dispose();
				dirtTex.dispose();
				stoneTex.dispose();
				cobbleTex.dispose();
				woodPlankTex.dispose();
				redWoodPlankTex.dispose();
				sandstoneTex.dispose();
				obsidianTex.dispose();
				marsSandTex.dispose();
				marsRockTex.dispose();
				moonDustTex.dispose();
				sandTex.dispose();
				gravelTex.dispose();
				waterTex.dispose();
				lavaTex.dispose();
				logTopTex.dispose();
				logSideTex.dispose();
				leavesTex.dispose();
				glassTex.dispose();
				brickTex.dispose();
				doorTex.dispose();
				coalOreTex.dispose();
				ironOreTex.dispose();
				mossyCobbleTex.dispose();
				clayTex.dispose();
				snowTex.dispose();
				iceTex.dispose();
				netherrackTex.dispose();
				tntTopTex.dispose();
				tntSideTex.dispose();
				tntBottomTex.dispose();
				(body.geometry as THREE.BufferGeometry).dispose();
				(head.geometry as THREE.BufferGeometry).dispose();
				(legLeft.geometry as THREE.BufferGeometry).dispose();
				(legRight.geometry as THREE.BufferGeometry).dispose();
				(armLeft.geometry as THREE.BufferGeometry).dispose();
				(armRight.geometry as THREE.BufferGeometry).dispose();
				botHeadGeo.dispose();
				botBodyGeo.dispose();
				botLimbGeo.dispose();
				botEyeGeo.dispose();
				botNoseGeo.dispose();
				villagerSkinMat.dispose();
				villagerEyeMat.dispose();
				for (const mat of villagerRobeMats) {
					mat.dispose();
				}

				clearChunks();
				clearPortals();
				clearBots();
				cameraOccluders.length = 0;

				for (const block of fallingBlocks) {
					scene.remove(block.mesh);
					world.removeRigidBody(block.body);
				}
				for (const particle of particles) {
					scene.remove(particle.mesh);
				}
				for (const particle of portalParticles) {
					scene.remove(particle.mesh);
				}
				pendingDetonations.length = 0;
				world.removeRigidBody(playerBody);
				world.removeCharacterController(controller);

				for (const audio of bgmCache.values()) {
					audio.pause();
				}

				renderer.dispose();
			};
		};

		init();

		return () => {
			cancelled = true;
			dispose();
		};
	});
</script>

<svelte:head>
	<title>Portal Biomes</title>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin="anonymous" />
	<link
		href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;600&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<div class="page">
	<div class="hud">
		<h1>Portal Biomes</h1>
		<div class="status">World: {worldLabel}</div>
		<div class="world-buttons">
			<button type="button" on:click={() => jumpWorld('earth')}>Earth</button>
			<button type="button" on:click={() => jumpWorld('mars')}>Mars</button>
			<button type="button" on:click={() => jumpWorld('moon')}>Moon</button>
		</div>
		<p>Walk into a portal to swap worlds. W / A / S / D or Arrow keys + mouse drag. Space to jump. Touch: left stick move, drag anywhere to look, tap Jump.</p>
	</div>
	<div class="scene" bind:this={container}></div>
	<div class="touch-controls">
		<div class="touch-pad joystick" bind:this={joystickEl}>
			<div class="thumb" bind:this={joystickThumbEl}></div>
		</div>
		<div class="touch-pad jump" bind:this={jumpEl}>Jump</div>
	</div>
</div>

<style>
	:global(html, body) {
		margin: 0;
		font-family: 'Space Grotesk', 'IBM Plex Sans', sans-serif;
		background: #0b1216;
		color: #e8f3f6;
		height: 100%;
	}

	:global(*),
	:global(*::before),
	:global(*::after) {
		box-sizing: border-box;
	}

	.page {
		min-height: 100svh;
		min-height: 100vh;
		width: 100%;
		background: radial-gradient(circle at 15% 20%, #2b404a, #0b1216 55%);
		overflow: hidden;
		position: relative;
	}

	.scene {
		position: absolute;
		inset: 0;
		touch-action: none;
	}

	.scene :global(canvas) {
		display: block;
		width: 100%;
		height: 100%;
		cursor: grab;
	}

	.scene :global(canvas:active) {
		cursor: grabbing;
	}

	.hud {
		position: absolute;
		z-index: 2;
		left: 28px;
		top: 24px;
		padding: 12px 16px;
		border-radius: 14px;
		background: rgba(12, 18, 22, 0.65);
		backdrop-filter: blur(10px);
		border: 1px solid rgba(143, 177, 185, 0.35);
		animation: hud-in 0.6s ease-out;
	}

	.hud h1 {
		margin: 0 0 6px;
		font-size: 18px;
		letter-spacing: 0.04em;
		text-transform: uppercase;
		color: #f9d18c;
	}

	.hud .status {
		font-size: 12px;
		letter-spacing: 0.18em;
		text-transform: uppercase;
		color: #b7f1ff;
		margin-bottom: 6px;
	}

	.world-buttons {
		display: flex;
		gap: 8px;
		margin: 8px 0 10px;
		flex-wrap: wrap;
	}

	.world-buttons button {
		appearance: none;
		border: 1px solid rgba(143, 177, 185, 0.35);
		background: rgba(18, 26, 32, 0.62);
		color: rgba(232, 243, 246, 0.92);
		border-radius: 999px;
		padding: 7px 12px;
		font-size: 11px;
		letter-spacing: 0.12em;
		text-transform: uppercase;
		cursor: pointer;
		transition: transform 0.12s ease, background 0.12s ease, border-color 0.12s ease;
	}

	.world-buttons button:hover {
		background: rgba(26, 38, 46, 0.72);
		border-color: rgba(183, 241, 255, 0.35);
	}

	.world-buttons button:active {
		transform: translateY(1px);
	}

	.hud p {
		margin: 0;
		font-size: 13px;
		opacity: 0.85;
	}

	.touch-controls {
		position: fixed;
		inset: 0;
		padding: env(safe-area-inset-top) env(safe-area-inset-right) env(safe-area-inset-bottom) env(safe-area-inset-left);
		pointer-events: none;
		display: none;
		z-index: 3;
	}

	.touch-pad {
		position: absolute;
		width: 96px;
		height: 96px;
		border-radius: 999px;
		border: 1px solid rgba(143, 177, 185, 0.35);
		background: rgba(8, 12, 15, 0.55);
		backdrop-filter: blur(6px);
		pointer-events: auto;
		touch-action: none;
		display: flex;
		align-items: center;
		justify-content: center;
		color: rgba(232, 243, 246, 0.7);
		font-size: 11px;
		letter-spacing: 0.16em;
		text-transform: uppercase;
	}

	.touch-pad.joystick {
		left: calc(20px + env(safe-area-inset-left));
		bottom: calc(18px + env(safe-area-inset-bottom));
	}

	.touch-pad.jump {
		right: calc(18px + env(safe-area-inset-right));
		bottom: calc(18px + env(safe-area-inset-bottom));
	}

	
	.touch-pad .thumb {
		width: 58px;
		height: 58px;
		border-radius: 999px;
		border: 1px solid rgba(143, 177, 185, 0.45);
		background: rgba(143, 177, 185, 0.2);
		transition: transform 0.05s linear;
	}

	@keyframes hud-in {
		from {
			transform: translateY(-10px);
			opacity: 0;
		}
		to {
			transform: translateY(0);
			opacity: 1;
		}
	}

	@media (max-width: 720px) {
		.hud {
			left: 16px;
			right: 16px;
		}

		.hud h1 {
			font-size: 16px;
		}
	}

	@media (pointer: coarse) {
		.touch-controls {
			display: block;
		}
	}
</style>
