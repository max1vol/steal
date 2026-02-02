<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import RAPIER from '@dimforge/rapier3d-compat';

	let container: HTMLDivElement | null = null;
	let joystickEl: HTMLDivElement | null = null;
	let joystickThumbEl: HTMLDivElement | null = null;
	let jumpEl: HTMLDivElement | null = null;
	let worldLabel = 'Verdant Expanse';

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

			const renderer = new THREE.WebGLRenderer({ antialias: true });
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
				moonDust: moonDustMats
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
						base: 4.6,
						amplitude: 3.2,
						ridgeAmp: 1.1,
						min: 2,
						max: 10,
						scale: 0.08,
						ridgeScale: 0.21
					},
					palette: {
						top: 'grass',
						sub: 'dirt',
						deep: 'stone',
						topVariants: ['cobble', 'wood', 'redwood'],
						deepVariants: ['cobble'],
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
						base: 3.8,
						amplitude: 2.6,
						ridgeAmp: 0.9,
						min: 2,
						max: 8,
						scale: 0.07,
						ridgeScale: 0.19
					},
					palette: {
						top: 'marsSand',
						sub: 'sandstone',
						deep: 'marsRock',
						topVariants: ['sandstone', 'marsRock'],
						deepVariants: ['obsidian', 'marsRock'],
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
						base: 3.2,
						amplitude: 2.4,
						ridgeAmp: 1.2,
						min: 2,
						max: 9,
						scale: 0.09,
						ridgeScale: 0.23
					},
					palette: {
						top: 'moonDust',
						sub: 'moonDust',
						deep: 'obsidian',
						topVariants: ['cobble'],
						deepVariants: ['moonDust', 'obsidian'],
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
				bodies: RAPIER.RigidBody[];
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

			const fractalNoise = (x: number, z: number, seed: number) => {
				let total = 0;
				let amplitude = 1;
				let frequency = 1;
				let max = 0;
				for (let i = 0; i < 4; i += 1) {
					const n = valueNoise(x * frequency, z * frequency, seed + i * 13) * 2 - 1;
					total += n * amplitude;
					max += amplitude;
					amplitude *= 0.5;
					frequency *= 2;
				}
				return total / max;
			};

			const computeHeight = (x: number, z: number, worldDef: WorldDefinition) => {
				const detail = fractalNoise(x * worldDef.height.scale, z * worldDef.height.scale, worldDef.seed);
				const ridge = Math.abs(
					fractalNoise(x * worldDef.height.ridgeScale, z * worldDef.height.ridgeScale, worldDef.seed + 91)
				);
				const height = worldDef.height.base + detail * worldDef.height.amplitude + ridge * worldDef.height.ridgeAmp;
				return THREE.MathUtils.clamp(Math.round(height), worldDef.height.min, worldDef.height.max);
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

			const pickBlockType = (x: number, z: number, y: number, height: number) => {
				const surfaceNoise = valueNoise(
					x * currentWorld.palette.variantScale,
					z * currentWorld.palette.variantScale,
					currentWorld.seed + 181
				);
				if (y === height - 1) {
					return pickVariant(currentWorld.palette.topVariants, surfaceNoise, currentWorld.palette.top);
				}
				if (y >= height - 3) {
					return currentWorld.palette.sub;
				}
				const deepNoise = valueNoise(
					x * currentWorld.palette.variantScale * 0.8,
					z * currentWorld.palette.variantScale * 0.8,
					currentWorld.seed + 419
				);
				return pickVariant(currentWorld.palette.deepVariants, deepNoise, currentWorld.palette.deep);
			};

			const getHeightAt = (x: number, z: number) => computeHeight(x, z, currentWorld);

			const buildChunk = (cx: number, cz: number) => {
				const key = `${cx},${cz}`;
				if (chunks.has(key)) {
					return;
				}
				const positionsByType: Record<BlockType, number[]> = {} as Record<BlockType, number[]>;
				for (const type of blockTypeKeys) {
					positionsByType[type] = [];
				}
				const bodies: RAPIER.RigidBody[] = [];
				for (let ix = 0; ix < chunkSize; ix += 1) {
					for (let iz = 0; iz < chunkSize; iz += 1) {
						const worldX = cx * chunkSize + ix;
						const worldZ = cz * chunkSize + iz;
						const height = computeHeight(worldX, worldZ, currentWorld);
						for (let y = 0; y < height; y += 1) {
							const blockType = pickBlockType(worldX, worldZ, y, height);
							positionsByType[blockType].push(worldX, y + 0.5, worldZ);
						}
						const columnBody = world.createRigidBody(
							RAPIER.RigidBodyDesc.fixed().setTranslation(worldX, height / 2, worldZ)
						);
						world.createCollider(
							RAPIER.ColliderDesc.cuboid(0.5, height / 2, 0.5),
							columnBody
						);
						bodies.push(columnBody);
					}
				}

				const meshes: THREE.InstancedMesh[] = [];
				for (const type of blockTypeKeys) {
					const positions = positionsByType[type];
					if (!positions.length) {
						continue;
					}
					const mesh = new THREE.InstancedMesh(blockGeo, blockMats[type], positions.length / 3);
					for (let i = 0; i < positions.length; i += 3) {
						chunkMatrix.makeTranslation(positions[i], positions[i + 1], positions[i + 2]);
						mesh.setMatrixAt(i / 3, chunkMatrix);
					}
					mesh.instanceMatrix.needsUpdate = true;
					scene.add(mesh);
					cameraOccluders.push(mesh);
					meshes.push(mesh);
				}

				chunks.set(key, { key, x: cx, z: cz, meshes, bodies });
			};

			const removeChunk = (chunk: Chunk) => {
				for (const mesh of chunk.meshes) {
					scene.remove(mesh);
					const occIndex = cameraOccluders.indexOf(mesh);
					if (occIndex >= 0) {
						cameraOccluders.splice(occIndex, 1);
					}
				}
				for (const body of chunk.bodies) {
					world.removeRigidBody(body);
				}
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

			const startHeight = getHeightAt(0, 0);
			player.position.set(0, startHeight, 0);
			const playerBody = world.createRigidBody(
				RAPIER.RigidBodyDesc.kinematicPositionBased().setTranslation(0, startHeight, 0)
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
					const y = getHeightAt(x, z) + 1.2;

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
				if (resetPlayer) {
					const spawnHeight = getHeightAt(0, 0);
					playerBody.setNextKinematicTranslation({ x: 0, y: spawnHeight, z: 0 });
					player.position.set(0, spawnHeight, 0);
					verticalVelocity = 0;
					grounded = false;
				}
				syncChunks(player.position.x, player.position.z, true);
				portalCooldown = 1.1;
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
				tntTopTex.dispose();
				tntSideTex.dispose();
				tntBottomTex.dispose();
				(body.geometry as THREE.BufferGeometry).dispose();
				(head.geometry as THREE.BufferGeometry).dispose();
				(legLeft.geometry as THREE.BufferGeometry).dispose();
				(legRight.geometry as THREE.BufferGeometry).dispose();
				(armLeft.geometry as THREE.BufferGeometry).dispose();
				(armRight.geometry as THREE.BufferGeometry).dispose();

				clearChunks();
				clearPortals();
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
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
	<link
		href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;600&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<div class="page">
	<div class="hud">
		<h1>Portal Biomes</h1>
		<div class="status">World: {worldLabel}</div>
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
