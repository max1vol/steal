	<script lang="ts">
		import { onMount } from 'svelte';
		import * as THREE from 'three';
		import RAPIER from '@dimforge/rapier3d-compat';

		type BlockType =
			| 'grass'
			| 'dirt'
			| 'stone'
			| 'cobble'
			| 'wood'
			| 'redwood'
			| 'sandstone'
			| 'obsidian'
			| 'marsSand'
			| 'marsRock'
			| 'moonDust'
			| 'sand'
			| 'gravel'
			| 'water'
			| 'lava'
			| 'log'
			| 'leaves'
			| 'glass'
			| 'brick'
			| 'door'
			| 'coalOre'
			| 'ironOre'
			| 'mossyCobble'
			| 'clay'
			| 'snow'
			| 'ice'
			| 'netherrack';

		const HOTBAR_SLOTS = 9;

		let blockIcons: Partial<Record<BlockType, string>> = {};
		let inventory: Partial<Record<BlockType, number>> = {};
		let hotbar: Array<BlockType | null> = Array.from({ length: HOTBAR_SLOTS }, () => null);
		let selectedHotbar = 0;

		type Rarity = 'common' | 'uncommon' | 'rare' | 'epic' | 'legendary';

		const RARITY_LABEL: Record<Rarity, string> = {
			common: 'Common',
			uncommon: 'Uncommon',
			rare: 'Rare',
			epic: 'Epic',
			legendary: 'Legendary'
		};

		const RARITY_COLOR: Record<Rarity, string> = {
			common: '#c9d2d6',
			uncommon: '#54e3a3',
			rare: '#6ab6ff',
			epic: '#ff6bd6',
			legendary: '#ffd26a'
		};

		const RARITY_MULTIPLIER: Record<Rarity, number> = {
			common: 1,
			uncommon: 1.7,
			rare: 3.2,
			epic: 6.2,
			legendary: 12
		};

		const MACHINERY_CATALOG = [
			{ id: 'duneRover', name: 'Dune Rover', category: 'transport', rarity: 'uncommon', baseCost: 60 },
			{ id: 'gravBike', name: 'Grav Bike', category: 'transport', rarity: 'rare', baseCost: 80 },
			{ id: 'orbitalSkiff', name: 'Orbital Skiff', category: 'ship', rarity: 'epic', baseCost: 120 },
			{ id: 'deepSpaceShuttle', name: 'Deep Space Shuttle', category: 'ship', rarity: 'legendary', baseCost: 160 },
			{ id: 'spaceSuitMk1', name: 'Spacesuit Mk I', category: 'equipment', rarity: 'uncommon', baseCost: 55 },
			{ id: 'spaceSuitMk4', name: 'Spacesuit Mk IV', category: 'equipment', rarity: 'epic', baseCost: 95 }
		] as const satisfies ReadonlyArray<{
			id: string;
			name: string;
			category: 'ship' | 'transport' | 'equipment';
			rarity: Rarity;
			baseCost: number;
		}>;

		type MachineryId = (typeof MACHINERY_CATALOG)[number]['id'];
		const MACHINERY_BY_ID = new Map(MACHINERY_CATALOG.map((entry) => [entry.id, entry] as const));

		const CARD_CATALOG = [
			{ id: 'ticketGreen', name: 'Ticket: Verdant Line', rarity: 'common' },
			{ id: 'ticketRust', name: 'Ticket: Rust Dunes', rarity: 'uncommon' },
			{ id: 'ticketLunar', name: 'Ticket: Lunar Pass', rarity: 'rare' },
			{ id: 'ticketVoid', name: 'Ticket: Void Sigil', rarity: 'epic' }
		] as const satisfies ReadonlyArray<{ id: string; name: string; rarity: Rarity }>;

		type CardId = (typeof CARD_CATALOG)[number]['id'];
		const CARD_BY_ID = new Map(CARD_CATALOG.map((entry) => [entry.id, entry] as const));

		const GAME_STATE_STORAGE_KEY = 'steal:game-state-v1';

		let crypto = 320;
		const STARTUP_GRANT_SC = 320;
		let startupGrantClaimed = false;
		let ownedMachinery: Partial<Record<MachineryId, number>> = {};
		let collectedCards: Partial<Record<CardId, number>> = {};

		let hudToast: { text: string; tone: 'info' | 'warn' | 'error' } | null = null;
			let interactionHint: string | null = null;
			let interactionActionLabel: string | null = null;
			let interactionAltActionLabel: string | null = null;
			let canDisembark = false;
			let interactRequested = false;
			let altInteractRequested = false;
			let disembarkRequested = false;

		const getOwnedMachineryCount = (id: MachineryId) => ownedMachinery[id] ?? 0;
		const getCollectedCardCount = (id: CardId) => collectedCards[id] ?? 0;
		const getMachineryCost = (id: MachineryId) => {
			const def = MACHINERY_BY_ID.get(id);
			if (!def) return 0;
			return Math.max(0, Math.round(def.baseCost * RARITY_MULTIPLIER[def.rarity]));
		};
		const formatCrypto = (amount: number) => Math.max(0, Math.floor(amount)).toLocaleString();

		const getInventoryCount = (type: BlockType) => inventory[type] ?? 0;
		const getBlockIcon = (type: BlockType) => blockIcons[type] ?? '';

		const requestInteract = () => {
			interactRequested = true;
		};

		const requestAltInteract = () => {
			altInteractRequested = true;
		};

		const requestDisembark = () => {
			disembarkRequested = true;
		};

		const addToInventory = (type: BlockType, amount = 1) => {
			const nextCount = (inventory[type] ?? 0) + amount;
			inventory = { ...inventory, [type]: nextCount };
			if (!hotbar.includes(type)) {
				const emptyIndex = hotbar.findIndex((slot) => slot === null);
				if (emptyIndex >= 0) {
					hotbar = hotbar.map((slot, idx) => (idx === emptyIndex ? type : slot));
				} else {
					hotbar = hotbar.map((slot, idx) => (idx === selectedHotbar ? type : slot));
				}
			}
		};

		const consumeFromInventory = (type: BlockType, amount = 1) => {
			const current = inventory[type] ?? 0;
			if (current < amount) {
				return false;
			}
			const next = { ...inventory } as Partial<Record<BlockType, number>>;
			const nextCount = current - amount;
			if (nextCount > 0) {
				next[type] = nextCount;
			} else {
				delete next[type];
				hotbar = hotbar.map((slot) => (slot === type ? null : slot));
			}
			inventory = next;
			return true;
		};

			let container: HTMLDivElement | null = null;
			let joystickEl: HTMLDivElement | null = null;
			let joystickThumbEl: HTMLDivElement | null = null;
			let jumpEl: HTMLDivElement | null = null;
			let downEl: HTMLDivElement | null = null;
			let worldLabel = 'Verdant Expanse';
			let worldJump: ((id: 'earth' | 'mars' | 'moon') => void) | null = null;
			let deployMachinery: ((id: MachineryId) => void) | null = null;

			const jumpWorld = (id: 'earth' | 'mars' | 'moon') => worldJump?.(id);
			const deployOwned = (id: MachineryId) => deployMachinery?.(id);

		const DEBUG_CAMERA_STORAGE_KEY = 'steal:debug-camera-v1';
		const DEBUG_CAMERA_DEFAULT = { x: 2.4, y: 2.4, z: 7.6 }; // relative to player head-top

		let debugCameraEnabled = false;
		let debugCameraOffsetX = DEBUG_CAMERA_DEFAULT.x;
		let debugCameraOffsetY = DEBUG_CAMERA_DEFAULT.y;
		let debugCameraOffsetZ = DEBUG_CAMERA_DEFAULT.z;

		const loadDebugCameraSettings = () => {
			if (typeof localStorage === 'undefined') {
				return;
			}
			try {
				const raw = localStorage.getItem(DEBUG_CAMERA_STORAGE_KEY);
				if (!raw) {
					return;
				}
				const parsed = JSON.parse(raw) as Partial<Record<'x' | 'y' | 'z', unknown>> | null;
				if (parsed && typeof parsed.x === 'number') {
					debugCameraOffsetX = parsed.x;
				}
				if (parsed && typeof parsed.y === 'number') {
					debugCameraOffsetY = parsed.y;
				}
				if (parsed && typeof parsed.z === 'number') {
					debugCameraOffsetZ = parsed.z;
				}
			} catch {
				// ignore
			}
		};

		const saveDebugCameraSettings = () => {
			if (!debugCameraEnabled || typeof localStorage === 'undefined') {
				return;
			}
			try {
				localStorage.setItem(
					DEBUG_CAMERA_STORAGE_KEY,
					JSON.stringify({ x: debugCameraOffsetX, y: debugCameraOffsetY, z: debugCameraOffsetZ })
				);
			} catch {
				// ignore
			}
		};

		const resetDebugCamera = () => {
			debugCameraOffsetX = DEBUG_CAMERA_DEFAULT.x;
			debugCameraOffsetY = DEBUG_CAMERA_DEFAULT.y;
			debugCameraOffsetZ = DEBUG_CAMERA_DEFAULT.z;
			saveDebugCameraSettings();
		};

		let toastTimeout: ReturnType<typeof setTimeout> | null = null;

		const showToast = (text: string, tone: 'info' | 'warn' | 'error' = 'info') => {
			hudToast = { text, tone };
			if (toastTimeout) {
				clearTimeout(toastTimeout);
				toastTimeout = null;
			}
			if (typeof window !== 'undefined') {
				toastTimeout = setTimeout(() => {
					hudToast = null;
					toastTimeout = null;
				}, 2800);
			}
		};

		const loadGameState = () => {
			if (typeof localStorage === 'undefined') {
				return;
			}
			try {
				const raw = localStorage.getItem(GAME_STATE_STORAGE_KEY);
				if (!raw) {
					startupGrantClaimed = true;
					return;
				}
				const parsed = JSON.parse(raw) as Partial<Record<string, unknown>> | null;
				if (!parsed || typeof parsed !== 'object') {
					return;
				}
				startupGrantClaimed = Boolean(parsed.startupGrantClaimed);
				if (typeof parsed.crypto === 'number' && Number.isFinite(parsed.crypto)) {
					crypto = Math.max(0, Math.floor(parsed.crypto));
				}
				const machineryIds = new Set<string>(MACHINERY_CATALOG.map((entry) => entry.id));
				if (parsed.ownedMachinery && typeof parsed.ownedMachinery === 'object') {
					const next: Partial<Record<MachineryId, number>> = {};
					for (const [key, value] of Object.entries(parsed.ownedMachinery as Record<string, unknown>)) {
						if (!machineryIds.has(key)) continue;
						if (typeof value !== 'number' || !Number.isFinite(value) || value <= 0) continue;
						next[key as MachineryId] = Math.floor(value);
					}
					ownedMachinery = next;
				}
				const cardIds = new Set<string>(CARD_CATALOG.map((entry) => entry.id));
				if (parsed.collectedCards && typeof parsed.collectedCards === 'object') {
					const next: Partial<Record<CardId, number>> = {};
					for (const [key, value] of Object.entries(parsed.collectedCards as Record<string, unknown>)) {
						if (!cardIds.has(key)) continue;
						if (typeof value !== 'number' || !Number.isFinite(value) || value <= 0) continue;
						next[key as CardId] = Math.floor(value);
					}
					collectedCards = next;
				}

				// One-time grant so players can meaningfully rent/buy machinery even on older saves.
				if (!startupGrantClaimed && crypto < STARTUP_GRANT_SC) {
					crypto = STARTUP_GRANT_SC;
					startupGrantClaimed = true;
					saveGameState();
				}
			} catch {
				// ignore
			}
		};

		const saveGameState = () => {
			if (typeof localStorage === 'undefined') {
				return;
			}
			try {
				localStorage.setItem(
					GAME_STATE_STORAGE_KEY,
					JSON.stringify({
						v: 2,
						crypto,
						startupGrantClaimed,
						ownedMachinery,
						collectedCards
					})
				);
			} catch {
				// ignore
			}
		};

		onMount(() => {
			const urlParams = new URLSearchParams(window.location.search);
			if (urlParams.has('debugCamera')) {
				loadDebugCameraSettings();
				debugCameraEnabled = true;
			}
			loadGameState();

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

				const camera = new THREE.PerspectiveCamera(70, 1, 0.1, 420);
				camera.rotation.order = 'YXZ';

				const cameraRig = new THREE.Group();
				cameraRig.rotation.order = 'YXZ';
				cameraRig.add(camera);
				scene.add(cameraRig);

				let cameraPitch = 0;
				const minPitch = -1.55;
				const maxPitch = 1.55;

				// First-person view model (hands + held block), created once materials are ready.
				let viewModelRoot: THREE.Group | null = null;
				let viewArmRoot: THREE.Group | null = null;
				let viewHeldItem: THREE.Mesh | null = null;
				let viewHeldType: BlockType | null = null;
				let viewSkinMat: THREE.MeshStandardMaterial | null = null;
				let viewSleeveMat: THREE.MeshStandardMaterial | null = null;

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

				const iconFromTexture = (texture: THREE.Texture) => {
					const image = texture.image;
					if (image instanceof HTMLCanvasElement) {
						return image.toDataURL();
					}
					return '';
				};

				blockIcons = {
					grass: '/textures/grass_top.png',
					dirt: '/textures/dirt.png',
					stone: '/textures/stone.png',
					cobble: '/textures/cobble.png',
					wood: '/textures/wood_plank.png',
					redwood: '/textures/red_wood_plank.png',
					sandstone: '/textures/sandstone.png',
					obsidian: '/textures/obsidian.png',
					marsSand: '/textures/mars_sand.png',
					marsRock: '/textures/mars_rock.png',
					moonDust: '/textures/moon_dust.png',
					sand: iconFromTexture(sandTex),
					gravel: iconFromTexture(gravelTex),
					water: iconFromTexture(waterTex),
					lava: iconFromTexture(lavaTex),
					log: iconFromTexture(logSideTex),
					leaves: iconFromTexture(leavesTex),
					glass: iconFromTexture(glassTex),
					brick: iconFromTexture(brickTex),
					door: iconFromTexture(doorTex),
					coalOre: iconFromTexture(coalOreTex),
					ironOre: iconFromTexture(ironOreTex),
					mossyCobble: iconFromTexture(mossyCobbleTex),
					clay: iconFromTexture(clayTex),
					snow: iconFromTexture(snowTex),
					ice: iconFromTexture(iceTex),
					netherrack: iconFromTexture(netherrackTex)
				};

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
					const portalFrameGeo = new THREE.BoxGeometry(1, 1, 1);
					const portalCoreGeo = new THREE.PlaneGeometry(2, 3);

					const blockOutlineMat = new THREE.LineBasicMaterial({
						color: 0x050607,
						transparent: true,
						opacity: 0.9,
						depthTest: true
					});
					blockOutlineMat.depthWrite = false;
					const blockOutline = new THREE.LineSegments(
						new THREE.EdgesGeometry(new THREE.BoxGeometry(1.002, 1.002, 1.002)),
						blockOutlineMat
					);
					blockOutline.visible = false;
					blockOutline.renderOrder = 10;
					scene.add(blockOutline);

					const breakStageTextures = Array.from({ length: 10 }, (_, stage) => {
						return createCanvasTexture((ctx, size) => {
							ctx.clearRect(0, 0, size, size);
							ctx.lineCap = 'round';
							ctx.lineJoin = 'round';
							ctx.strokeStyle = `rgba(0, 0, 0, ${0.12 + stage * 0.065})`;
							ctx.lineWidth = 1.5;
							const cracks = 7 + stage * 7;
							for (let i = 0; i < cracks; i += 1) {
								const x0 = Math.random() * size;
								const y0 = Math.random() * size;
								const x1 = x0 + (Math.random() * 2 - 1) * (size * (0.25 + stage * 0.02));
								const y1 = y0 + (Math.random() * 2 - 1) * (size * (0.25 + stage * 0.02));
								ctx.beginPath();
								ctx.moveTo(x0, y0);
								ctx.lineTo(x1, y1);
								ctx.stroke();
								if (stage >= 3 && Math.random() < 0.55) {
									const bt = 0.55 + Math.random() * 0.25;
									const bx = x0 + (x1 - x0) * bt;
									const by = y0 + (y1 - y0) * bt;
									const b2x = bx + (Math.random() * 2 - 1) * (size * 0.16);
									const b2y = by + (Math.random() * 2 - 1) * (size * 0.16);
									ctx.beginPath();
									ctx.moveTo(bx, by);
									ctx.lineTo(b2x, b2y);
									ctx.stroke();
								}
							}
						});
					});

					const blockBreakMat = new THREE.MeshBasicMaterial({
						map: breakStageTextures[0],
						transparent: true,
						opacity: 0,
						side: THREE.DoubleSide,
						depthTest: true,
						depthWrite: false
					});
					blockBreakMat.polygonOffset = true;
					blockBreakMat.polygonOffsetFactor = -1;
					blockBreakMat.polygonOffsetUnits = -1;
					const blockBreakOverlay = new THREE.Mesh(new THREE.BoxGeometry(1.01, 1.01, 1.01), blockBreakMat);
					blockBreakOverlay.visible = false;
					blockBreakOverlay.renderOrder = 9;
					scene.add(blockBreakOverlay);

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

				{
					viewSkinMat = new THREE.MeshStandardMaterial({ color: 0xffd2a1, roughness: 0.75 });
					viewSleeveMat = new THREE.MeshStandardMaterial({ color: 0x334856, roughness: 0.9 });
					for (const mat of [viewSkinMat, viewSleeveMat]) {
						mat.depthTest = false;
						mat.depthWrite = false;
					}

					viewModelRoot = new THREE.Group();
					viewModelRoot.name = 'viewModel';
					camera.add(viewModelRoot);

					viewArmRoot = new THREE.Group();
					viewArmRoot.position.set(0.68, -0.74, -1.05);
					viewArmRoot.rotation.set(-0.55, 0.62, 0.18);
					viewModelRoot.add(viewArmRoot);

					const sleeve = new THREE.Mesh(blockGeo, viewSleeveMat);
					sleeve.scale.set(0.22, 0.62, 0.22);
					sleeve.position.set(0, -0.18, 0);
					sleeve.renderOrder = 1000;
					sleeve.frustumCulled = false;
					viewArmRoot.add(sleeve);

					const hand = new THREE.Mesh(blockGeo, viewSkinMat);
					hand.scale.set(0.22, 0.22, 0.22);
					hand.position.set(0, -0.53, -0.02);
					hand.renderOrder = 1000;
					hand.frustumCulled = false;
					viewArmRoot.add(hand);

					viewHeldItem = new THREE.Mesh(blockGeo, blockMats.grass);
					viewHeldItem.scale.setScalar(0.34);
					viewHeldItem.position.set(-0.16, -0.58, -0.05);
					viewHeldItem.rotation.set(0.25, 0.4, 0.1);
					viewHeldItem.renderOrder = 1000;
					viewHeldItem.frustumCulled = false;
					viewHeldItem.visible = false;
					viewArmRoot.add(viewHeldItem);
				}

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

				type WorldId = WorldDefinition['id'];

					type WorldEdits = {
						removed: Set<string>;
						added: Map<string, BlockType>;
					};

				const worldEdits = new Map<WorldId, WorldEdits>();
					const getWorldEdits = (worldId: WorldId) => {
						const existing = worldEdits.get(worldId);
						if (existing) return existing;
						const edits: WorldEdits = { removed: new Set(), added: new Map() };
						worldEdits.set(worldId, edits);
						return edits;
					};

				const blockKey = (x: number, y: number, z: number) => `${x},${y},${z}`;

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
							base: 6.8,
							amplitude: 9.4,
							ridgeAmp: 3.4,
							min: 2,
							max: 24,
							scale: 0.055,
							ridgeScale: 0.19,
							riverScale: 0.03,
							riverWidth: 0.26,
							riverDepth: 5
						},
						fluids: {
							waterLevel: 8,
							lavaLevel: 4,
							lavaScale: 0.045,
							lavaThreshold: 0.89,
							lavaCarveDepth: 3
						},
						vegetation: {
							treeDensity: 0.065,
							treeScale: 0.09,
							treeMinHeight: 4,
							treeMaxHeight: 8
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
				const mineableMeshes: THREE.Object3D[] = [];
				const blockTypeKeys = Object.keys(blockMats) as BlockType[];
				const chunkMatrix = new THREE.Matrix4();

			type Chunk = {
				key: string;
				x: number;
				z: number;
				meshes: THREE.InstancedMesh[];
				body: RAPIER.RigidBody;
				blocks: Map<string, BlockType>;
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

					if (
						worldDef.id === 'earth' &&
						worldDef.fluids.waterLevel > 0 &&
						biome.id !== 'ocean' &&
						biome.id !== 'beach' &&
						biome.id !== 'desert' &&
						biome.id !== 'mountains' &&
						biome.id !== 'volcanic'
					) {
						const dist = Math.hypot(x, z);
						const spawnSafety = smoothstep(18, 85, dist);
						const lakeField = valueNoise(warpedX * 0.008, warpedZ * 0.008, worldDef.seed + 5555);
						const lakeStrength = smoothstep(0.82, 0.93, lakeField) * spawnSafety;
						const lowlands =
							1 - smoothstep(worldDef.fluids.waterLevel + 6, worldDef.fluids.waterLevel + 14, height);
						height -= lakeStrength * lowlands * 5.5;
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
						const edits = getWorldEdits(currentWorld.id);
						const removedBlocks = edits.removed;
						const addedBlocks = edits.added;

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
				const blockAt = new Map<string, BlockType>();
				const setBlockAt = (x: number, y: number, z: number, type: BlockType) => {
					const k = blockKey(x, y, z);
					const prev = blockAt.get(k);
					if (!prev) {
						blockAt.set(k, type);
						return;
					}
					const prevFluid = prev === 'water' || prev === 'lava';
					const nextFluid = type === 'water' || type === 'lava';
					// Prefer non-fluids over fluids so a solid at the same cell "wins" for occupancy checks.
					if (prevFluid && !nextFluid) {
						blockAt.set(k, type);
						return;
					}
					if (!prevFluid && nextFluid) {
						return;
					}
					blockAt.set(k, type);
				};

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
							const key = blockKey(x, y, z);
							if (addedBlocks.has(key)) {
								return;
							}
							if (removedBlocks.has(key)) {
								return;
							}
							setBlockAt(x, y, z, type);
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

				type BasePlan = {
					x: number;
					z: number;
					width: number;
					depth: number;
					wallHeight: number;
					roofHeight: number;
					facing: 0 | 1 | 2 | 3;
					baseY: number;
					doorX: number;
					doorZ: number;
				};

				const computeBaseY = (bx: number, bz: number, w: number, d: number) => {
					let minH = Number.POSITIVE_INFINITY;
					let maxH = Number.NEGATIVE_INFINITY;
					for (let dx = 0; dx < w; dx += 1) {
						for (let dz = 0; dz < d; dz += 1) {
							const h = getHeightAt(bx + dx, bz + dz);
							minH = Math.min(minH, h);
							maxH = Math.max(maxH, h);
						}
					}
					// This base is meant to be big and dramatic, so allow slightly rougher ground than houses.
					if (maxH - minH > 3) {
						return -1;
					}
					return maxH;
				};

				const emitBase = (plan: BasePlan) => {
					const baseX = plan.x;
					const baseZ = plan.z;
					const y0 = plan.baseY;

					const floorMat: BlockType = 'obsidian';
					const wallMat: BlockType = 'brick';
					const frameMat: BlockType = 'obsidian';
					const windowMat: BlockType = 'glass';
					const accentMat: BlockType = 'ice';

					for (let dx = 0; dx < plan.width; dx += 1) {
						for (let dz = 0; dz < plan.depth; dz += 1) {
							const x = baseX + dx;
							const z = baseZ + dz;
							if (x < chunkMinX || x >= chunkMinX + chunkSize || z < chunkMinZ || z >= chunkMinZ + chunkSize) {
								continue;
							}
							const idx = (x - chunkMinX) * chunkSize + (z - chunkMinZ);
							noPlants[idx] = 1;
							flatTarget[idx] = Math.max(flatTarget[idx], y0);
							const surface = getHeightAt(x, z);
							if (surface < y0) {
								foundationStart[idx] = foundationStart[idx] === -1 ? surface : Math.min(foundationStart[idx], surface);
								foundationMaterial[idx] = frameMat;
							}
							topOverride[idx] = floorMat;
						}
					}

					const wallTop = y0 + plan.wallHeight;
					const isWall = (dx: number, dz: number) =>
						dx === 0 || dz === 0 || dx === plan.width - 1 || dz === plan.depth - 1;

					const doorSpan = 4; // larger than villager doors
					const doorHalf = Math.floor(doorSpan / 2);
					const doorXs = plan.facing === 0 || plan.facing === 2
						? [plan.doorX - doorHalf, plan.doorX - doorHalf + 1, plan.doorX - doorHalf + 2, plan.doorX - doorHalf + 3]
						: [plan.doorX];
					const doorZs = plan.facing === 1 || plan.facing === 3
						? [plan.doorZ - doorHalf, plan.doorZ - doorHalf + 1, plan.doorZ - doorHalf + 2, plan.doorZ - doorHalf + 3]
						: [plan.doorZ];

					const isDoorCell = (x: number, z: number) => {
						if (plan.facing === 0 || plan.facing === 2) {
							return z === plan.doorZ && doorXs.includes(x);
						}
						return x === plan.doorX && doorZs.includes(z);
					};

					for (let dy = 0; dy < plan.wallHeight; dy += 1) {
						const y = y0 + dy;
						for (let dx = 0; dx < plan.width; dx += 1) {
							for (let dz = 0; dz < plan.depth; dz += 1) {
								if (!isWall(dx, dz)) continue;
								const x = baseX + dx;
								const z = baseZ + dz;

								const isCorner = (dx === 0 || dx === plan.width - 1) && (dz === 0 || dz === plan.depth - 1);
								if (isDoorCell(x, z) && dy < 3) {
									continue;
								}

								const windowRow = dy === 2 || dy === 3;
								const canWindow = windowRow && !isCorner && !isDoorCell(x, z);
								const supportEvery = 3;
								const onSupport = (dx % supportEvery === 0) || (dz % supportEvery === 0);
								if (canWindow && !onSupport) {
									emitSolidBlock(x, y, z, windowMat, true);
									continue;
								}
								emitSolidBlock(x, y, z, isCorner ? frameMat : wallMat, true);
							}
						}
					}

					// Massive entrance.
					for (const x of doorXs) {
						for (const z of doorZs) {
							for (let dy = 0; dy < 3; dy += 1) {
								emitSolidBlock(x, y0 + dy, z, 'door', false);
							}
							emitSolidBlock(x, y0 + 3, z, frameMat, true);
						}
					}

					// Roof: obsidian rim + glass skylight + icy crown.
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
								emitSolidBlock(x, y, z, frameMat, true);
							}
						}
					}

					const skylightInset = Math.min(3, Math.floor(Math.min(plan.width, plan.depth) / 5));
					const skylightY = wallTop + plan.roofHeight - 1;
					for (let dx = skylightInset; dx < plan.width - skylightInset; dx += 1) {
						for (let dz = skylightInset; dz < plan.depth - skylightInset; dz += 1) {
							const x = baseX + dx;
							const z = baseZ + dz;
							const rim =
								dx === skylightInset ||
								dz === skylightInset ||
								dx === plan.width - skylightInset - 1 ||
								dz === plan.depth - skylightInset - 1;
							emitSolidBlock(x, skylightY, z, rim ? accentMat : windowMat, true);
						}
					}

					// Beacon spire.
					const cx = baseX + Math.floor(plan.width / 2);
					const cz = baseZ + Math.floor(plan.depth / 2);
					for (let dy = 0; dy < 7; dy += 1) {
						const y = skylightY + 1 + dy;
						emitSolidBlock(cx, y, cz, dy < 5 ? windowMat : accentMat, true);
						if (dy % 2 === 0) {
							emitSolidBlock(cx + 1, y, cz, frameMat, true);
							emitSolidBlock(cx - 1, y, cz, frameMat, true);
							emitSolidBlock(cx, y, cz + 1, frameMat, true);
							emitSolidBlock(cx, y, cz - 1, frameMat, true);
						}
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

							let basePlan: BasePlan | null = null;
							if (isSpawnVillage) {
								const baseWidth = 18;
								const baseDepth = 20;
								const wallHeight = 7;
								const roofHeight = 4;
								const candidates: Array<{ ox: number; oz: number }> = [
									{ ox: 28, oz: 18 },
									{ ox: -30, oz: 16 },
									{ ox: 22, oz: -30 },
									{ ox: -28, oz: -26 }
								];
								for (const candidate of candidates) {
									const cx = centerX + candidate.ox;
									const cz = centerZ + candidate.oz;
									const bx = cx - Math.floor(baseWidth / 2);
									const bz = cz - Math.floor(baseDepth / 2);
									const by = computeBaseY(bx, bz, baseWidth, baseDepth);
									if (by < 0) continue;
									if (waterLevel > 0 && by < waterLevel) continue;

									const toCenterX = centerX - cx;
									const toCenterZ = centerZ - cz;
									const facing: 0 | 1 | 2 | 3 =
										Math.abs(toCenterX) > Math.abs(toCenterZ)
											? (toCenterX > 0 ? 3 : 1)
											: (toCenterZ > 0 ? 0 : 2);
									const doorX =
										facing === 0 ? bx + Math.floor(baseWidth / 2)
											: facing === 2 ? bx + Math.floor(baseWidth / 2)
												: facing === 1 ? bx + baseWidth - 1
													: bx;
									const doorZ =
										facing === 1 ? bz + Math.floor(baseDepth / 2)
											: facing === 3 ? bz + Math.floor(baseDepth / 2)
												: facing === 0 ? bz
													: bz + baseDepth - 1;
									basePlan = {
										x: bx,
										z: bz,
										width: baseWidth,
										depth: baseDepth,
										wallHeight,
										roofHeight,
										facing,
										baseY: by,
										doorX,
										doorZ
									};
									break;
								}
							}
							if (basePlan) {
								emitBase(basePlan);
								const pathStartX = basePlan.doorX + (basePlan.facing === 1 ? 1 : basePlan.facing === 3 ? -1 : 0);
								const pathStartZ = basePlan.doorZ + (basePlan.facing === 2 ? 1 : basePlan.facing === 0 ? -1 : 0);
								drawPath(pathStartX, pathStartZ, centerX, centerZ, roadMat);
							}

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
								if (basePlan) {
									const margin = 2;
									const baseMinX = basePlan.x - margin;
									const baseMaxX = basePlan.x + basePlan.width - 1 + margin;
									const baseMinZ = basePlan.z - margin;
									const baseMaxZ = basePlan.z + basePlan.depth - 1 + margin;
									const houseMinX = hx;
									const houseMaxX = hx + w - 1;
									const houseMinZ = hz;
									const houseMaxZ = hz + d - 1;
									const overlaps = houseMinX <= baseMaxX && houseMaxX >= baseMinX && houseMinZ <= baseMaxZ && houseMaxZ >= baseMinZ;
									if (overlaps) return;
								}
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

							const terrainColliderLayers = 3;
							const topLayerStart = Math.max(0, height - terrainColliderLayers);
							if (height > 0 && topLayerStart > 0) {
								world.createCollider(
									RAPIER.ColliderDesc.cuboid(0.5, topLayerStart / 2, 0.5).setTranslation(ix, topLayerStart / 2, iz),
									chunkBody
								);
							}

								for (let y = 0; y < height; y += 1) {
									const cellKey = blockKey(worldX, y, worldZ);
									if (addedBlocks.has(cellKey) || removedBlocks.has(cellKey)) {
										continue;
									}
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
								setBlockAt(worldX, y, worldZ, blockType);
								if (y >= topLayerStart && isSolidBlockType(blockType)) {
									world.createCollider(
										RAPIER.ColliderDesc.cuboid(0.5, 0.5, 0.5).setTranslation(ix, y + 0.5, iz),
										chunkBody
									);
								}
							}

							if (useLava) {
								for (let y = height; y < lavaLevel; y += 1) {
									if (addedBlocks.has(blockKey(worldX, y, worldZ))) {
										continue;
									}
									positionsByType.lava.push(worldX, y + 0.5, worldZ);
									setBlockAt(worldX, y, worldZ, 'lava');
								}
							} else if (waterLevel > 0 && height < waterLevel) {
								const maxFill = info.biome.id === 'swamp' ? waterLevel + 1 : waterLevel;
								for (let y = height; y < maxFill; y += 1) {
									if (addedBlocks.has(blockKey(worldX, y, worldZ))) {
										continue;
									}
									positionsByType.water.push(worldX, y + 0.5, worldZ);
									setBlockAt(worldX, y, worldZ, 'water');
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
									if (addedBlocks.has(blockKey(worldX, y, worldZ))) {
										continue;
									}
									positionsByType.water.push(worldX, y + 0.5, worldZ);
									setBlockAt(worldX, y, worldZ, 'water');
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
									if (!addedBlocks.has(blockKey(worldX, height, worldZ))) {
									positionsByType.lava.push(worldX, height + 0.5, worldZ);
									setBlockAt(worldX, height, worldZ, 'lava');
									}
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

							}
						}

					for (const [key, type] of addedBlocks.entries()) {
						const parts = key.split(',');
						if (parts.length !== 3) continue;
						const ax = Number(parts[0]);
						const ay = Number(parts[1]);
						const az = Number(parts[2]);
						if (!Number.isFinite(ax) || !Number.isFinite(ay) || !Number.isFinite(az)) continue;
						const x = Math.trunc(ax);
						const y = Math.trunc(ay);
						const z = Math.trunc(az);
						if (x < chunkMinX || x >= chunkMinX + chunkSize || z < chunkMinZ || z >= chunkMinZ + chunkSize) {
							continue;
						}
						if (y <= 0) {
							continue;
						}
						setBlockAt(x, y, z, type);
						positionsByType[type].push(x, y + 0.5, z);
						if (isSolidBlockType(type)) {
							solidBlockColliders.push(x, y + 0.5, z);
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
						mesh.userData = { blockType: type, chunkKey: key };
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
						if (type !== 'water' && type !== 'lava') {
							mineableMeshes.push(mesh);
						}
						if (type !== 'water' && type !== 'door') {
							cameraOccluders.push(mesh);
						}
						meshes.push(mesh);
					}

				chunks.set(key, { key, x: cx, z: cz, meshes, body: chunkBody, blocks: blockAt });
			};

				const removeChunk = (chunk: Chunk) => {
					for (const mesh of chunk.meshes) {
						scene.remove(mesh);
						const mineIndex = mineableMeshes.indexOf(mesh);
						if (mineIndex >= 0) {
							mineableMeshes.splice(mineIndex, 1);
						}
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

				const rebuildChunkAt = (worldX: number, worldZ: number) => {
					const cx = Math.floor(worldX / chunkSize);
					const cz = Math.floor(worldZ / chunkSize);
					const key = `${cx},${cz}`;
					const existing = chunks.get(key);
					if (existing) {
						removeChunk(existing);
						chunks.delete(key);
					}
					buildChunk(cx, cz);
				};

				const getLoadedBlockType = (x: number, y: number, z: number) => {
					const cx = Math.floor(x / chunkSize);
					const cz = Math.floor(z / chunkSize);
					const chunk = chunks.get(`${cx},${cz}`);
					if (!chunk) return null;
					return chunk.blocks.get(blockKey(x, y, z)) ?? null;
				};

				type BlockTarget = {
					x: number;
					y: number;
					z: number;
					type: BlockType;
					distance: number;
				};

				const isMineableType = (type: BlockType) => type !== 'water' && type !== 'lava';

				const blockBreakSeconds: Record<BlockType, number> = {
					grass: 0.38,
					dirt: 0.38,
					stone: 0.95,
					cobble: 0.85,
					wood: 0.65,
					redwood: 0.7,
					sandstone: 0.75,
					obsidian: 1.25,
					marsSand: 0.4,
					marsRock: 0.9,
					moonDust: 0.4,
					sand: 0.35,
					gravel: 0.45,
					water: 999,
					lava: 999,
					log: 0.75,
					leaves: 0.18,
					glass: 0.32,
					brick: 0.85,
					door: 0.45,
					coalOre: 1.05,
					ironOre: 1.1,
					mossyCobble: 0.9,
					clay: 0.5,
					snow: 0.16,
					ice: 0.55,
					netherrack: 0.7
				};

				const CRYPTO_REWARD_BY_BLOCK: Partial<Record<BlockType, number>> = {
					coalOre: 4,
					ironOre: 6,
					obsidian: 9,
					netherrack: 5
				};

				const addCrypto = (amount: number) => {
					const next = Math.max(0, Math.floor(crypto + amount));
					if (next === crypto) return;
					crypto = next;
					saveGameState();
				};

				const spendCrypto = (amount: number) => {
					const cost = Math.max(0, Math.floor(amount));
					if (crypto < cost) {
						return false;
					}
					crypto = crypto - cost;
					saveGameState();
					return true;
				};

					const breakTargetBlock = (target: BlockTarget) => {
						if (!isMineableType(target.type)) {
							return false;
						}
						if (target.y <= 0) {
							return false;
						}
						const edits = getWorldEdits(currentWorld.id);
						const key = blockKey(target.x, target.y, target.z);
						if (edits.added.has(key)) {
							edits.added.delete(key);
						} else {
							if (edits.removed.has(key)) {
								return false;
							}
							edits.removed.add(key);
						}
						addToInventory(target.type, 1);
						const reward = CRYPTO_REWARD_BY_BLOCK[target.type] ?? 0;
						if (reward > 0) {
							addCrypto(reward);
						}
						rebuildChunkAt(target.x, target.z);
						return true;
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
				player.visible = false;

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

				type CritterKind = 'sheep' | 'cow' | 'chicken' | 'spider' | 'crab';

				type Critter = {
					kind: CritterKind;
					group: THREE.Group;
					legs: THREE.Mesh[];
					wings?: THREE.Mesh[];
					home: THREE.Vector2;
					target: THREE.Vector2;
					speed: number;
					phase: number;
					targetTimer: number;
				};

				const critters: Critter[] = [];
				const sheepWoolMat = new THREE.MeshStandardMaterial({ color: 0xf6f6f6, roughness: 1 });
				const sheepSkinMat = new THREE.MeshStandardMaterial({ color: 0xffc6c6, roughness: 0.95 });
				const cowHideMat = new THREE.MeshStandardMaterial({ color: 0x2a1f1b, roughness: 0.95 });
				const cowSpotMat = new THREE.MeshStandardMaterial({ color: 0xf2f2f2, roughness: 0.95 });
				const cowSnoutMat = new THREE.MeshStandardMaterial({ color: 0xd98989, roughness: 0.9 });
				const chickenFeatherMat = new THREE.MeshStandardMaterial({ color: 0xf7f7f7, roughness: 0.98 });
				const chickenBeakMat = new THREE.MeshStandardMaterial({ color: 0xf0b241, roughness: 0.85 });
				const chickenCombMat = new THREE.MeshStandardMaterial({ color: 0xcf3a2f, roughness: 0.8 });
				const spiderMat = new THREE.MeshStandardMaterial({ color: 0x1a1b1d, roughness: 0.9 });
				const spiderEyeMat = new THREE.MeshStandardMaterial({
					color: 0xff3a2b,
					emissive: 0xff3a2b,
					emissiveIntensity: 0.75,
					roughness: 0.4
				});
				const crabMat = new THREE.MeshStandardMaterial({ color: 0xd8683a, roughness: 0.85 });

				const placeCritterOnSurface = (group: THREE.Group, x: number, z: number) => {
					const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
					const ground = Math.max(getHeightAt(x, z), fluidSurface) + 0.25;
					group.position.set(x, ground, z);
				};

				const createSheep = (seed: number, homeX: number, homeZ: number): Critter => {
					const group = new THREE.Group();

					const body = new THREE.Mesh(blockGeo, sheepWoolMat);
					body.scale.set(0.95, 0.65, 0.55);
					body.position.set(0, 0.9, 0);
					group.add(body);

					const head = new THREE.Mesh(blockGeo, sheepWoolMat);
					head.scale.set(0.45, 0.45, 0.45);
					head.position.set(0, 1.12, 0.62);
					group.add(head);

					const face = new THREE.Mesh(blockGeo, sheepSkinMat);
					face.scale.set(0.34, 0.26, 0.18);
					face.position.set(0, -0.05, 0.34);
					head.add(face);

					const legs: THREE.Mesh[] = [];
					const legOffsets: Array<[number, number]> = [
						[-0.32, 0.2],
						[0.32, 0.2],
						[-0.32, -0.2],
						[0.32, -0.2]
					];
					for (const [lx, lz] of legOffsets) {
						const leg = new THREE.Mesh(blockGeo, sheepSkinMat);
						leg.scale.set(0.18, 0.5, 0.18);
						leg.position.set(lx, 0.25, lz);
						group.add(leg);
						legs.push(leg);
					}

					placeCritterOnSurface(group, homeX, homeZ);

					return {
						kind: 'sheep',
						group,
						legs,
						home: new THREE.Vector2(homeX, homeZ),
						target: new THREE.Vector2(homeX, homeZ),
						speed: 1.55 + (hash2D(seed, seed * 11, currentWorld.seed + 7701) - 0.5) * 0.35,
						phase: hash2D(seed * 5, seed * 19, currentWorld.seed + 7711) * Math.PI * 2,
						targetTimer: 0
					};
				};

				const createCow = (seed: number, homeX: number, homeZ: number): Critter => {
					const group = new THREE.Group();

					const body = new THREE.Mesh(blockGeo, cowHideMat);
					body.scale.set(1.15, 0.72, 0.6);
					body.position.set(0, 0.95, 0);
					group.add(body);

					const spotLeft = new THREE.Mesh(blockGeo, cowSpotMat);
					spotLeft.scale.set(0.35, 0.32, 0.08);
					spotLeft.position.set(-0.55, 0.1, 0.05);
					body.add(spotLeft);
					const spotRight = spotLeft.clone();
					spotRight.position.x = 0.55;
					body.add(spotRight);

					const head = new THREE.Mesh(blockGeo, cowHideMat);
					head.scale.set(0.55, 0.55, 0.62);
					head.position.set(0, 1.1, 0.78);
					group.add(head);

					const snout = new THREE.Mesh(blockGeo, cowSnoutMat);
					snout.scale.set(0.38, 0.26, 0.2);
					snout.position.set(0, -0.06, 0.38);
					head.add(snout);

					const hornLeft = new THREE.Mesh(blockGeo, cowSpotMat);
					hornLeft.scale.set(0.1, 0.1, 0.1);
					hornLeft.position.set(-0.22, 0.28, 0.18);
					head.add(hornLeft);
					const hornRight = hornLeft.clone();
					hornRight.position.x = 0.22;
					head.add(hornRight);

					const legs: THREE.Mesh[] = [];
					const legOffsets: Array<[number, number]> = [
						[-0.42, 0.22],
						[0.42, 0.22],
						[-0.42, -0.22],
						[0.42, -0.22]
					];
					for (const [lx, lz] of legOffsets) {
						const leg = new THREE.Mesh(blockGeo, cowHideMat);
						leg.scale.set(0.2, 0.58, 0.2);
						leg.position.set(lx, 0.29, lz);
						group.add(leg);
						legs.push(leg);
					}

					placeCritterOnSurface(group, homeX, homeZ);

					return {
						kind: 'cow',
						group,
						legs,
						home: new THREE.Vector2(homeX, homeZ),
						target: new THREE.Vector2(homeX, homeZ),
						speed: 1.35 + (hash2D(seed, seed * 11, currentWorld.seed + 7751) - 0.5) * 0.25,
						phase: hash2D(seed * 9, seed * 17, currentWorld.seed + 7761) * Math.PI * 2,
						targetTimer: 0
					};
				};

				const createChicken = (seed: number, homeX: number, homeZ: number): Critter => {
					const group = new THREE.Group();

					const body = new THREE.Mesh(blockGeo, chickenFeatherMat);
					body.scale.set(0.52, 0.46, 0.58);
					body.position.set(0, 0.72, 0);
					group.add(body);

					const head = new THREE.Mesh(blockGeo, chickenFeatherMat);
					head.scale.set(0.32, 0.32, 0.32);
					head.position.set(0, 0.95, 0.42);
					group.add(head);

					const beak = new THREE.Mesh(blockGeo, chickenBeakMat);
					beak.scale.set(0.16, 0.12, 0.14);
					beak.position.set(0, -0.02, 0.24);
					head.add(beak);

					const comb = new THREE.Mesh(blockGeo, chickenCombMat);
					comb.scale.set(0.12, 0.18, 0.12);
					comb.position.set(0, 0.22, 0);
					head.add(comb);

					const wings: THREE.Mesh[] = [];
					const wingLeft = new THREE.Mesh(blockGeo, chickenFeatherMat);
					wingLeft.scale.set(0.16, 0.28, 0.04);
					wingLeft.position.set(-0.34, 0.04, 0.05);
					body.add(wingLeft);
					wings.push(wingLeft);
					const wingRight = wingLeft.clone();
					wingRight.position.x = 0.34;
					body.add(wingRight);
					wings.push(wingRight);

					const legs: THREE.Mesh[] = [];
					const legOffsets: Array<[number, number]> = [
						[-0.14, 0.08],
						[0.14, 0.08]
					];
					for (const [lx, lz] of legOffsets) {
						const leg = new THREE.Mesh(blockGeo, chickenBeakMat);
						leg.scale.set(0.08, 0.34, 0.08);
						leg.position.set(lx, 0.17, lz);
						group.add(leg);
						legs.push(leg);
					}

					placeCritterOnSurface(group, homeX, homeZ);

					return {
						kind: 'chicken',
						group,
						legs,
						wings,
						home: new THREE.Vector2(homeX, homeZ),
						target: new THREE.Vector2(homeX, homeZ),
						speed: 1.75 + (hash2D(seed, seed * 3, currentWorld.seed + 7771) - 0.5) * 0.35,
						phase: hash2D(seed * 2, seed * 29, currentWorld.seed + 7781) * Math.PI * 2,
						targetTimer: 0
					};
				};

				const createSpider = (seed: number, homeX: number, homeZ: number): Critter => {
					const group = new THREE.Group();

					const abdomen = new THREE.Mesh(blockGeo, spiderMat);
					abdomen.scale.set(0.92, 0.22, 0.92);
					abdomen.position.set(0, 0.36, -0.08);
					group.add(abdomen);

					const head = new THREE.Mesh(blockGeo, spiderMat);
					head.scale.set(0.72, 0.22, 0.72);
					head.position.set(0, 0.36, 0.62);
					group.add(head);

					const eyeLeft = new THREE.Mesh(blockGeo, spiderEyeMat);
					eyeLeft.scale.set(0.08, 0.08, 0.02);
					eyeLeft.position.set(-0.16, 0.06, 0.37);
					head.add(eyeLeft);
					const eyeRight = eyeLeft.clone();
					eyeRight.position.x = 0.16;
					head.add(eyeRight);

					const legs: THREE.Mesh[] = [];
					const side = [-1, 1] as const;
					for (const s of side) {
						for (let i = 0; i < 4; i += 1) {
							const leg = new THREE.Mesh(blockGeo, spiderMat);
							leg.scale.set(0.78, 0.07, 0.07);
							leg.position.set(s * 0.62, 0.28, -0.2 + i * 0.33);
							leg.rotation.y = s * 0.35;
							leg.rotation.z = s * (0.55 + i * 0.06);
							leg.userData.baseZ = leg.rotation.z;
							leg.userData.phase = i * 0.8 + (s < 0 ? 0.4 : 0);
							group.add(leg);
							legs.push(leg);
						}
					}

					placeCritterOnSurface(group, homeX, homeZ);

					return {
						kind: 'spider',
						group,
						legs,
						home: new THREE.Vector2(homeX, homeZ),
						target: new THREE.Vector2(homeX, homeZ),
						speed: 2.15 + (hash2D(seed, seed * 7, currentWorld.seed + 7801) - 0.5) * 0.55,
						phase: hash2D(seed * 3, seed * 17, currentWorld.seed + 7811) * Math.PI * 2,
						targetTimer: 0
					};
				};

				const createCrab = (seed: number, homeX: number, homeZ: number): Critter => {
					const group = new THREE.Group();

					const body = new THREE.Mesh(blockGeo, crabMat);
					body.scale.set(0.62, 0.2, 0.62);
					body.position.set(0, 0.28, 0);
					group.add(body);

					const clawLeft = new THREE.Mesh(blockGeo, crabMat);
					clawLeft.scale.set(0.18, 0.12, 0.32);
					clawLeft.position.set(-0.5, 0.28, 0.15);
					group.add(clawLeft);
					const clawRight = clawLeft.clone();
					clawRight.position.x = 0.5;
					group.add(clawRight);

					const legs: THREE.Mesh[] = [];
					for (const s of [-1, 1] as const) {
						for (let i = 0; i < 3; i += 1) {
							const leg = new THREE.Mesh(blockGeo, crabMat);
							leg.scale.set(0.32, 0.06, 0.06);
							leg.position.set(s * 0.42, 0.22, -0.2 + i * 0.2);
							leg.rotation.y = s * 0.7;
							leg.userData.baseZ = 0;
							leg.userData.phase = i * 0.9 + (s < 0 ? 0.4 : 0);
							group.add(leg);
							legs.push(leg);
						}
					}

					placeCritterOnSurface(group, homeX, homeZ);

					return {
						kind: 'crab',
						group,
						legs,
						home: new THREE.Vector2(homeX, homeZ),
						target: new THREE.Vector2(homeX, homeZ),
						speed: 1.75 + (hash2D(seed * 5, seed * 3, currentWorld.seed + 7901) - 0.5) * 0.35,
						phase: hash2D(seed * 2, seed * 13, currentWorld.seed + 7911) * Math.PI * 2,
						targetTimer: 0
					};
				};

				const critterBiomeOk = (kind: CritterKind, biomeId: BiomeId) => {
					if (currentWorld.id === 'moon') {
						return false;
					}
					if (currentWorld.id === 'mars') {
						return kind === 'spider' && (biomeId === 'mountains' || biomeId === 'desert' || biomeId === 'volcanic');
					}
					if (kind === 'crab') return biomeId === 'beach';
					if (kind === 'sheep') return biomeId === 'plains' || biomeId === 'forest' || biomeId === 'mountains';
					if (kind === 'cow') return biomeId === 'plains' || biomeId === 'forest';
					if (kind === 'chicken') return biomeId === 'plains' || biomeId === 'forest' || biomeId === 'beach';
					if (kind === 'spider') return biomeId === 'plains' || biomeId === 'forest' || biomeId === 'swamp' || biomeId === 'mountains';
					return false;
				};

				const pickCritterTarget = (critter: Critter, seed: number) => {
					const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
					for (let attempt = 0; attempt < 12; attempt += 1) {
						const a = hash2D(seed + attempt * 19, seed - attempt * 7, currentWorld.seed + 9400) * Math.PI * 2;
						const rBase = critter.kind === 'crab' ? 2.5 : 4.5;
						const rSpan = critter.kind === 'crab' ? 8 : 18;
						const r = rBase + hash2D(seed + attempt * 11, seed + attempt * 23, currentWorld.seed + 9401) * rSpan;
						const tx = Math.round(critter.home.x + Math.cos(a) * r);
						const tz = Math.round(critter.home.y + Math.sin(a) * r);
						const info = getTerrainInfo(tx, tz, currentWorld);
						if (!critterBiomeOk(critter.kind, info.biome.id)) {
							continue;
						}
						if (info.lavaStrength > 0.22) {
							continue;
						}
						if (critter.kind !== 'crab' && currentWorld.fluids.waterLevel > 0 && info.height < currentWorld.fluids.waterLevel) {
							continue;
						}
						if (info.biome.id === 'ocean') {
							continue;
						}

						let minH = Number.POSITIVE_INFINITY;
						let maxH = Number.NEGATIVE_INFINITY;
						for (let ox = -1; ox <= 1; ox += 1) {
							for (let oz = -1; oz <= 1; oz += 1) {
								const h = computeHeight(tx + ox, tz + oz, currentWorld);
								minH = Math.min(minH, h);
								maxH = Math.max(maxH, h);
							}
						}
						const slopeLimit = critter.kind === 'crab' ? 2 : 3;
						if (maxH - minH > slopeLimit) {
							continue;
						}

						critter.target.set(tx, tz);
						critter.targetTimer = 1.4 + hash2D(tx, tz, currentWorld.seed + 9500) * 3.2;
						critter.group.position.y = Math.max(critter.group.position.y, Math.max(info.height, fluidSurface) + 0.25);
						return;
					}
					critter.target.copy(critter.home);
					critter.targetTimer = 1.6;
				};

				const clearCritters = () => {
					for (const critter of critters) {
						scene.remove(critter.group);
					}
					critters.length = 0;
				};

				const spawnCrittersForWorld = () => {
					clearCritters();
					if (currentWorld.id === 'moon') {
						return;
					}
					const anchor = spawnAnchors[currentWorld.id];
					const used = new Set<string>();
					const desiredCount = currentWorld.id === 'earth' ? 18 : 10;

					let seed = 0;
					let created = 0;
					for (let attempts = 0; attempts < desiredCount * 80 && created < desiredCount; attempts += 1) {
						const a = hash2D(seed * 7 + attempts, seed * 11 - attempts * 3, currentWorld.seed + 9600) * Math.PI * 2;
						const r = 10 + hash2D(seed * 13 + attempts * 5, seed * 19 - attempts * 2, currentWorld.seed + 9601) * 52;
						const x = Math.round(anchor.x + Math.cos(a) * r);
						const z = Math.round(anchor.z + Math.sin(a) * r);
						const key = `${x},${z}`;
						if (used.has(key)) {
							continue;
						}

						const info = getTerrainInfo(x, z, currentWorld);
						if (info.lavaStrength > 0.22) {
							continue;
						}
						if (currentWorld.fluids.waterLevel > 0 && info.height < currentWorld.fluids.waterLevel && info.biome.id !== 'beach') {
							continue;
						}
						if (info.biome.id === 'ocean') {
							continue;
						}

						let kind: CritterKind = 'sheep';
						const roll = hash2D(x, z, currentWorld.seed + 9700);
						if (currentWorld.id === 'mars') {
							kind = 'spider';
						} else {
							switch (info.biome.id) {
								case 'beach':
									kind = roll < 0.7 ? 'crab' : 'chicken';
									break;
								case 'plains':
									kind = roll < 0.42 ? 'sheep'
										: roll < 0.72 ? 'cow'
											: roll < 0.93 ? 'chicken'
												: 'spider';
									break;
								case 'forest':
									kind = roll < 0.38 ? 'sheep'
										: roll < 0.62 ? 'cow'
											: roll < 0.9 ? 'chicken'
												: 'spider';
									break;
								case 'swamp':
									kind = 'spider';
									break;
								case 'mountains':
									kind = roll < 0.85 ? 'spider' : 'sheep';
									break;
								default:
									kind = roll < 0.6 ? 'sheep' : 'spider';
									break;
							}
						}
						if (!critterBiomeOk(kind, info.biome.id)) {
							continue;
						}

						used.add(key);
						const critter =
							kind === 'sheep'
								? createSheep(seed, x, z)
								: kind === 'cow'
									? createCow(seed, x, z)
									: kind === 'chicken'
										? createChicken(seed, x, z)
								: kind === 'crab'
									? createCrab(seed, x, z)
									: createSpider(seed, x, z);
						critters.push(critter);
						scene.add(critter.group);
						pickCritterTarget(critter, seed + 31);
						seed += 1;
						created += 1;
					}
				};

				type MachineryDef = (typeof MACHINERY_CATALOG)[number];

				type VehicleMode = 'hover' | 'flight' | 'jetpack';

				type MachineryParts = {
					rotors: THREE.Object3D[];
					wheels: THREE.Object3D[];
					gyros: THREE.Object3D[];
					arms: THREE.Object3D[];
					thrusters: THREE.Object3D[];
					thrusterPoints: THREE.Vector3[]; // local-space emission points for particles
					seatHeight: number; // camera anchor (relative to group origin)
					interactRadius: number;
				};

				type MachineryInstance = {
					def: MachineryDef;
					group: THREE.Group;
					label: THREE.Sprite;
					home: THREE.Vector2;
					target: THREE.Vector2;
					speed: number;
					phase: number;
					targetTimer: number;
					life: number;
					hover: number;
					parts: MachineryParts;
					isOwned: boolean;
					boarded: boolean;
					rentalRemaining: number; // seconds; 0 when inactive, Infinity for owned
					rentalCost: number;
					rentalDuration: number;
					desiredHover: number; // hover height above surface when boarded (hover mode)
					lastPos: THREE.Vector3;
					wheelRoll: number;
				};

				const machineries: MachineryInstance[] = [];
				let activeMachine: MachineryInstance | null = null;
				let deployedOwnedMachine: MachineryInstance | null = null;
				const machineryLabelTextures = new Map<MachineryId, THREE.Texture>();
				const machineryLabelMats = new Map<MachineryId, THREE.SpriteMaterial>();

				const cardTextures = new Map<CardId, THREE.Texture>();
				const cardMats = new Map<CardId, THREE.SpriteMaterial>();

				const rarityAccentMats = new Map<Rarity, THREE.MeshStandardMaterial>();
				for (const rarity of Object.keys(RARITY_LABEL) as Rarity[]) {
					const color = new THREE.Color(RARITY_COLOR[rarity]);
					const mat = new THREE.MeshStandardMaterial({
						color,
						emissive: color,
						emissiveIntensity: 0.85,
						roughness: 0.3,
						metalness: 0.35
					});
					rarityAccentMats.set(rarity, mat);
				}

				const machineryHullMat = new THREE.MeshStandardMaterial({
					color: 0x1a232b,
					roughness: 0.35,
					metalness: 0.35
				});
				const machineryDetailMat = new THREE.MeshStandardMaterial({
					color: 0x394851,
					roughness: 0.6,
					metalness: 0.2
				});
				const machineryGlassMat = new THREE.MeshStandardMaterial({
					color: 0xa8e7ff,
					roughness: 0.1,
					metalness: 0.15,
					transparent: true,
					opacity: 0.72,
					depthWrite: false
				});

				const getRarityAccentMat = (rarity: Rarity) => rarityAccentMats.get(rarity) ?? machineryDetailMat;

				const createMachineryLabelTexture = (def: MachineryDef) => {
					const canvas = document.createElement('canvas');
					canvas.width = 420;
					canvas.height = 96;
					const ctx = canvas.getContext('2d');
					if (ctx) {
						ctx.imageSmoothingEnabled = false;
						ctx.clearRect(0, 0, canvas.width, canvas.height);
						ctx.fillStyle = 'rgba(8, 12, 16, 0.75)';
						ctx.fillRect(0, 0, canvas.width, canvas.height);
						ctx.strokeStyle = RARITY_COLOR[def.rarity];
						ctx.lineWidth = 6;
						ctx.strokeRect(8, 8, canvas.width - 16, canvas.height - 16);
						ctx.fillStyle = '#fef3dd';
						ctx.font = 'bold 30px "Space Grotesk", sans-serif';
						ctx.textAlign = 'center';
						ctx.textBaseline = 'middle';
						const purchaseCost = getMachineryCost(def.id);
						const rent = getRentalOffer(def);
						ctx.fillText(`${def.name}`, canvas.width / 2, 38);
						ctx.fillStyle = 'rgba(183, 241, 255, 0.85)';
						ctx.font = 'bold 16px "Space Grotesk", sans-serif';
						ctx.fillText(
							`RENT ${rent.cost} SC / ${rent.duration}s  •  BUY ${purchaseCost} SC  •  ${RARITY_LABEL[def.rarity].toUpperCase()}`,
							canvas.width / 2,
							70
						);
					}
					const texture = new THREE.CanvasTexture(canvas);
					texture.colorSpace = THREE.SRGBColorSpace;
					texture.magFilter = THREE.NearestFilter;
					texture.minFilter = THREE.NearestMipMapNearestFilter;
					return texture;
				};

				const getMachineryLabelMat = (def: MachineryDef) => {
					const cachedMat = machineryLabelMats.get(def.id);
					if (cachedMat) {
						return cachedMat;
					}
					const texture = createMachineryLabelTexture(def);
					machineryLabelTextures.set(def.id, texture);
					const mat = new THREE.SpriteMaterial({
						map: texture,
						transparent: true,
						depthTest: false
					});
					machineryLabelMats.set(def.id, mat);
					return mat;
				};

				const createTicketTexture = (card: (typeof CARD_CATALOG)[number]) => {
					const canvas = document.createElement('canvas');
					canvas.width = 384;
					canvas.height = 256;
					const ctx = canvas.getContext('2d');
					if (ctx) {
						ctx.imageSmoothingEnabled = false;
						ctx.clearRect(0, 0, canvas.width, canvas.height);

						const rarityColor = RARITY_COLOR[card.rarity];
						ctx.fillStyle = 'rgba(10, 16, 20, 0.85)';
						ctx.fillRect(0, 0, canvas.width, canvas.height);

						ctx.fillStyle = 'rgba(249, 209, 140, 0.12)';
						ctx.fillRect(20, 24, canvas.width - 40, canvas.height - 48);

						ctx.strokeStyle = rarityColor;
						ctx.lineWidth = 8;
						ctx.strokeRect(20, 24, canvas.width - 40, canvas.height - 48);

						// Perforation line.
						ctx.strokeStyle = 'rgba(232, 243, 246, 0.18)';
						ctx.lineWidth = 3;
						ctx.setLineDash([8, 8]);
						ctx.beginPath();
						ctx.moveTo(38, canvas.height * 0.62);
						ctx.lineTo(canvas.width - 38, canvas.height * 0.62);
						ctx.stroke();
						ctx.setLineDash([]);

						ctx.fillStyle = '#fef3dd';
						ctx.font = 'bold 30px "Space Grotesk", sans-serif';
						ctx.textAlign = 'center';
						ctx.textBaseline = 'middle';
						ctx.fillText('TICKET', canvas.width / 2, 72);

						ctx.fillStyle = 'rgba(183, 241, 255, 0.9)';
						ctx.font = 'bold 20px "Space Grotesk", sans-serif';
						ctx.fillText(RARITY_LABEL[card.rarity].toUpperCase(), canvas.width / 2, 106);

						ctx.fillStyle = '#fef3dd';
						ctx.font = 'bold 22px "Space Grotesk", sans-serif';
						ctx.fillText(card.name.replace('Ticket: ', ''), canvas.width / 2, 158);

						ctx.fillStyle = 'rgba(232, 243, 246, 0.65)';
						ctx.font = 'bold 16px "Space Grotesk", sans-serif';
						ctx.fillText('COLLECTABLE', canvas.width / 2, 206);
					}
					const texture = new THREE.CanvasTexture(canvas);
					texture.colorSpace = THREE.SRGBColorSpace;
					texture.magFilter = THREE.NearestFilter;
					texture.minFilter = THREE.NearestMipMapNearestFilter;
					return texture;
				};

				const getCardMat = (card: (typeof CARD_CATALOG)[number]) => {
					const cached = cardMats.get(card.id);
					if (cached) {
						return cached;
					}
					const texture = createTicketTexture(card);
					cardTextures.set(card.id, texture);
					const mat = new THREE.SpriteMaterial({
						map: texture,
						transparent: true,
						depthTest: true
					});
					cardMats.set(card.id, mat);
					return mat;
				};

				const weightedPick = <T,>(items: readonly T[], weight: (item: T) => number) => {
					let total = 0;
					for (const item of items) {
						total += Math.max(0, weight(item));
					}
					if (total <= 0) {
						return items[0] ?? null;
					}
					let roll = Math.random() * total;
					for (const item of items) {
						roll -= Math.max(0, weight(item));
						if (roll <= 0) {
							return item;
						}
					}
					return items[items.length - 1] ?? null;
				};

				const raritySpawnWeight = (rarity: Rarity) => {
					switch (rarity) {
						case 'common':
							return 1;
						case 'uncommon':
							return 0.55;
						case 'rare':
							return 0.22;
						case 'epic':
							return 0.09;
						case 'legendary':
							return 0.03;
						default:
							return 0.15;
					}
				};

					const clearMachinery = () => {
						for (const machine of machineries) {
							scene.remove(machine.group);
						}
						machineries.length = 0;
						activeMachine = null;
						deployedOwnedMachine = null;
					};

				const getMachinerySpeed = (def: MachineryDef) => {
					if (def.category === 'ship') return 3.1;
					if (def.category === 'transport') return 2.35;
					return 1.6;
				};

				const getMachineryHover = (def: MachineryDef) => {
					if (def.category === 'ship') return 4.8;
					if (def.category === 'transport') return 1.4;
					return 0.9;
				};

				const getVehicleMode = (def: MachineryDef): VehicleMode => {
					if (def.category === 'ship') return 'flight';
					if (def.category === 'transport') return 'hover';
					return 'jetpack';
				};

				const getRentalOffer = (def: MachineryDef) => {
					const purchaseCost = getMachineryCost(def.id);
					const base = Math.max(12, Math.round(purchaseCost * 0.18));
					const duration =
						def.category === 'ship' ? 120
							: def.category === 'transport' ? 100
								: 90;
					return { cost: base, duration };
				};

				const formatClock = (seconds: number) => {
					const s = Math.max(0, Math.floor(seconds));
					const m = Math.floor(s / 60);
					const r = s % 60;
					return `${m}:${String(r).padStart(2, '0')}`;
				};

				const getVehicleDriveSpeed = (def: MachineryDef) => {
					// These are deliberately higher than NPC wander speeds.
					const rarityMul = 0.9 + (RARITY_MULTIPLIER[def.rarity] - 1) * 0.06;
					if (def.category === 'ship') return 14 * rarityMul;
					if (def.category === 'transport') return 8 * rarityMul;
					return 6.5 * rarityMul;
				};

				const getVehicleVerticalSpeed = (def: MachineryDef) => {
					if (def.category === 'ship') return 9.5;
					if (def.category === 'transport') return 4.5;
					return 6.5;
				};

				const getVehicleMaxAltitudeAboveSurface = (def: MachineryDef) => {
					if (def.category === 'ship') return 110;
					if (def.category === 'transport') return 6.5;
					return 18;
				};

				const createMachineryModel = (def: MachineryDef) => {
					const group = new THREE.Group();
					const accent = getRarityAccentMat(def.rarity);
					const parts: MachineryParts = {
						rotors: [],
						wheels: [],
						gyros: [],
						arms: [],
						thrusters: [],
						thrusterPoints: [],
						seatHeight: 2.2,
						interactRadius: 4.2
					};

					const addThruster = (parent: THREE.Object3D, localPoint: THREE.Vector3, size = 0.35) => {
						const thruster = new THREE.Mesh(blockGeo, machineryDetailMat);
						thruster.scale.set(size * 0.9, size * 0.9, size * 0.9);
						thruster.position.copy(localPoint);
						parent.add(thruster);
						parts.thrusters.push(thruster);

						const glow = new THREE.Mesh(blockGeo, accent);
						glow.scale.set(size * 0.45, size * 0.45, size * 0.25);
						glow.position.set(0, 0, -size * 0.9);
						thruster.add(glow);

						// Particles emit behind the thruster; local to the *vehicle* group.
						const emit = localPoint.clone();
						emit.z -= size * 1.2;
						parts.thrusterPoints.push(emit);
					};

					if (def.id === 'orbitalSkiff' || def.id === 'deepSpaceShuttle') {
						parts.seatHeight = 3.25;
						parts.interactRadius = 6.3;

						const hull = new THREE.Mesh(blockGeo, machineryHullMat);
						hull.scale.set(8.6, 1.35, 3.6);
						hull.position.set(0, 2.35, 0.1);
						group.add(hull);

						const nose = new THREE.Mesh(blockGeo, machineryDetailMat);
						nose.scale.set(3.2, 0.75, 2.4);
						nose.position.set(0, 2.45, 2.35);
						group.add(nose);

						const cockpit = new THREE.Mesh(blockGeo, machineryGlassMat);
						cockpit.scale.set(2.6, 0.95, 1.9);
						cockpit.position.set(0, 2.95, 1.8);
						group.add(cockpit);

						for (const s of [-1, 1] as const) {
							const wing = new THREE.Mesh(blockGeo, machineryDetailMat);
							wing.scale.set(3.9, 0.22, 1.65);
							wing.position.set(s * 6.05, 2.15, 0.2);
							group.add(wing);

							const tip = new THREE.Mesh(blockGeo, accent);
							tip.scale.set(0.7, 0.18, 0.7);
							tip.position.set(s * 1.95, 0.05, -0.2);
							wing.add(tip);
						}

						const fin = new THREE.Mesh(blockGeo, accent);
						fin.scale.set(0.5, 2.9, 0.5);
						fin.position.set(0, 4.3, -1.5);
						group.add(fin);

						// Rotating “gravity ring”
						const ring = new THREE.Group();
						ring.position.set(0, 2.55, 0.1);
						const ringCount = 14;
						for (let i = 0; i < ringCount; i += 1) {
							const angle = (i / ringCount) * Math.PI * 2;
							const cube = new THREE.Mesh(blockGeo, accent);
							cube.scale.set(0.35, 0.35, 0.35);
							cube.position.set(Math.cos(angle) * 4.6, Math.sin(angle) * 2.0, 0);
							ring.add(cube);
						}
						group.add(ring);
						parts.gyros.push(ring);

						// Engines (rear)
						const engineY = 2.05;
						const engineZ = -2.6;
						addThruster(group, new THREE.Vector3(-2.4, engineY, engineZ), 0.55);
						addThruster(group, new THREE.Vector3(2.4, engineY, engineZ), 0.55);
						if (def.id === 'deepSpaceShuttle') {
							addThruster(group, new THREE.Vector3(0, engineY + 0.2, engineZ - 0.35), 0.62);
						}
					} else if (def.id === 'duneRover') {
						parts.seatHeight = 2.05;
						parts.interactRadius = 4.9;

						const chassis = new THREE.Mesh(blockGeo, machineryHullMat);
						chassis.scale.set(5.2, 0.85, 3.0);
						chassis.position.set(0, 1.25, 0);
						group.add(chassis);

						const cab = new THREE.Mesh(blockGeo, machineryGlassMat);
						cab.scale.set(2.2, 0.95, 1.8);
						cab.position.set(0.35, 1.95, 0.85);
						group.add(cab);

						const rollBar = new THREE.Mesh(blockGeo, accent);
						rollBar.scale.set(0.22, 1.5, 2.6);
						rollBar.position.set(-1.2, 1.95, -0.15);
						group.add(rollBar);

						const wheelOffsets: Array<[number, number]> = [
							[-2.1, 1.25],
							[2.1, 1.25],
							[-2.1, -1.25],
							[2.1, -1.25]
						];
						for (const [wx, wz] of wheelOffsets) {
							const wheel = new THREE.Mesh(blockGeo, machineryDetailMat);
							wheel.scale.set(0.9, 0.9, 0.55);
							wheel.position.set(wx, 0.6, wz);
							group.add(wheel);
							parts.wheels.push(wheel);
							const hub = new THREE.Mesh(blockGeo, accent);
							hub.scale.set(0.25, 0.25, 0.12);
							wheel.add(hub);
						}

						const antenna = new THREE.Mesh(blockGeo, accent);
						antenna.scale.set(0.1, 1.3, 0.1);
						antenna.position.set(2.1, 2.25, -1.1);
						group.add(antenna);
						parts.rotors.push(antenna);
					} else if (def.id === 'gravBike') {
						parts.seatHeight = 1.95;
						parts.interactRadius = 4.4;

						const spine = new THREE.Mesh(blockGeo, machineryHullMat);
						spine.scale.set(6.2, 0.55, 1.35);
						spine.position.set(0, 1.1, 0);
						group.add(spine);

						const seat = new THREE.Mesh(blockGeo, machineryDetailMat);
						seat.scale.set(1.8, 0.55, 1.2);
						seat.position.set(0.8, 1.45, 0);
						group.add(seat);

						const nose = new THREE.Mesh(blockGeo, accent);
						nose.scale.set(1.6, 0.35, 1.0);
						nose.position.set(0, 1.25, 2.7);
						group.add(nose);

						const gyroFront = new THREE.Group();
						gyroFront.position.set(0, 1.0, 2.1);
						group.add(gyroFront);
						parts.gyros.push(gyroFront);

						const gyroBack = new THREE.Group();
						gyroBack.position.set(0, 1.0, -1.8);
						group.add(gyroBack);
						parts.gyros.push(gyroBack);

						for (const gyro of [gyroFront, gyroBack]) {
							for (let i = 0; i < 10; i += 1) {
								const angle = (i / 10) * Math.PI * 2;
								const cube = new THREE.Mesh(blockGeo, accent);
								cube.scale.set(0.22, 0.22, 0.22);
								cube.position.set(Math.cos(angle) * 1.1, 0, Math.sin(angle) * 1.1);
								gyro.add(cube);
							}
						}

						addThruster(group, new THREE.Vector3(-0.8, 1.05, -2.7), 0.42);
						addThruster(group, new THREE.Vector3(0.8, 1.05, -2.7), 0.42);
					} else {
						// Exo-suit “equipment”: big enough to board; has jetpack parts.
						parts.seatHeight = 2.35;
						parts.interactRadius = 3.8;

						const torso = new THREE.Mesh(blockGeo, machineryHullMat);
						torso.scale.set(1.8, 2.15, 1.2);
						torso.position.set(0, 2.0, 0);
						group.add(torso);

						const visor = new THREE.Mesh(blockGeo, machineryGlassMat);
						visor.scale.set(1.4, 1.0, 1.0);
						visor.position.set(0, 3.0, 0.55);
						group.add(visor);

						const pelvis = new THREE.Mesh(blockGeo, machineryDetailMat);
						pelvis.scale.set(1.5, 0.65, 1.1);
						pelvis.position.set(0, 1.05, 0);
						group.add(pelvis);

						for (const s of [-1, 1] as const) {
							const leg = new THREE.Mesh(blockGeo, machineryDetailMat);
							leg.scale.set(0.55, 1.3, 0.55);
							leg.position.set(s * 0.55, 0.45, 0);
							group.add(leg);
							parts.arms.push(leg);

							const foot = new THREE.Mesh(blockGeo, accent);
							foot.scale.set(0.7, 0.22, 1.0);
							foot.position.set(0, -0.75, 0.25);
							leg.add(foot);
						}

						const armPivotL = new THREE.Group();
						armPivotL.position.set(-1.25, 2.45, 0);
						group.add(armPivotL);
						const armPivotR = armPivotL.clone();
						armPivotR.position.x = 1.25;
						group.add(armPivotR);
						parts.arms.push(armPivotL, armPivotR);

						for (const pivot of [armPivotL, armPivotR]) {
							const upper = new THREE.Mesh(blockGeo, machineryDetailMat);
							upper.scale.set(0.45, 1.05, 0.45);
							upper.position.set(0, -0.55, 0);
							pivot.add(upper);
							const fore = new THREE.Mesh(blockGeo, machineryHullMat);
							fore.scale.set(0.42, 0.9, 0.42);
							fore.position.set(0, -1.25, 0);
							pivot.add(fore);
							const hand = new THREE.Mesh(blockGeo, accent);
							hand.scale.set(0.5, 0.25, 0.7);
							hand.position.set(0, -1.8, 0.2);
							pivot.add(hand);
						}

						const jetpack = new THREE.Mesh(blockGeo, machineryHullMat);
						jetpack.scale.set(1.45, 1.5, 0.55);
						jetpack.position.set(0, 2.25, -0.95);
						group.add(jetpack);

						addThruster(group, new THREE.Vector3(-0.55, 1.95, -1.25), 0.42);
						addThruster(group, new THREE.Vector3(0.55, 1.95, -1.25), 0.42);
					}

					return { group, parts };
				};

				const pickMachineryTarget = (machine: MachineryInstance, seed: number) => {
					const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
					for (let attempt = 0; attempt < 12; attempt += 1) {
						const a = hash2D(seed + attempt * 37, seed - attempt * 19, currentWorld.seed + 12100) * Math.PI * 2;
						const r = 6 + hash2D(seed + attempt * 11, seed + attempt * 23, currentWorld.seed + 12101) * 26;
						const tx = Math.round(machine.home.x + Math.cos(a) * r);
						const tz = Math.round(machine.home.y + Math.sin(a) * r);
						const info = getTerrainInfo(tx, tz, currentWorld);
						if (info.lavaStrength > 0.22) continue;
						if (currentWorld.fluids.waterLevel > 0 && info.height < currentWorld.fluids.waterLevel) continue;

						let minH = Number.POSITIVE_INFINITY;
						let maxH = Number.NEGATIVE_INFINITY;
						for (let ox = -1; ox <= 1; ox += 1) {
							for (let oz = -1; oz <= 1; oz += 1) {
								const h = computeHeight(tx + ox, tz + oz, currentWorld);
								minH = Math.min(minH, h);
								maxH = Math.max(maxH, h);
							}
						}
						if (maxH - minH > 3) continue;
						machine.target.set(tx, tz);
						machine.targetTimer = 2 + hash2D(tx, tz, currentWorld.seed + 12120) * 4.8;
						machine.group.position.y = Math.max(machine.group.position.y, Math.max(info.height, fluidSurface) + machine.hover);
						return;
					}
					machine.target.copy(machine.home);
					machine.targetTimer = 2;
				};

				const findSpawnSpotNearPlayer = (minR: number, maxR: number) => {
					const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
					const px = player.position.x;
					const pz = player.position.z;
					for (let attempt = 0; attempt < 32; attempt += 1) {
						const a = Math.random() * Math.PI * 2;
						const r = THREE.MathUtils.lerp(minR, maxR, Math.random());
						const x = Math.round(px + Math.cos(a) * r);
						const z = Math.round(pz + Math.sin(a) * r);
						const info = getTerrainInfo(x, z, currentWorld);
						if (info.lavaStrength > 0.22) continue;
						if (currentWorld.fluids.waterLevel > 0 && info.height < currentWorld.fluids.waterLevel) continue;
						if (info.biome.id === 'ocean') continue;

						let minH = Number.POSITIVE_INFINITY;
						let maxH = Number.NEGATIVE_INFINITY;
						for (let ox = -1; ox <= 1; ox += 1) {
							for (let oz = -1; oz <= 1; oz += 1) {
								const h = computeHeight(x + ox, z + oz, currentWorld);
								minH = Math.min(minH, h);
								maxH = Math.max(maxH, h);
							}
						}
						if (maxH - minH > 3) continue;
						const surface = Math.max(info.height, fluidSurface);
						return { x, z, y: surface + 0.25 };
					}
					return null;
				};

				const spawnMachinery = () => {
					const def = weightedPick(MACHINERY_CATALOG, (entry) => raritySpawnWeight(entry.rarity));
					if (!def) return false;
					const spot = findSpawnSpotNearPlayer(16, 44);
					if (!spot) return false;

					const model = createMachineryModel(def);
					const group = model.group;
					const hover = getMachineryHover(def);
					group.position.set(spot.x, spot.y + hover, spot.z);

					const label = new THREE.Sprite(getMachineryLabelMat(def));
					label.position.set(0, model.parts.seatHeight + 2.35, 0);
					label.scale.set(def.category === 'ship' ? 8.4 : 7.2, 1.35, 1);
					group.add(label);

					const rental = getRentalOffer(def);
					const machine: MachineryInstance = {
						def,
						group,
						label,
						home: new THREE.Vector2(spot.x, spot.z),
						target: new THREE.Vector2(spot.x, spot.z),
						speed: getMachinerySpeed(def),
						phase: Math.random() * Math.PI * 2,
						targetTimer: 1.2 + Math.random() * 3.2,
						life: 45 + Math.random() * 55,
						hover,
						parts: model.parts,
						isOwned: false,
						boarded: false,
						rentalRemaining: 0,
						rentalCost: rental.cost,
						rentalDuration: rental.duration,
						desiredHover: hover,
						lastPos: group.position.clone(),
						wheelRoll: 0
					};

					machineries.push(machine);
					scene.add(group);
					pickMachineryTarget(machine, Math.floor(Math.random() * 10_000));
					return true;
				};

				const removeMachineryInstance = (machine: MachineryInstance) => {
					scene.remove(machine.group);
					const idx = machineries.indexOf(machine);
					if (idx >= 0) {
						machineries.splice(idx, 1);
					}
					if (activeMachine === machine) {
						activeMachine = null;
					}
					if (deployedOwnedMachine === machine) {
						deployedOwnedMachine = null;
					}
				};

				const spawnMachineryInstance = (def: MachineryDef, spot: { x: number; y: number; z: number }, isOwned: boolean) => {
					const model = createMachineryModel(def);
					const group = model.group;
					const hover = getMachineryHover(def);
					group.position.set(spot.x, spot.y + hover, spot.z);

					const label = new THREE.Sprite(getMachineryLabelMat(def));
					label.position.set(0, model.parts.seatHeight + 2.35, 0);
					label.scale.set(def.category === 'ship' ? 8.4 : 7.2, 1.35, 1);
					group.add(label);

					const rental = getRentalOffer(def);
					const machine: MachineryInstance = {
						def,
						group,
						label,
						home: new THREE.Vector2(spot.x, spot.z),
						target: new THREE.Vector2(spot.x, spot.z),
						speed: isOwned ? 0 : getMachinerySpeed(def),
						phase: Math.random() * Math.PI * 2,
						targetTimer: isOwned ? 999 : 1.2 + Math.random() * 3.2,
						life: isOwned ? Number.POSITIVE_INFINITY : 45 + Math.random() * 55,
						hover,
						parts: model.parts,
						isOwned,
						boarded: false,
						rentalRemaining: isOwned ? Number.POSITIVE_INFINITY : 0,
						rentalCost: rental.cost,
						rentalDuration: rental.duration,
						desiredHover: hover,
						lastPos: group.position.clone(),
						wheelRoll: 0
					};

					machineries.push(machine);
					scene.add(group);
					if (!isOwned) {
						pickMachineryTarget(machine, Math.floor(Math.random() * 10_000));
					}
					return machine;
				};

					const deployOwnedMachinery = (id: MachineryId) => {
						if ((ownedMachinery[id] ?? 0) <= 0) {
							showToast('You do not own that machinery yet.', 'warn');
							return null;
						}
					const def = MACHINERY_BY_ID.get(id);
					if (!def) {
						showToast('Unknown machinery id.', 'error');
						return null;
					}
					if (deployedOwnedMachine) {
						removeMachineryInstance(deployedOwnedMachine);
					}
					const spot =
						findSpawnSpotNearPlayer(10, 18) ??
						(() => {
							const surface = getHeightAt(Math.round(player.position.x + 6), Math.round(player.position.z + 6));
							return { x: player.position.x + 6, z: player.position.z + 6, y: surface + 0.25 };
						})();
						const machine = spawnMachineryInstance(def, { x: spot.x, y: spot.y, z: spot.z }, true);
						deployedOwnedMachine = machine;
						return machine;
					};

					deployMachinery = (id) => {
						const machine = deployOwnedMachinery(id);
						if (machine) {
							showToast(`Deployed ${machine.def.name}.`, 'info');
						}
					};

					const claimMachineryAsOwned = (machine: MachineryInstance) => {
						if (machine.isOwned) {
							return;
						}
						if (deployedOwnedMachine && deployedOwnedMachine !== machine) {
							removeMachineryInstance(deployedOwnedMachine);
						}
						machine.isOwned = true;
						machine.life = Number.POSITIVE_INFINITY;
						machine.speed = 0;
						machine.targetTimer = 999;
						machine.rentalRemaining = Number.POSITIVE_INFINITY;
						machine.home.set(machine.group.position.x, machine.group.position.z);
						machine.target.copy(machine.home);
						machine.label.visible = !machine.boarded;
						deployedOwnedMachine = machine;
					};

					const purchaseMachinery = (machine: MachineryInstance) => {
						if (machine.isOwned) {
							showToast('This machinery is already owned.', 'warn');
							return false;
						}
						const cost = getMachineryCost(machine.def.id);
						if (!spendCrypto(cost)) {
							showToast(`Not enough SC for ${machine.def.name} (${cost} SC).`, 'warn');
							return false;
						}
					ownedMachinery = {
							...ownedMachinery,
							[machine.def.id]: (ownedMachinery[machine.def.id] ?? 0) + 1
						};
						saveGameState();
						claimMachineryAsOwned(machine);
						showToast(`Purchased ${machine.def.name} for ${cost} SC.`, 'info');
						machinerySpawnTimer = Math.min(machinerySpawnTimer, 2.5);
						return true;
					};

					const beginRental = (machine: MachineryInstance) => {
						if (machine.isOwned) {
							return true;
						}
						if (machine.rentalRemaining > 0) {
							return true;
						}
						if (!spendCrypto(machine.rentalCost)) {
							showToast(`Not enough SC to rent ${machine.def.name} (${machine.rentalCost} SC).`, 'warn');
							return false;
						}
						machine.rentalRemaining = machine.rentalDuration;
						machine.speed = 0;
						machine.targetTimer = 999;
						showToast(`Rented ${machine.def.name} for ${machine.rentalCost} SC (${machine.rentalDuration}s).`, 'info');
						return true;
					};

					const extendRental = (machine: MachineryInstance) => {
						if (machine.isOwned || !Number.isFinite(machine.rentalRemaining) || machine.rentalRemaining <= 0) {
							return true;
						}
						if (!spendCrypto(machine.rentalCost)) {
							showToast(`Not enough SC to extend rental (${machine.rentalCost} SC).`, 'warn');
							return false;
						}
						machine.rentalRemaining += machine.rentalDuration;
						showToast(`Rental extended (+${machine.rentalDuration}s).`, 'info');
						return true;
					};

					type VehicleParticle = {
						mesh: THREE.Mesh;
						velocity: THREE.Vector3;
						life: number;
				};

				const vehicleParticles: VehicleParticle[] = [];
				const vehicleParticleMats = new Map<Rarity, THREE.MeshStandardMaterial>();

				const getVehicleParticleMat = (rarity: Rarity) => {
					const cached = vehicleParticleMats.get(rarity);
					if (cached) return cached;
					const color = new THREE.Color(RARITY_COLOR[rarity]);
					const mat = new THREE.MeshStandardMaterial({
						color,
						emissive: color,
						emissiveIntensity: 1.15,
						roughness: 0.35,
						metalness: 0.2
					});
					vehicleParticleMats.set(rarity, mat);
					return mat;
				};

				const spawnVehicleBurst = (position: THREE.Vector3, rarity: Rarity) => {
					const mat = getVehicleParticleMat(rarity);
					const count = 70;
					for (let i = 0; i < count; i += 1) {
						const mesh = new THREE.Mesh(portalParticleGeo, mat);
						mesh.position.copy(position);
						mesh.position.x += THREE.MathUtils.randFloatSpread(2);
						mesh.position.y += THREE.MathUtils.randFloat(0.4, 2.6);
						mesh.position.z += THREE.MathUtils.randFloatSpread(2);
						const velocity = new THREE.Vector3(
							THREE.MathUtils.randFloatSpread(2),
							THREE.MathUtils.randFloat(1.2, 3.2),
							THREE.MathUtils.randFloatSpread(2)
						)
							.normalize()
							.multiplyScalar(THREE.MathUtils.randFloat(2.6, 6));
						vehicleParticles.push({ mesh, velocity, life: THREE.MathUtils.randFloat(0.55, 1.25) });
						scene.add(mesh);
					}
				};

				const spawnThrusterTrail = (machine: MachineryInstance, intensity: number) => {
					if (machine.parts.thrusterPoints.length === 0) return;
					const mat = getVehicleParticleMat(machine.def.rarity);
					const backDir = new THREE.Vector3(0, 0, -1).applyQuaternion(machine.group.quaternion);
					const count = Math.max(1, Math.floor(machine.parts.thrusterPoints.length * intensity));
					for (let i = 0; i < count; i += 1) {
						const point = machine.parts.thrusterPoints[i % machine.parts.thrusterPoints.length];
						const worldPoint = machine.group.localToWorld(new THREE.Vector3(point.x, point.y, point.z));
						const mesh = new THREE.Mesh(portalParticleGeo, mat);
						mesh.position.copy(worldPoint);
						mesh.position.x += THREE.MathUtils.randFloatSpread(0.22);
						mesh.position.y += THREE.MathUtils.randFloatSpread(0.18);
						mesh.position.z += THREE.MathUtils.randFloatSpread(0.22);
						const velocity = backDir
							.clone()
							.multiplyScalar(THREE.MathUtils.randFloat(3.4, 6.8) * intensity)
							.add(
								new THREE.Vector3(
									THREE.MathUtils.randFloatSpread(0.9),
									THREE.MathUtils.randFloatSpread(0.6),
									THREE.MathUtils.randFloatSpread(0.9)
								)
							);
						vehicleParticles.push({ mesh, velocity, life: THREE.MathUtils.randFloat(0.25, 0.55) });
						scene.add(mesh);
					}
				};

					const enterVehicle = (machine: MachineryInstance) => {
						if (activeMachine) {
							return;
						}
						activeMachine = machine;
						machine.boarded = true;
						machine.label.visible = false;
						machine.desiredHover = machine.hover;
						machine.wheelRoll = 0;
						machine.lastPos.copy(machine.group.position);
						player.rotation.y = machine.group.rotation.y + Math.PI;
						playerBody.setNextKinematicTranslation({
							x: machine.group.position.x,
							y: machine.group.position.y,
							z: machine.group.position.z
						});
						player.position.copy(machine.group.position);
						verticalVelocity = 0;
						grounded = false;
						input.jump = false;
						input.down = false;
						spawnVehicleBurst(machine.group.position, machine.def.rarity);
						showToast(`Boarded ${machine.def.name}.`, 'info');
					};

				const exitVehicle = (reason: 'manual' | 'expired') => {
					const machine = activeMachine;
					if (!machine) return;
					activeMachine = null;
					machine.boarded = false;
					machine.label.visible = true;

					spawnVehicleBurst(machine.group.position, machine.def.rarity);

					// Place the player next to the vehicle, snapped to the surface.
					const right = new THREE.Vector3(1, 0, 0).applyAxisAngle(new THREE.Vector3(0, 1, 0), player.rotation.y);
					const offset = right.multiplyScalar(machine.parts.interactRadius + 1.3);
					const tx = Math.round(machine.group.position.x + offset.x);
					const tz = Math.round(machine.group.position.z + offset.z);
					const surface = getHeightAt(tx, tz);
					const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
					const ty = Math.max(surface, fluidSurface) + 0.25;
					playerBody.setNextKinematicTranslation({ x: tx, y: ty, z: tz });
					player.position.set(tx, ty, tz);
					verticalVelocity = 0;
					grounded = false;
					syncChunks(tx, tz, true);

					if (reason === 'expired') {
						showToast('Rental expired. You were kicked out.', 'warn');
					} else {
						showToast('Disembarked.', 'info');
					}
				};

				type CardDef = (typeof CARD_CATALOG)[number];

				type CardInstance = {
					def: CardDef;
					sprite: THREE.Sprite;
					position: THREE.Vector3;
					phase: number;
					life: number;
				};

				const cards: CardInstance[] = [];

				const clearCards = () => {
					for (const card of cards) {
						scene.remove(card.sprite);
					}
					cards.length = 0;
				};

				const CARD_REWARD_BY_RARITY: Record<Rarity, number> = {
					common: 14,
					uncommon: 28,
					rare: 60,
					epic: 120,
					legendary: 250
				};

				const spawnCard = () => {
					const def = weightedPick(CARD_CATALOG, (entry) => raritySpawnWeight(entry.rarity));
					if (!def) return false;
					const spot = findSpawnSpotNearPlayer(10, 52);
					if (!spot) return false;
					const sprite = new THREE.Sprite(getCardMat(def));
					sprite.position.set(spot.x, spot.y + 1.35, spot.z);
					sprite.scale.set(1.2, 0.82, 1);
					scene.add(sprite);
					cards.push({
						def,
						sprite,
						position: sprite.position.clone(),
						phase: Math.random() * Math.PI * 2,
						life: 55 + Math.random() * 70
					});
					return true;
				};

				const collectCard = (card: CardInstance) => {
					const reward = CARD_REWARD_BY_RARITY[card.def.rarity] ?? 10;
					collectedCards = {
						...collectedCards,
						[card.def.id]: (collectedCards[card.def.id] ?? 0) + 1
					};
					addCrypto(reward);
					showToast(`Found ${card.def.name}. +${reward} SC`, 'info');
					scene.remove(card.sprite);
					const idx = cards.indexOf(card);
					if (idx >= 0) {
						cards.splice(idx, 1);
					}
					cardSpawnTimer = Math.min(cardSpawnTimer, 2.5);
				};

					const resetWorldFindables = () => {
						clearMachinery();
						clearCards();
						interactRequested = false;
						altInteractRequested = false;
						disembarkRequested = false;
						canDisembark = false;
						interactionHint = null;
						interactionActionLabel = null;
						interactionAltActionLabel = null;
						machinerySpawnTimer = THREE.MathUtils.randFloat(6, 10);
						cardSpawnTimer = THREE.MathUtils.randFloat(3, 6);

					// Initial pop so there's always something to find after switching worlds.
					for (let i = 0; i < 2; i += 1) spawnMachinery();
					for (let i = 0; i < 4; i += 1) spawnCard();
				};

					const machineryMaxActive = 3;
					let machinerySpawnTimer = 7;
					const cardsMaxActive = 9;
					let cardSpawnTimer = 4;
					const PASSIVE_INCOME_INTERVAL = 20;
					const PASSIVE_INCOME_AMOUNT = 18;
					let passiveIncomeTimer = PASSIVE_INCOME_INTERVAL;

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
			let sfxCtx: AudioContext | null = null;
			let breakNoise: AudioBuffer | null = null;

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

			const ensureSfxContext = () => {
				if (!audioUnlocked) {
					return null;
				}
				const Ctor: typeof AudioContext | undefined =
					window.AudioContext ?? ((window as unknown as { webkitAudioContext?: typeof AudioContext }).webkitAudioContext);
				if (!Ctor) {
					return null;
				}
				if (!sfxCtx) {
					sfxCtx = new Ctor();
					const length = Math.floor(sfxCtx.sampleRate * 0.14);
					breakNoise = sfxCtx.createBuffer(1, length, sfxCtx.sampleRate);
					const data = breakNoise.getChannelData(0);
					for (let i = 0; i < length; i += 1) {
						const t = i / Math.max(1, length - 1);
						const env = Math.pow(1 - t, 2.25);
						data[i] = (Math.random() * 2 - 1) * env;
					}
				}
				if (sfxCtx.state === 'suspended') {
					sfxCtx.resume().catch(() => {});
				}
				return sfxCtx;
			};

			const playBlockBreakSound = (type: BlockType) => {
				const ctx = ensureSfxContext();
				if (!ctx || !breakNoise) {
					return;
				}
				const now = ctx.currentTime;
				const gain = ctx.createGain();
				const baseVolume =
					type === 'glass' ? 0.085
						: type === 'stone' || type === 'cobble' || type === 'obsidian' ? 0.12
							: type === 'sand' || type === 'gravel' ? 0.095
								: 0.105;
				gain.gain.setValueAtTime(0.0001, now);
				gain.gain.exponentialRampToValueAtTime(baseVolume, now + 0.008);
				gain.gain.exponentialRampToValueAtTime(0.0001, now + 0.14);

				const filter = ctx.createBiquadFilter();
				filter.type = 'bandpass';
				const centerFreq =
					type === 'glass' ? 1450
						: type === 'wood' || type === 'log' || type === 'redwood' || type === 'door' ? 380
							: type === 'sand' || type === 'gravel' || type === 'clay' ? 520
								: 720;
				filter.frequency.setValueAtTime(centerFreq, now);
				filter.Q.setValueAtTime(0.9, now);

				const src = ctx.createBufferSource();
				src.buffer = breakNoise;
				const rate = type === 'sand' || type === 'gravel' ? 0.85 : 1;
				src.playbackRate.setValueAtTime(rate, now);
				src.connect(filter);
				filter.connect(gain);
				gain.connect(ctx.destination);
				src.start(now);
				src.stop(now + 0.16);

				const tickOsc = ctx.createOscillator();
				tickOsc.type = 'triangle';
				const tickFreq =
					type === 'glass' ? 1220
						: type === 'stone' || type === 'cobble' || type === 'obsidian' ? 230
							: type === 'wood' || type === 'log' || type === 'redwood' || type === 'door' ? 360
								: 520;
				tickOsc.frequency.setValueAtTime(tickFreq, now);
				const tickGain = ctx.createGain();
				tickGain.gain.setValueAtTime(0.0001, now);
				tickGain.gain.exponentialRampToValueAtTime(baseVolume * 0.65, now + 0.006);
				tickGain.gain.exponentialRampToValueAtTime(0.0001, now + 0.06);
				tickOsc.connect(tickGain);
				tickGain.connect(ctx.destination);
				tickOsc.start(now);
				tickOsc.stop(now + 0.07);
			};

			const playBlockPlaceSound = (type: BlockType) => {
				const ctx = ensureSfxContext();
				if (!ctx || !breakNoise) {
					return;
				}
				const now = ctx.currentTime;
				const gain = ctx.createGain();
				const baseVolume =
					type === 'glass' ? 0.06
						: type === 'stone' || type === 'cobble' || type === 'obsidian' ? 0.08
							: type === 'sand' || type === 'gravel' || type === 'clay' ? 0.07
								: 0.075;
				gain.gain.setValueAtTime(0.0001, now);
				gain.gain.exponentialRampToValueAtTime(baseVolume, now + 0.004);
				gain.gain.exponentialRampToValueAtTime(0.0001, now + 0.065);

				const filter = ctx.createBiquadFilter();
				filter.type = 'lowpass';
				const cutoff =
					type === 'glass' ? 2400
						: type === 'wood' || type === 'log' || type === 'redwood' || type === 'door' ? 900
							: type === 'sand' || type === 'gravel' || type === 'clay' ? 700
								: 1250;
				filter.frequency.setValueAtTime(cutoff, now);
				filter.Q.setValueAtTime(0.7, now);

				const src = ctx.createBufferSource();
				src.buffer = breakNoise;
				const rate =
					type === 'sand' || type === 'gravel' ? 1.05
						: type === 'glass' ? 1.2
							: 1.1;
				src.playbackRate.setValueAtTime(rate, now);
				src.connect(filter);
				filter.connect(gain);
				gain.connect(ctx.destination);
				src.start(now);
				src.stop(now + 0.08);
			};

			const input = {
				forward: false,
				back: false,
				left: false,
				right: false,
				jump: false,
				down: false
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
						case 'ShiftLeft':
						case 'ShiftRight':
						case 'ControlLeft':
						case 'ControlRight':
							input.down = true;
							break;
						case 'Digit1':
						case 'Digit2':
						case 'Digit3':
						case 'Digit4':
						case 'Digit5':
						case 'Digit6':
						case 'Digit7':
						case 'Digit8':
					case 'Digit9':
							selectedHotbar = Math.min(HOTBAR_SLOTS - 1, Math.max(0, Number(event.code.replace('Digit', '')) - 1));
							break;
						case 'KeyE':
							if (!event.repeat) {
								requestInteract();
							}
							break;
						case 'KeyB':
							if (!event.repeat) {
								requestAltInteract();
							}
							break;
						case 'KeyF':
							if (!event.repeat) {
								requestDisembark();
							}
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
					case 'ShiftLeft':
					case 'ShiftRight':
					case 'ControlLeft':
					case 'ControlRight':
						input.down = false;
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
					lastMouseX: 0,
					lastMouseY: 0
				};

				let miningDown = false;
				let miningPointerId: number | null = null;
				let pointerLocked = false;
				let placeRequested = false;

				const handlePointerLockChange = () => {
					pointerLocked = document.pointerLockElement === renderer.domElement;
					if (!pointerLocked) {
						miningDown = false;
						miningPointerId = null;
						placeRequested = false;
					}
				};
				document.addEventListener('pointerlockchange', handlePointerLockChange);

				const handleContextMenu = (event: MouseEvent) => {
					event.preventDefault();
				};

				const handleMouseDown = (event: PointerEvent) => {
					if (event.pointerType !== 'mouse') {
						return;
					}
					ensureAudio();
					if (!pointerLocked && typeof renderer.domElement.requestPointerLock === 'function') {
						renderer.domElement.requestPointerLock();
					}
					if (event.button === 0) {
						miningDown = true;
					} else if (event.button === 2 && pointerLocked) {
						placeRequested = true;
					}
					event.preventDefault();
				};

				const handleMouseMove = (event: PointerEvent) => {
					if (event.pointerType !== 'mouse' || !pointerLocked) {
						return;
					}
					// three.js +Y yaw turns left; invert X so moving mouse right turns right.
					pointerState.lookYaw -= event.movementX * 0.0024;
					pointerState.lookPitch -= event.movementY * 0.0024;
				};

				const handleMouseUp = (event: PointerEvent) => {
					if (event.pointerType !== 'mouse') {
						return;
					}
					if (event.button === 0) {
						miningDown = false;
					}
				};

			renderer.domElement.addEventListener('pointerdown', handleMouseDown);
			renderer.domElement.addEventListener('contextmenu', handleContextMenu);
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

				const handleDownDown = (event: PointerEvent) => {
					if (event.pointerType === 'mouse') {
						return;
					}
					ensureAudio();
					input.down = true;
					event.preventDefault();
					event.stopPropagation();
				};

				const handleDownUp = (event: PointerEvent) => {
					if (event.pointerType === 'mouse') {
						return;
					}
					input.down = false;
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
					miningPointerId = event.pointerId;
					miningDown = true;
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
				pointerState.lookYaw -= dx * 0.004;
				pointerState.lookPitch -= dy * 0.004;
				event.preventDefault();
			};

				const handleLookUp = (event: PointerEvent) => {
					if (event.pointerId != lookState.pointerId) {
						return;
					}
					lookState.pointerId = null;
					if (event.pointerId === miningPointerId) {
						miningDown = false;
						miningPointerId = null;
					}
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
				downEl?.addEventListener('pointerdown', handleDownDown, { passive: false });
				downEl?.addEventListener('pointerup', handleDownUp);
				downEl?.addEventListener('pointercancel', handleDownUp);
				renderer.domElement.addEventListener('pointerdown', handleLookDown, { passive: false });
				renderer.domElement.addEventListener('pointermove', handleLookMove, { passive: false });
				renderer.domElement.addEventListener('pointerup', handleLookUp);
				renderer.domElement.addEventListener('pointercancel', handleLookUp);

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
						const coreMat = portal.core.material as THREE.MeshStandardMaterial;
						coreMat.map?.dispose();
						coreMat.dispose();
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
						const y = Math.max(terrainY, fluidSurface);

						const group = new THREE.Group();
						const portalAxis = Math.abs(x) > Math.abs(z) ? 'x' : 'z';
						const portalRotationY = portalAxis === 'x' ? Math.PI / 2 : 0;
						group.rotation.y = portalRotationY;

						// Minecraft-ish 4x5 obsidian frame (outer), with a 2x3 portal surface inside.
						const frameW = 4;
						const frameH = 5;
						const frameCenters: Array<[number, number, number]> = [];
						for (let by = 0; by < frameH; by += 1) {
							for (let bx = 0; bx < frameW; bx += 1) {
								const edge = bx === 0 || bx === frameW - 1 || by === 0 || by === frameH - 1;
								if (!edge) continue;
								frameCenters.push([bx - (frameW - 1) / 2, by + 0.5, 0]);
							}
						}
						const frame = new THREE.InstancedMesh(portalFrameGeo, portalFrameMat, frameCenters.length);
						frame.castShadow = true;
						frame.receiveShadow = true;
						const portalMatrix = new THREE.Matrix4();
						for (let i = 0; i < frameCenters.length; i += 1) {
							const [fx, fy, fz] = frameCenters[i] ?? [0, 0, 0];
							portalMatrix.makeTranslation(fx, fy, fz);
							frame.setMatrixAt(i, portalMatrix);
						}
						frame.instanceMatrix.needsUpdate = true;
						group.add(frame);

						const coreColor = new THREE.Color(target.portalColor);
						const portalCoreTex = createCanvasTexture((ctx, size) => {
							const r = Math.floor(coreColor.r * 255);
							const g = Math.floor(coreColor.g * 255);
							const b = Math.floor(coreColor.b * 255);
							ctx.fillStyle = `rgb(${Math.floor(r * 0.2)}, ${Math.floor(g * 0.2)}, ${Math.floor(b * 0.2)})`;
							ctx.fillRect(0, 0, size, size);
							for (let i = 0; i < 240; i += 1) {
								const px = Math.floor(Math.random() * size);
								const py = Math.floor(Math.random() * size);
								const a = 0.06 + Math.random() * 0.18;
								ctx.fillStyle = `rgba(${r}, ${g}, ${b}, ${a})`;
								ctx.fillRect(px, py, 1, 1);
							}
							ctx.strokeStyle = `rgba(${Math.min(255, r + 40)}, ${Math.min(255, g + 40)}, ${Math.min(255, b + 40)}, 0.25)`;
							ctx.lineWidth = 2;
							for (let i = 0; i < 6; i += 1) {
								ctx.beginPath();
								ctx.moveTo(Math.random() * size, Math.random() * size);
								ctx.lineTo(Math.random() * size, Math.random() * size);
								ctx.stroke();
							}
						});
						portalCoreTex.repeat.set(1, 1.6);
						const coreMat = new THREE.MeshStandardMaterial({
							map: portalCoreTex,
							color: coreColor,
							emissive: coreColor,
							emissiveIntensity: 0.95,
							transparent: true,
							opacity: 0.75,
							roughness: 0.2,
							side: THREE.DoubleSide,
							depthWrite: false
						});
						const core = new THREE.Mesh(portalCoreGeo, coreMat);
						core.position.set(0, 2.5, 0.51);
						group.add(core);

						const labelTexture = createLabelTexture(`To ${target.name}`, target.portalColor);
						const labelMat = new THREE.SpriteMaterial({
						map: labelTexture,
						transparent: true,
						depthTest: false
					});
						const label = new THREE.Sprite(labelMat);
						label.position.set(0, 5.65, 0);
						label.scale.set(4.4, 1.05, 1);
						group.add(label);

						group.position.set(x + (portalRotationY === 0 ? 0.5 : 0), y, z + (portalRotationY === 0 ? 0 : 0.5));
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
					spawnCrittersForWorld();
					if (resetPlayer) {
						const spawn = findSafeSpawn(worldDef);
						playerBody.setNextKinematicTranslation({ x: spawn.x, y: spawn.y, z: spawn.z });
						player.position.set(spawn.x, spawn.y, spawn.z);
					verticalVelocity = 0;
					grounded = false;
				}
				resetWorldFindables();
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

			const blockBreakTints: Partial<Record<BlockType, number>> = {
				grass: 0x74c85a,
				dirt: 0x80512c,
				stone: 0x9ea2a3,
				cobble: 0x8b8b8b,
				wood: 0xc39a5d,
				redwood: 0x81432a,
				sandstone: 0xd8c48a,
				obsidian: 0x2a1a3d,
				marsSand: 0xc88c76,
				marsRock: 0x8f5f53,
				moonDust: 0xd9dfe9,
				sand: 0xd9c58c,
				gravel: 0x8a8e8f,
				log: 0x9b7048,
				leaves: 0x4f8f4b,
				glass: 0x9ad6e8,
				brick: 0x9b4a3a,
				door: 0x6e4a2e,
				coalOre: 0x4c4c4c,
				ironOre: 0x9c7657,
				mossyCobble: 0x6f7f6a,
				clay: 0xa7b4b7,
				snow: 0xf2f5ff,
				ice: 0xb9e3ff,
				netherrack: 0x6c2f2e
			};

			const breakParticleMats = new Map<BlockType, THREE.MeshStandardMaterial>();

			const getBreakParticleMat = (type: BlockType) => {
				const cached = breakParticleMats.get(type);
				if (cached) {
					return cached;
				}
				const tint = blockBreakTints[type] ?? 0xffc06b;
				const mat = new THREE.MeshStandardMaterial({
					color: tint,
					roughness: 0.9,
					metalness: 0
				});
				breakParticleMats.set(type, mat);
				return mat;
			};

			const spawnBlockBreakParticles = (x: number, y: number, z: number, type: BlockType) => {
				const centerY = y + 0.5;
				const count =
					type === 'leaves' || type === 'snow' ? 10
						: type === 'glass' ? 14
							: 18;
				const mat = getBreakParticleMat(type);
				for (let i = 0; i < count; i += 1) {
					const mesh = new THREE.Mesh(particleGeo, mat);
					mesh.position.set(
						x + THREE.MathUtils.randFloatSpread(0.45),
						centerY + THREE.MathUtils.randFloat(0.05, 0.55),
						z + THREE.MathUtils.randFloatSpread(0.45)
					);
					const velocity = new THREE.Vector3(
						THREE.MathUtils.randFloatSpread(1.8),
						THREE.MathUtils.randFloat(1.2, 3.0),
						THREE.MathUtils.randFloatSpread(1.8)
					)
						.normalize()
						.multiplyScalar(THREE.MathUtils.randFloat(1.6, 3.4));
					const scale = THREE.MathUtils.randFloat(0.8, 1.35);
					mesh.scale.setScalar(scale);
					particles.push({ mesh, velocity, life: THREE.MathUtils.randFloat(0.35, 0.7) });
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
				const headTopOffset = new THREE.Vector3(0, playerHeight, 0);
					const tempVec = new THREE.Vector3();
					const tempVec2 = new THREE.Vector3();
					const tempVec3 = new THREE.Vector3();
					const viewEuler = new THREE.Euler(0, 0, 0, 'YXZ');
					const viewDir = new THREE.Vector3();
				const viewTarget = new THREE.Vector3();
				const raycaster = new THREE.Raycaster();
				const mineRaycaster = new THREE.Raycaster();
				const mineNdc = new THREE.Vector2(0, 0);
					const mineMatrix = new THREE.Matrix4();
					const minePos = new THREE.Vector3();
					const maxMineDistance = 5.2;

					let aimedBlock: BlockTarget | null = null;
					const aimedHitPoint = new THREE.Vector3();
					let hasAimedHitPoint = false;
						let miningTargetKey: string | null = null;
					let miningProgress = 0;
					let miningRequired = 0;
					let lastBreakStage = -1;
					
					const clock = new THREE.Clock();
					const earthDayLengthSeconds = 300;
			let frame = 0;
			let spawnTimer = 0.6;
			let verticalVelocity = 0;
			let grounded = false;

			applyWorld(currentWorld, false);

				const tick = () => {
					const delta = Math.min(clock.getDelta(), 0.05);
					const time = clock.elapsedTime;
					updateAudio(delta);
					passiveIncomeTimer -= delta;
					while (passiveIncomeTimer <= 0) {
						passiveIncomeTimer += PASSIVE_INCOME_INTERVAL;
						addCrypto(PASSIVE_INCOME_AMOUNT);
					}

				let isNight = false;
				if (currentWorld.id === 'earth') {
					const dayPhase = (time / earthDayLengthSeconds) % 1;
					const sunHeight = Math.sin(dayPhase * Math.PI * 2 + Math.PI / 2);
					const nightFactor = 1 - smoothstep(-0.2, 0.2, sunHeight);
					isNight = nightFactor > 0.5;
					hemiLight.intensity = lerp(currentWorld.light.hemiIntensity, currentWorld.light.hemiIntensity * 0.28, nightFactor);
					dirLight.intensity = lerp(currentWorld.light.dirIntensity, currentWorld.light.dirIntensity * 0.12, nightFactor);
				} else {
					hemiLight.intensity = currentWorld.light.hemiIntensity;
					dirLight.intensity = currentWorld.light.dirIntensity;
				}

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
						input.down = false;
						if (interactionHint !== null) interactionHint = null;
						if (interactionActionLabel !== null) interactionActionLabel = null;
						if (interactionAltActionLabel !== null) interactionAltActionLabel = null;
						interactRequested = false;
						altInteractRequested = false;
						disembarkRequested = false;
						canDisembark = false;
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

					const tntHits = new Set<FallingBlock>();
					const pushHits = new Set<FallingBlock>();
					let movementStrength = Math.min(Math.abs(moveInput), 1);

					let disembarkedThisFrame = false;
					if (disembarkRequested) {
						disembarkRequested = false;
						if (activeMachine) {
							exitVehicle('manual');
							disembarkedThisFrame = true;
						}
					}

					if (!disembarkedThisFrame) {
						const drivingMachine = activeMachine;
						const currentPos = playerBody.translation();

						tempVec.copy(forwardBase).applyQuaternion(player.quaternion);
						const desiredMovement = tempVec2.copy(tempVec);

						if (drivingMachine) {
							const mode = getVehicleMode(drivingMachine.def);
							const driveSpeed = getVehicleDriveSpeed(drivingMachine.def);
							const verticalSpeed = getVehicleVerticalSpeed(drivingMachine.def);
							const maxAltitude = getVehicleMaxAltitudeAboveSurface(drivingMachine.def);

							verticalVelocity = 0;
							grounded = false;

							desiredMovement.multiplyScalar(driveSpeed * moveInput * delta);

							const verticalInput = (input.jump ? 1 : 0) + (input.down ? -1 : 0);
							const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
							const mx = Math.round(currentPos.x);
							const mz = Math.round(currentPos.z);
							const surface = getHeightAt(mx, mz);
							const baseSurface = Math.max(surface, fluidSurface) + 0.25;

							if (mode === 'hover') {
								drivingMachine.desiredHover = THREE.MathUtils.clamp(
									drivingMachine.desiredHover + verticalInput * verticalSpeed * delta,
									drivingMachine.hover * 0.65,
									maxAltitude
								);
								const bob = Math.sin(time * 2.1 + drivingMachine.phase) * 0.08;
								const targetY = baseSurface + drivingMachine.desiredHover + bob;
								desiredMovement.y = (targetY - currentPos.y) * Math.min(1, delta * 8);
							} else {
								desiredMovement.y = verticalInput * verticalSpeed * delta;
							}
						} else {
							if (grounded && verticalVelocity < 0) {
								verticalVelocity = 0;
							}
							if (input.jump && grounded) {
								verticalVelocity = 7.2;
								grounded = false;
								input.jump = false;
							}
							verticalVelocity += gravity * delta;

							desiredMovement.multiplyScalar(4.2 * moveInput * delta);
							desiredMovement.y = verticalVelocity * delta;
						}

						controller.computeColliderMovement(playerCollider, desiredMovement);
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
						const nextPos = {
							x: currentPos.x + actualMovement.x,
							y: currentPos.y + actualMovement.y,
							z: currentPos.z + actualMovement.z
						};

						if (drivingMachine) {
							const mode = getVehicleMode(drivingMachine.def);
							const maxAltitude = getVehicleMaxAltitudeAboveSurface(drivingMachine.def);
							const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
							const mx = Math.round(nextPos.x);
							const mz = Math.round(nextPos.z);
							const surface = getHeightAt(mx, mz);
							const baseSurface = Math.max(surface, fluidSurface) + 0.25;
							const minY = baseSurface + (mode === 'hover' ? drivingMachine.hover * 0.65 : 0.25);
							const maxY = baseSurface + maxAltitude;
							nextPos.y = THREE.MathUtils.clamp(nextPos.y, minY, maxY);
						}

						playerBody.setNextKinematicTranslation(nextPos);
						player.position.set(nextPos.x, nextPos.y, nextPos.z);
						syncChunks(player.position.x, player.position.z);

						grounded = drivingMachine ? false : controller.computedGrounded();
						if (grounded && verticalVelocity < 0) {
							verticalVelocity = 0;
						}

						if (drivingMachine) {
							drivingMachine.group.position.set(nextPos.x, nextPos.y, nextPos.z);
							drivingMachine.group.rotation.y = player.rotation.y + Math.PI;
							const turnBank = THREE.MathUtils.clamp(-turnInput * 0.32, -0.35, 0.35);
							drivingMachine.group.rotation.z = lerp(
								drivingMachine.group.rotation.z,
								turnBank,
								Math.min(1, delta * 6)
							);
							const pitchTilt = THREE.MathUtils.clamp(moveInput * 0.08, -0.1, 0.1);
							drivingMachine.group.rotation.x = lerp(
								drivingMachine.group.rotation.x,
								pitchTilt,
								Math.min(1, delta * 6)
							);
							const verticalInput = (input.jump ? 1 : 0) + (input.down ? -1 : 0);
							const throttle = THREE.MathUtils.clamp(Math.abs(moveInput) + Math.abs(verticalInput) * 0.9, 0, 1);
							if (throttle > 0.1) {
								spawnThrusterTrail(drivingMachine, throttle);
							}
						}
					} else {
						movementStrength = 0;
					}

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

						for (let i = 0; i < critters.length; i += 1) {
							const critter = critters[i];
							critter.targetTimer -= delta;
							let effectiveSpeed = critter.speed;
							let chasingPlayer = false;
							if (critter.kind === 'spider') {
								const dxp = player.position.x - critter.group.position.x;
								const dzp = player.position.z - critter.group.position.z;
								const playerDistSq = dxp * dxp + dzp * dzp;
								const aggroRadius = 18;
								if (playerDistSq <= aggroRadius * aggroRadius) {
									const sx = Math.round(critter.group.position.x);
									const sz = Math.round(critter.group.position.z);
									const info = getTerrainInfo(sx, sz, currentWorld);
									const inSwamp = info.biome.id === 'swamp';
									const wantsAggro = currentWorld.id === 'mars' ? true : isNight || inSwamp;
									if (wantsAggro) {
										chasingPlayer = true;
										critter.target.set(player.position.x, player.position.z);
										critter.targetTimer = 0.15;
										effectiveSpeed *= 1.55;
									}
								}
							}

							let dx = critter.target.x - critter.group.position.x;
							let dz = critter.target.y - critter.group.position.z;
							let distSq = dx * dx + dz * dz;
							if (!chasingPlayer && (critter.targetTimer <= 0 || distSq < 0.75 * 0.75)) {
								pickCritterTarget(critter, i * 133 + Math.floor(time * 12));
								dx = critter.target.x - critter.group.position.x;
								dz = critter.target.y - critter.group.position.z;
								distSq = dx * dx + dz * dz;
							}

							if (distSq > 0.0001) {
								const dist = Math.sqrt(distSq);
								const step = effectiveSpeed * delta;
								const stepScale = Math.min(step / dist, 1);
								critter.group.position.x += dx * stepScale;
								critter.group.position.z += dz * stepScale;
								critter.group.rotation.y = Math.atan2(dx, dz);
								critter.phase += delta * 10 * (0.35 + effectiveSpeed * 0.18);
								if (critter.kind === 'sheep' || critter.kind === 'cow') {
									const gaitMul = critter.kind === 'cow' ? 0.75 : 0.95;
									const stepMul = critter.kind === 'cow' ? 2.3 : 2.8;
									const gait = Math.sin(critter.phase) * gaitMul * Math.min(1, step * stepMul);
									(critter.legs[0] ?? { rotation: { x: 0 } }).rotation.x = gait;
									(critter.legs[1] ?? { rotation: { x: 0 } }).rotation.x = -gait;
									(critter.legs[2] ?? { rotation: { x: 0 } }).rotation.x = -gait;
									(critter.legs[3] ?? { rotation: { x: 0 } }).rotation.x = gait;
								} else if (critter.kind === 'chicken') {
									const gait = Math.sin(critter.phase) * 0.9 * Math.min(1, step * 3.2);
									(critter.legs[0] ?? { rotation: { x: 0 } }).rotation.x = gait;
									(critter.legs[1] ?? { rotation: { x: 0 } }).rotation.x = -gait;
									const flap = Math.abs(Math.sin(critter.phase * 2.2)) * 0.85 * Math.min(1, step * 3.8);
									const wingL = critter.wings?.[0];
									const wingR = critter.wings?.[1];
									if (wingL && wingR) {
										wingL.rotation.z = 0.25 + flap;
										wingR.rotation.z = -(0.25 + flap);
									}
								} else if (critter.kind === 'spider') {
									const amp = 0.32 * Math.min(1, step * 3.2);
									for (const leg of critter.legs) {
										const baseZ = (leg.userData.baseZ as number) ?? 0;
										const phase = (leg.userData.phase as number) ?? 0;
										leg.rotation.z = baseZ + Math.sin(critter.phase + phase) * amp;
									}
								} else if (critter.kind === 'crab') {
									const amp = 0.4 * Math.min(1, step * 3.4);
									for (const leg of critter.legs) {
										const phase = (leg.userData.phase as number) ?? 0;
										leg.rotation.z = Math.sin(critter.phase + phase) * amp;
									}
								}
							} else {
								if (critter.kind === 'sheep' || critter.kind === 'cow') {
									for (const leg of critter.legs) {
										leg.rotation.x *= 0.85;
									}
								} else if (critter.kind === 'chicken') {
									for (const leg of critter.legs) {
										leg.rotation.x *= 0.82;
									}
									const wingL = critter.wings?.[0];
									const wingR = critter.wings?.[1];
									if (wingL && wingR) {
										wingL.rotation.z *= 0.78;
										wingR.rotation.z *= 0.78;
									}
								} else {
									for (const leg of critter.legs) {
										const baseZ = (leg.userData.baseZ as number) ?? 0;
										leg.rotation.z = baseZ + (leg.rotation.z - baseZ) * 0.85;
									}
								}
							}

							const cx = Math.round(critter.group.position.x);
							const cz = Math.round(critter.group.position.z);
							const surface = getHeightAt(cx, cz);
							const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
							const desiredY = Math.max(surface, fluidSurface) + 0.25;
							critter.group.position.y += (desiredY - critter.group.position.y) * Math.min(1, delta * 8);
							}
	
							machinerySpawnTimer -= delta;
							const roamingMachineryCount = machineries.reduce((count, machine) => count + (machine.isOwned ? 0 : 1), 0);
							if (roamingMachineryCount < machineryMaxActive && machinerySpawnTimer <= 0) {
								const spawned = spawnMachinery();
								machinerySpawnTimer = THREE.MathUtils.randFloat(8, 14) + (spawned ? 0 : 2.5);
							} else if (roamingMachineryCount >= machineryMaxActive) {
								machinerySpawnTimer = Math.max(machinerySpawnTimer, 1);
							}

							cardSpawnTimer -= delta;
							if (cards.length < cardsMaxActive && cardSpawnTimer <= 0) {
								const spawned = spawnCard();
								cardSpawnTimer = THREE.MathUtils.randFloat(4, 9) + (spawned ? 0 : 2);
							} else if (cards.length >= cardsMaxActive) {
								cardSpawnTimer = Math.max(cardSpawnTimer, 1);
							}

							let nearestMachine: MachineryInstance | null = null;
							let nearestMachineDist = Number.POSITIVE_INFINITY;

							for (let i = machineries.length - 1; i >= 0; i -= 1) {
								const machine = machineries[i];
								if (!machine.isOwned && Number.isFinite(machine.rentalRemaining) && machine.rentalRemaining > 0) {
									machine.rentalRemaining = Math.max(0, machine.rentalRemaining - delta);
									if (machine.rentalRemaining <= 0) {
										if (machine.boarded) {
											exitVehicle('expired');
										}
										removeMachineryInstance(machine);
										machinerySpawnTimer = Math.min(machinerySpawnTimer, 2.5);
										continue;
									}
								}

								if (!machine.isOwned && machine.rentalRemaining <= 0 && !machine.boarded) {
									machine.life -= delta;
									if (machine.life <= 0) {
										removeMachineryInstance(machine);
										machinerySpawnTimer = Math.min(machinerySpawnTimer, 3.5);
										continue;
									}
								}

								const canWander = !machine.isOwned && machine.rentalRemaining <= 0 && !machine.boarded;
								if (canWander) {
									machine.targetTimer -= delta;
									let dx = machine.target.x - machine.group.position.x;
									let dz = machine.target.y - machine.group.position.z;
									let distSq = dx * dx + dz * dz;
									if (machine.targetTimer <= 0 || distSq < 1.2 * 1.2) {
										pickMachineryTarget(machine, i * 311 + Math.floor(time * 9));
										dx = machine.target.x - machine.group.position.x;
										dz = machine.target.y - machine.group.position.z;
										distSq = dx * dx + dz * dz;
									}

									if (distSq > 0.0001) {
										const dist = Math.sqrt(distSq);
										const step = machine.speed * delta;
										const stepScale = Math.min(step / dist, 1);
										machine.group.position.x += dx * stepScale;
										machine.group.position.z += dz * stepScale;
										machine.group.rotation.y = Math.atan2(dx, dz);
										machine.phase += delta * (1.2 + machine.speed * 0.4);
									}
								} else {
									machine.phase += delta * 1.2;
								}

								if (!machine.boarded) {
									const mx = Math.round(machine.group.position.x);
									const mz = Math.round(machine.group.position.z);
									const surface = getHeightAt(mx, mz);
									const fluidSurface = Math.max(currentWorld.fluids.waterLevel, currentWorld.fluids.lavaLevel);
									const bob = Math.sin(time * 2.4 + machine.phase) * 0.08;
									const desiredY = Math.max(surface, fluidSurface) + machine.hover + bob;
									machine.group.position.y += (desiredY - machine.group.position.y) * Math.min(1, delta * 6);
									machine.group.rotation.x = lerp(machine.group.rotation.x, 0, Math.min(1, delta * 4));
									machine.group.rotation.z = lerp(machine.group.rotation.z, 0, Math.min(1, delta * 4));
								}

								const dxm = machine.group.position.x - machine.lastPos.x;
								const dzm = machine.group.position.z - machine.lastPos.z;
								const travel = Math.hypot(dxm, dzm);
								if (travel > 0.00001) {
									machine.wheelRoll += travel * 1.9;
								}
								machine.lastPos.copy(machine.group.position);

								for (const rotor of machine.parts.rotors) {
									rotor.rotation.y += delta * 7.5;
								}
								for (const gyro of machine.parts.gyros) {
									gyro.rotation.y += delta * 2.2;
								}
								for (const wheel of machine.parts.wheels) {
									wheel.rotation.x = machine.wheelRoll;
								}
								const armSwing = Math.sin(time * 2.6 + machine.phase) * 0.22;
								for (let p = 0; p < machine.parts.arms.length; p += 1) {
									const arm = machine.parts.arms[p];
									if (!arm.userData.baseRotSet) {
										arm.userData.baseRotSet = true;
										arm.userData.baseRotX = arm.rotation.x;
										arm.userData.baseRotY = arm.rotation.y;
										arm.userData.baseRotZ = arm.rotation.z;
									}
									const baseX = (arm.userData.baseRotX as number) ?? 0;
									const baseZ = (arm.userData.baseRotZ as number) ?? 0;
									const sign = p % 2 === 0 ? 1 : -1;
									arm.rotation.x = baseX + armSwing * sign;
									arm.rotation.z = baseZ + armSwing * sign * 0.65;
								}
								const thrusterPulse = 0.65 + Math.sin(time * 8.5 + machine.phase) * 0.35;
								const moving = travel > 0.002 || machine.boarded;
								for (const thruster of machine.parts.thrusters) {
									const glow = thruster.children[0] as THREE.Object3D | undefined;
									if (!glow) continue;
									if (!glow.userData.baseScaleZ) {
										glow.userData.baseScaleX = glow.scale.x;
										glow.userData.baseScaleY = glow.scale.y;
										glow.userData.baseScaleZ = glow.scale.z;
									}
									const baseZ = (glow.userData.baseScaleZ as number) ?? glow.scale.z;
									const boost = moving ? 1 : 0.45;
									glow.scale.z = baseZ * (0.85 + thrusterPulse * 0.95 * boost);
								}

								if (!activeMachine) {
									const dxp = player.position.x - machine.group.position.x;
									const dzp = player.position.z - machine.group.position.z;
									const dist = Math.hypot(dxp, dzp);
									if (dist < nearestMachineDist) {
										nearestMachineDist = dist;
										nearestMachine = machine;
									}
								}
							}

						for (let i = cards.length - 1; i >= 0; i -= 1) {
							const card = cards[i];
							card.life -= delta;
							if (card.life <= 0) {
								scene.remove(card.sprite);
								cards.splice(i, 1);
								cardSpawnTimer = Math.min(cardSpawnTimer, 2.5);
								continue;
							}
							const bob = Math.sin(time * 2.6 + card.phase) * 0.12;
							const pulse = 1 + Math.sin(time * 3.4 + card.phase) * 0.05;
							card.sprite.position.set(card.position.x, card.position.y + bob, card.position.z);
							card.sprite.scale.set(1.2 * pulse, 0.82 * pulse, 1);

							const dxp = player.position.x - card.sprite.position.x;
							const dzp = player.position.z - card.sprite.position.z;
							if (dxp * dxp + dzp * dzp <= 1.45 * 1.45) {
								collectCard(card);
							}
							}

							let nextHint: string | null = null;
							let nextAction: string | null = null;
							let nextAltAction: string | null = null;

							if (activeMachine) {
								const machine = activeMachine;
								const purchaseCost = getMachineryCost(machine.def.id);
								if (machine.isOwned) {
									nextHint = `${machine.def.name} (owned). Press F to disembark.`;
								} else if (Number.isFinite(machine.rentalRemaining) && machine.rentalRemaining > 0) {
									nextHint = `${machine.def.name} rental: ${formatClock(machine.rentalRemaining)} left. Press E to extend (+${machine.rentalDuration}s) for ${machine.rentalCost} SC. Press F to disembark.`;
									nextAction = crypto >= machine.rentalCost ? 'Extend' : null;
									nextAltAction = crypto >= purchaseCost ? 'Buy' : null;
								} else {
									nextHint = `${machine.def.name}. Press F to disembark.`;
								}
							} else if (nearestMachine && nearestMachineDist <= nearestMachine.parts.interactRadius) {
								const machine = nearestMachine;
								const purchaseCost = getMachineryCost(machine.def.id);
								const rarityLabel = RARITY_LABEL[machine.def.rarity];
								if (machine.isOwned) {
									nextHint = `Press E to board ${machine.def.name} (${rarityLabel}).`;
									nextAction = 'Board';
								} else if (machine.rentalRemaining > 0) {
									nextHint = `Press E to board ${machine.def.name} (${rarityLabel}). Rental: ${formatClock(machine.rentalRemaining)} left.`;
									nextAction = 'Board';
									nextAltAction = crypto >= purchaseCost ? 'Buy' : null;
								} else {
									const canRent = crypto >= machine.rentalCost;
									const canBuy = crypto >= purchaseCost;
									nextHint = `Press E to rent ${machine.def.name} (${rarityLabel}) for ${machine.rentalCost} SC (${machine.rentalDuration}s). Press B to buy for ${purchaseCost} SC.`;
									nextAction = canRent ? 'Rent' : null;
									nextAltAction = canBuy ? 'Buy' : null;
									if (!canRent && !canBuy) {
										nextHint = `${machine.def.name} (${rarityLabel}) rent ${machine.rentalCost} SC / buy ${purchaseCost} SC. You have ${crypto} SC.`;
									} else if (!canRent) {
										nextHint = `${machine.def.name} (${rarityLabel}) rent ${machine.rentalCost} SC (need ${machine.rentalCost - crypto} more). Press B to buy for ${purchaseCost} SC.`;
									} else if (!canBuy) {
										nextHint = `Press E to rent ${machine.def.name} (${rarityLabel}) for ${machine.rentalCost} SC (${machine.rentalDuration}s). Buy costs ${purchaseCost} SC.`;
									}
								}
							}
							if (interactionHint !== nextHint) interactionHint = nextHint;
							if (interactionActionLabel !== nextAction) interactionActionLabel = nextAction;
							if (interactionAltActionLabel !== nextAltAction) interactionAltActionLabel = nextAltAction;

							if (interactRequested) {
								interactRequested = false;
								if (activeMachine) {
									extendRental(activeMachine);
								} else if (nearestMachine && nearestMachineDist <= nearestMachine.parts.interactRadius) {
									if (nearestMachine.isOwned || nearestMachine.rentalRemaining > 0 || beginRental(nearestMachine)) {
										enterVehicle(nearestMachine);
									}
								} else {
									showToast('No machinery close enough.', 'warn');
								}
							}

								if (altInteractRequested) {
									altInteractRequested = false;
									if (activeMachine) {
										purchaseMachinery(activeMachine);
									} else if (nearestMachine && nearestMachineDist <= nearestMachine.parts.interactRadius) {
										purchaseMachinery(nearestMachine);
									} else {
										showToast('No machinery close enough.', 'warn');
									}
								}

								if (canDisembark !== Boolean(activeMachine)) {
									canDisembark = Boolean(activeMachine);
								}
							}

						cameraRig.position.copy(player.position);
						cameraRig.rotation.y = player.rotation.y;
						cameraRig.updateMatrixWorld();

						const walkBob = Math.sin(time * 8) * 0.04 * movementStrength;
						const rideBob = activeMachine ? Math.sin(time * 5 + activeMachine.phase) * 0.03 * movementStrength : walkBob;
						const cameraY = activeMachine ? activeMachine.parts.seatHeight + rideBob : playerEyeHeight + walkBob;
						camera.position.set(0, cameraY, 0);
						camera.rotation.set(cameraPitch, 0, 0);
						camera.updateMatrixWorld();

					viewEuler.set(cameraPitch, player.rotation.y, 0);
					viewDir.copy(forwardBase).applyEuler(viewEuler);

					if (viewArmRoot && viewHeldItem) {
						const held = hotbar[selectedHotbar];
						if (!held) {
							viewHeldType = null;
							viewHeldItem.visible = false;
						} else {
							viewHeldItem.visible = true;
							if (held !== viewHeldType) {
								viewHeldType = held;
								viewHeldItem.material = blockMats[held];
							}
						}

						const swingT =
							miningDown && miningRequired > 0
								? THREE.MathUtils.clamp(miningProgress / Math.max(0.01, miningRequired), 0, 1)
								: 0;
						const swing = swingT > 0 ? Math.sin(swingT * Math.PI) : 0;
						const bobX = Math.sin(time * 4) * 0.02 * movementStrength;
						const bobY = Math.abs(Math.sin(time * 8)) * 0.03 * movementStrength;
						viewArmRoot.position.set(0.68 + bobX, -0.74 + bobY, -1.05);
						viewArmRoot.rotation.set(-0.55 - swing * 1.15, 0.62 + swing * 0.08, 0.18 + swing * 0.65);
					}

						aimedBlock = null;
						hasAimedHitPoint = false;
						mineRaycaster.setFromCamera(mineNdc, camera);
						mineRaycaster.far = maxMineDistance;
						const mineHits = mineRaycaster.intersectObjects(mineableMeshes, false);
						for (const hit of mineHits) {
						const mesh = hit.object as THREE.InstancedMesh;
						const type = mesh.userData?.blockType as BlockType | undefined;
						if (!type || !isMineableType(type)) {
							continue;
						}
						if (hit.instanceId == null) {
							continue;
						}
						mesh.getMatrixAt(hit.instanceId, mineMatrix);
						minePos.setFromMatrixPosition(mineMatrix);
							const x = Math.round(minePos.x);
							const y = Math.floor(minePos.y);
							const z = Math.round(minePos.z);
							aimedBlock = { x, y, z, type, distance: hit.distance };
							aimedHitPoint.copy(hit.point);
							hasAimedHitPoint = true;
							break;
						}

						if (aimedBlock) {
							blockOutline.visible = true;
							blockOutline.position.set(aimedBlock.x, aimedBlock.y + 0.5, aimedBlock.z);
							blockBreakOverlay.position.copy(blockOutline.position);
						} else {
							blockOutline.visible = false;
							blockBreakOverlay.visible = false;
							blockBreakMat.opacity = 0;
							lastBreakStage = -1;
						}

						if (!isTransitioning && placeRequested && aimedBlock && hasAimedHitPoint) {
							const held = hotbar[selectedHotbar];
							if (held && getInventoryCount(held) > 0 && held !== 'water' && held !== 'lava') {
								tempVec3.set(
									aimedHitPoint.x - aimedBlock.x,
									aimedHitPoint.y - (aimedBlock.y + 0.5),
									aimedHitPoint.z - aimedBlock.z
								);
								const ax = Math.abs(tempVec3.x);
								const ay = Math.abs(tempVec3.y);
								const az = Math.abs(tempVec3.z);
								let ox = 0;
								let oy = 0;
								let oz = 0;
								if (ax >= ay && ax >= az) {
									ox = tempVec3.x >= 0 ? 1 : -1;
								} else if (ay >= ax && ay >= az) {
									oy = tempVec3.y >= 0 ? 1 : -1;
								} else {
									oz = tempVec3.z >= 0 ? 1 : -1;
								}

								const px = aimedBlock.x + ox;
								const py = aimedBlock.y + oy;
								const pz = aimedBlock.z + oz;
								if (py > 0) {
									const placeKey = blockKey(px, py, pz);
									const edits = getWorldEdits(currentWorld.id);
									if (!edits.added.has(placeKey)) {
										const occupied = getLoadedBlockType(px, py, pz);
										const replaceable = occupied === null || occupied === 'water' || occupied === 'lava';

										const baseHeight = getHeightAt(px, pz);
										const insideNaturalSolid = py < baseHeight && !edits.removed.has(placeKey);

										const playerMinX = player.position.x - playerWidth / 2;
										const playerMaxX = player.position.x + playerWidth / 2;
										const playerMinY = player.position.y;
										const playerMaxY = player.position.y + playerHeight;
										const playerMinZ = player.position.z - playerDepth / 2;
										const playerMaxZ = player.position.z + playerDepth / 2;
										const blockMinX = px - 0.5;
										const blockMaxX = px + 0.5;
										const blockMinY = py;
										const blockMaxY = py + 1;
										const blockMinZ = pz - 0.5;
										const blockMaxZ = pz + 0.5;
										const intersectsPlayer =
											blockMinX < playerMaxX &&
											blockMaxX > playerMinX &&
											blockMinY < playerMaxY &&
											blockMaxY > playerMinY &&
											blockMinZ < playerMaxZ &&
											blockMaxZ > playerMinZ;

										if (replaceable && !insideNaturalSolid && !intersectsPlayer) {
											edits.added.set(placeKey, held);
											if (consumeFromInventory(held, 1)) {
												rebuildChunkAt(px, pz);
												playBlockPlaceSound(held);
											} else {
												edits.added.delete(placeKey);
											}
										}
									}
								}
							}
						}
						placeRequested = false;

						if (!isTransitioning && miningDown && aimedBlock) {
							const nextKey = blockKey(aimedBlock.x, aimedBlock.y, aimedBlock.z);
							const required = blockBreakSeconds[aimedBlock.type] ?? 0.7;
							if (nextKey !== miningTargetKey) {
								miningTargetKey = nextKey;
								miningProgress = 0;
								miningRequired = required;
								lastBreakStage = -1;
							} else {
								miningRequired = required;
							}
							miningProgress += delta;
							const t = THREE.MathUtils.clamp(miningProgress / Math.max(0.01, miningRequired), 0, 1);
							const stage = Math.min(9, Math.floor(t * 10));
							if (stage !== lastBreakStage) {
								lastBreakStage = stage;
								blockBreakMat.map = breakStageTextures[stage] ?? breakStageTextures[0];
								blockBreakMat.needsUpdate = true;
							}
							blockBreakMat.opacity = 0.25 + t * 0.75;
							blockBreakOverlay.visible = true;

							if (miningProgress >= miningRequired) {
								if (breakTargetBlock(aimedBlock)) {
									spawnBlockBreakParticles(aimedBlock.x, aimedBlock.y, aimedBlock.z, aimedBlock.type);
									playBlockBreakSound(aimedBlock.type);
								}
								miningProgress = 0;
								miningTargetKey = null;
								lastBreakStage = -1;
								blockBreakOverlay.visible = false;
								blockBreakMat.opacity = 0;
							}
						} else {
							miningProgress = 0;
							miningTargetKey = null;
							miningRequired = 0;
							lastBreakStage = -1;
							blockBreakOverlay.visible = false;
							blockBreakMat.opacity = 0;
						}

						for (const portal of portals) {
							const coreMat = portal.core.material as THREE.MeshStandardMaterial;
							coreMat.opacity = 0.6 + Math.sin(time * 2.4 + portal.pulseOffset) * 0.15;
							const coreMap = coreMat.map;
							if (coreMap) {
								coreMap.offset.y = (coreMap.offset.y + delta * 0.12) % 1;
								coreMap.offset.x = (coreMap.offset.x + delta * 0.04) % 1;
							}
							portal.core.rotation.z += delta * 0.6;
						}

						// Cheap "water movement" so oceans/rivers feel less static.
						waterTex.offset.x = (waterTex.offset.x + delta * 0.012) % 1;
						waterTex.offset.y = (waterTex.offset.y + delta * 0.008) % 1;
						lavaTex.offset.x = (lavaTex.offset.x + delta * 0.02) % 1;
						lavaTex.offset.y = (lavaTex.offset.y + delta * 0.014) % 1;

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

					for (let i = vehicleParticles.length - 1; i >= 0; i -= 1) {
						const particle = vehicleParticles[i];
						particle.velocity.y += gravity * 0.05 * delta;
						particle.mesh.position.addScaledVector(particle.velocity, delta);
						particle.life -= delta;
						if (particle.life <= 0) {
							scene.remove(particle.mesh);
							vehicleParticles.splice(i, 1);
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
				document.removeEventListener('pointerlockchange', handlePointerLockChange);
				renderer.domElement.removeEventListener('pointerdown', handleMouseDown);
				renderer.domElement.removeEventListener('contextmenu', handleContextMenu);
				joystickEl?.removeEventListener('pointerdown', handleJoystickDown);
				joystickEl?.removeEventListener('pointermove', handleJoystickMove);
				joystickEl?.removeEventListener('pointerup', handleJoystickUp);
				joystickEl?.removeEventListener('pointercancel', handleJoystickUp);
					jumpEl?.removeEventListener('pointerdown', handleJumpDown);
					jumpEl?.removeEventListener('pointerup', handleJumpUp);
					jumpEl?.removeEventListener('pointercancel', handleJumpUp);
					downEl?.removeEventListener('pointerdown', handleDownDown);
					downEl?.removeEventListener('pointerup', handleDownUp);
					downEl?.removeEventListener('pointercancel', handleDownUp);
					renderer.domElement.removeEventListener('pointerdown', handleLookDown);
					renderer.domElement.removeEventListener('pointermove', handleLookMove);
					renderer.domElement.removeEventListener('pointerup', handleLookUp);
					renderer.domElement.removeEventListener('pointercancel', handleLookUp);
				container?.removeChild(renderer.domElement);
				worldJump = null;
				if (toastTimeout) {
					clearTimeout(toastTimeout);
					toastTimeout = null;
				}
					hudToast = null;
					interactionHint = null;
					interactionActionLabel = null;
					interactionAltActionLabel = null;
					canDisembark = false;
					interactRequested = false;
					altInteractRequested = false;
					disembarkRequested = false;
					deployMachinery = null;

						blockGeo.dispose();
						fallingBlockGeo.dispose();
						particleGeo.dispose();
						portalParticleGeo.dispose();
						portalFrameGeo.dispose();
						portalCoreGeo.dispose();
						viewModelRoot?.removeFromParent();
						viewSkinMat?.dispose();
						viewSleeveMat?.dispose();

						scene.remove(blockOutline);
						(blockOutline.geometry as THREE.BufferGeometry).dispose();
						blockOutlineMat.dispose();
						scene.remove(blockBreakOverlay);
						(blockBreakOverlay.geometry as THREE.BufferGeometry).dispose();
						blockBreakMat.dispose();
						for (const tex of breakStageTextures) {
							tex.dispose();
						}
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
					for (const mat of vehicleParticleMats.values()) {
						mat.dispose();
					}
					vehicleParticleMats.clear();
					for (const mat of breakParticleMats.values()) {
						mat.dispose();
					}
				breakParticleMats.clear();
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
					sheepWoolMat.dispose();
					sheepSkinMat.dispose();
					cowHideMat.dispose();
					cowSpotMat.dispose();
					cowSnoutMat.dispose();
					chickenFeatherMat.dispose();
					chickenBeakMat.dispose();
					chickenCombMat.dispose();
					spiderMat.dispose();
					spiderEyeMat.dispose();
					crabMat.dispose();
					clearMachinery();
					clearCards();
					machineryHullMat.dispose();
					machineryDetailMat.dispose();
					machineryGlassMat.dispose();
					for (const mat of rarityAccentMats.values()) {
						mat.dispose();
					}
					rarityAccentMats.clear();
					for (const mat of machineryLabelMats.values()) {
						mat.dispose();
					}
					machineryLabelMats.clear();
					for (const tex of machineryLabelTextures.values()) {
						tex.dispose();
					}
					machineryLabelTextures.clear();
					for (const mat of cardMats.values()) {
						mat.dispose();
					}
					cardMats.clear();
					for (const tex of cardTextures.values()) {
						tex.dispose();
					}
					cardTextures.clear();

					clearChunks();
					clearPortals();
					clearBots();
					clearCritters();
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
					for (const particle of vehicleParticles) {
						scene.remove(particle.mesh);
					}
					vehicleParticles.length = 0;
					pendingDetonations.length = 0;
					world.removeRigidBody(playerBody);
					world.removeCharacterController(controller);

				for (const audio of bgmCache.values()) {
					audio.pause();
				}
				sfxCtx?.close().catch(() => {});

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
			<div class="currency">SC: {formatCrypto(crypto)}</div>
			<div class="world-buttons">
					<button type="button" on:click={() => jumpWorld('earth')}>Earth</button>
					<button type="button" on:click={() => jumpWorld('mars')}>Mars</button>
					<button type="button" on:click={() => jumpWorld('moon')}>Moon</button>
				</div>
				<details class="collection">
					<summary>Collection</summary>
					<div class="collection-section">
						<div class="collection-title">Machinery</div>
						{#each MACHINERY_CATALOG as item (item.id)}
								{#if getOwnedMachineryCount(item.id) > 0}
									<div class="collection-row">
										<div class="collection-name">{item.name}</div>
										<div class="collection-meta">
											<span class="collection-rarity">{RARITY_LABEL[item.rarity]}</span>
											<span class="collection-count">x{getOwnedMachineryCount(item.id)}</span>
											<button
												type="button"
												class="collection-action"
												disabled={!deployMachinery}
												on:click={() => deployOwned(item.id)}
											>
												Deploy
											</button>
										</div>
									</div>
								{/if}
						{/each}
						{#if !MACHINERY_CATALOG.some((entry) => getOwnedMachineryCount(entry.id) > 0)}
							<div class="collection-empty">No machinery yet. Catch and buy wandering machinery.</div>
						{/if}
					</div>
					<div class="collection-section">
						<div class="collection-title">Cards</div>
						{#each CARD_CATALOG as card (card.id)}
							{#if getCollectedCardCount(card.id) > 0}
								<div class="collection-row">
									<div class="collection-name">{card.name}</div>
									<div class="collection-meta">
										<span class="collection-rarity">{RARITY_LABEL[card.rarity]}</span>
										<span class="collection-count">x{getCollectedCardCount(card.id)}</span>
									</div>
								</div>
							{/if}
						{/each}
						{#if !CARD_CATALOG.some((entry) => getCollectedCardCount(entry.id) > 0)}
							<div class="collection-empty">No cards yet. Explore to find tickets.</div>
						{/if}
					</div>
				</details>
				{#if debugCameraEnabled}
					<details class="debug-camera">
						<summary>Debug Camera</summary>
						<div class="debug-camera-grid" role="group" aria-label="Camera offset from player head top">
							<div class="debug-camera-row">
								<label for="debug-camera-x">X</label>
								<input
									id="debug-camera-x"
									type="range"
									min="-12"
									max="12"
									step="0.1"
									value={debugCameraOffsetX}
									on:input={(event) => {
										debugCameraOffsetX = (event.currentTarget as HTMLInputElement).valueAsNumber;
										saveDebugCameraSettings();
									}}
								/>
								<output for="debug-camera-x">{debugCameraOffsetX.toFixed(2)}</output>
							</div>
							<div class="debug-camera-row">
								<label for="debug-camera-y">Y</label>
								<input
									id="debug-camera-y"
									type="range"
									min="-6"
									max="12"
									step="0.1"
									value={debugCameraOffsetY}
									on:input={(event) => {
										debugCameraOffsetY = (event.currentTarget as HTMLInputElement).valueAsNumber;
										saveDebugCameraSettings();
									}}
								/>
								<output for="debug-camera-y">{debugCameraOffsetY.toFixed(2)}</output>
							</div>
							<div class="debug-camera-row">
								<label for="debug-camera-z">Z</label>
								<input
									id="debug-camera-z"
									type="range"
									min="-20"
									max="20"
									step="0.1"
									value={debugCameraOffsetZ}
									on:input={(event) => {
										debugCameraOffsetZ = (event.currentTarget as HTMLInputElement).valueAsNumber;
										saveDebugCameraSettings();
									}}
								/>
								<output for="debug-camera-z">{debugCameraOffsetZ.toFixed(2)}</output>
							</div>
							<div class="debug-camera-actions">
								<button type="button" on:click={resetDebugCamera}>Reset</button>
								<div class="debug-camera-note">Offsets are relative to player head-top.</div>
							</div>
						</div>
					</details>
				{/if}
					<p>Walk into a portal to swap worlds. W / A / S / D or Arrow keys. Click the world to capture the mouse (Esc to release), then move to look. Space to jump (ascend in vehicles), Shift/Ctrl to descend in vehicles. Hold left click (or touch and hold) to mine the highlighted block and collect it into your hotbar. Right click to place the selected hotbar block (keys 1-9). Press E near machinery to rent/board it, B to buy it, and F to disembark. Explore to find collectible tickets.</p>
			</div>
			<div class="scene" bind:this={container}></div>
			{#if hudToast}
				<div class="toast" class:warn={hudToast.tone === 'warn'} class:error={hudToast.tone === 'error'}>
					{hudToast.text}
				</div>
				{/if}
				{#if interactionHint}
					<div class="interaction">
						<div class="interaction-text">{interactionHint}</div>
						{#if interactionActionLabel || interactionAltActionLabel}
							<div class="interaction-actions">
								{#if interactionActionLabel}
									<button type="button" on:click={requestInteract}>{interactionActionLabel}</button>
								{/if}
								{#if interactionAltActionLabel}
									<button type="button" class="alt" on:click={requestAltInteract}>{interactionAltActionLabel}</button>
								{/if}
							</div>
						{/if}
					</div>
				{/if}
			<div class="crosshair" aria-hidden="true"></div>
			<div class="hotbar" aria-label="Backpack">
			{#each hotbar as slot, idx}
				<div class="hotbar-slot" class:selected={idx === selectedHotbar}>
					{#if slot}
						<div
							class="hotbar-icon"
							style={`background-image: url('${getBlockIcon(slot)}')`}
							title={slot}
						></div>
						<div class="hotbar-count">{getInventoryCount(slot)}</div>
					{/if}
				</div>
			{/each}
			</div>
			<div class="touch-controls">
				<div class="touch-pad joystick" bind:this={joystickEl}>
					<div class="thumb" bind:this={joystickThumbEl}></div>
				</div>
				<div class="touch-pad jump" bind:this={jumpEl}>Jump</div>
				<div class="touch-pad down" bind:this={downEl}>Down</div>
				{#if interactionActionLabel}
					<button type="button" class="touch-pad interact" on:pointerdown|preventDefault={() => requestInteract()}>
						{interactionActionLabel}
					</button>
				{/if}
				{#if interactionAltActionLabel}
					<button type="button" class="touch-pad interact alt" on:pointerdown|preventDefault={() => requestAltInteract()}>
						{interactionAltActionLabel}
					</button>
				{/if}
				{#if canDisembark}
					<button type="button" class="touch-pad exit" on:pointerdown|preventDefault={() => requestDisembark()}>
						Exit
					</button>
				{/if}
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

	.hud .currency {
		font-size: 12px;
		letter-spacing: 0.18em;
		text-transform: uppercase;
		color: rgba(249, 209, 140, 0.95);
		margin-bottom: 8px;
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

		.collection {
			margin-top: 10px;
			padding-top: 10px;
			border-top: 1px solid rgba(143, 177, 185, 0.22);
		}

		.collection summary {
			cursor: pointer;
			user-select: none;
			font-size: 11px;
			letter-spacing: 0.14em;
			text-transform: uppercase;
			color: rgba(183, 241, 255, 0.9);
			outline: none;
		}

		.collection-section {
			margin-top: 10px;
			display: grid;
			gap: 6px;
		}

		.collection-title {
			font-size: 10px;
			letter-spacing: 0.16em;
			text-transform: uppercase;
			color: rgba(183, 241, 255, 0.75);
		}

		.collection-row {
			display: flex;
			align-items: center;
			justify-content: space-between;
			gap: 12px;
			font-size: 12px;
			padding: 6px 8px;
			border-radius: 10px;
			background: rgba(8, 12, 15, 0.38);
			border: 1px solid rgba(143, 177, 185, 0.22);
		}

		.collection-name {
			color: rgba(232, 243, 246, 0.92);
		}

		.collection-meta {
			display: flex;
			gap: 10px;
			align-items: baseline;
			font-variant-numeric: tabular-nums;
			color: rgba(232, 243, 246, 0.75);
		}

		.collection-rarity {
			font-size: 11px;
			letter-spacing: 0.1em;
			text-transform: uppercase;
			color: rgba(183, 241, 255, 0.78);
		}

			.collection-count {
				font-weight: 600;
				color: rgba(249, 209, 140, 0.9);
			}

			.collection-action {
				appearance: none;
				border: 1px solid rgba(143, 177, 185, 0.35);
				background: rgba(18, 26, 32, 0.62);
				color: rgba(232, 243, 246, 0.9);
				border-radius: 999px;
				padding: 4px 8px;
				font-size: 10px;
				letter-spacing: 0.12em;
				text-transform: uppercase;
				cursor: pointer;
			}

			.collection-action:hover:not(:disabled) {
				background: rgba(26, 38, 46, 0.72);
				border-color: rgba(183, 241, 255, 0.35);
			}

			.collection-action:disabled {
				opacity: 0.45;
				cursor: default;
			}

			.collection-empty {
				font-size: 12px;
				opacity: 0.75;
			}

		.debug-camera {
			margin-top: 10px;
			padding-top: 10px;
			border-top: 1px solid rgba(143, 177, 185, 0.22);
		}

		.debug-camera summary {
			cursor: pointer;
			user-select: none;
			font-size: 11px;
			letter-spacing: 0.14em;
			text-transform: uppercase;
			color: rgba(183, 241, 255, 0.9);
			outline: none;
		}

		.debug-camera-grid {
			margin-top: 10px;
			display: grid;
			gap: 10px;
		}

		.debug-camera-row {
			display: grid;
			grid-template-columns: 16px 1fr 54px;
			align-items: center;
			gap: 10px;
		}

		.debug-camera-row label {
			font-size: 12px;
			font-weight: 600;
			color: rgba(232, 243, 246, 0.92);
		}

		.debug-camera-row input[type='range'] {
			width: 100%;
		}

		.debug-camera-row output {
			font-variant-numeric: tabular-nums;
			text-align: right;
			font-size: 12px;
			color: rgba(232, 243, 246, 0.85);
		}

		.debug-camera-actions {
			display: grid;
			grid-template-columns: auto 1fr;
			align-items: center;
			gap: 10px;
		}

		.debug-camera-actions button {
			appearance: none;
			border: 1px solid rgba(143, 177, 185, 0.35);
			background: rgba(18, 26, 32, 0.62);
			color: rgba(232, 243, 246, 0.92);
			border-radius: 999px;
			padding: 6px 10px;
			font-size: 11px;
			letter-spacing: 0.12em;
			text-transform: uppercase;
			cursor: pointer;
		}

		.debug-camera-actions button:hover {
			background: rgba(26, 38, 46, 0.72);
			border-color: rgba(183, 241, 255, 0.35);
		}

		.debug-camera-note {
			font-size: 12px;
			opacity: 0.75;
		}

		.crosshair {
			position: absolute;
			left: 50%;
			top: 50%;
			width: 16px;
			height: 16px;
			transform: translate(-50%, -50%);
			z-index: 4;
			pointer-events: none;
		}

		.crosshair::before,
		.crosshair::after {
			content: '';
			position: absolute;
			background: rgba(255, 250, 242, 0.92);
			box-shadow: 0 0 0 1px rgba(6, 8, 10, 0.55);
		}

		.crosshair::before {
			left: 50%;
			top: 2px;
			width: 2px;
			height: 12px;
			transform: translateX(-50%);
		}

		.crosshair::after {
			top: 50%;
			left: 2px;
			width: 12px;
			height: 2px;
			transform: translateY(-50%);
		}

		.hotbar {
			position: absolute;
			left: 50%;
			bottom: calc(16px + env(safe-area-inset-bottom));
			transform: translateX(-50%);
			z-index: 4;
			pointer-events: none;
			display: grid;
			grid-auto-flow: column;
			gap: 8px;
			padding: 10px 12px;
			border-radius: 16px;
			background: rgba(12, 18, 22, 0.62);
			backdrop-filter: blur(10px);
			border: 1px solid rgba(143, 177, 185, 0.35);
		}

		.hotbar-slot {
			width: 46px;
			height: 46px;
			border-radius: 12px;
			background: rgba(8, 12, 15, 0.55);
			border: 1px solid rgba(143, 177, 185, 0.35);
			position: relative;
			overflow: hidden;
		}

		.hotbar-slot.selected {
			border-color: rgba(249, 209, 140, 0.9);
			box-shadow: 0 0 0 2px rgba(249, 209, 140, 0.22);
		}

		.hotbar-icon {
			position: absolute;
			inset: 8px;
			background-size: cover;
			background-position: center;
			image-rendering: pixelated;
			filter: drop-shadow(0 2px 3px rgba(0, 0, 0, 0.45));
		}

		.hotbar-count {
			position: absolute;
			right: 7px;
			bottom: 5px;
			font-size: 12px;
			font-weight: 600;
			color: rgba(255, 250, 242, 0.95);
			text-shadow: 0 1px 2px rgba(0, 0, 0, 0.7);
		}

		.touch-controls {
			position: fixed;
			inset: 0;
			padding: env(safe-area-inset-top) env(safe-area-inset-right) env(safe-area-inset-bottom) env(safe-area-inset-left);
			pointer-events: none;
		display: none;
		z-index: 3;
	}

	.toast {
		position: absolute;
		left: 50%;
		top: calc(18px + env(safe-area-inset-top));
		transform: translateX(-50%);
		z-index: 6;
		padding: 10px 14px;
		border-radius: 999px;
		background: rgba(10, 16, 20, 0.8);
		backdrop-filter: blur(10px);
		border: 1px solid rgba(143, 177, 185, 0.35);
		color: rgba(232, 243, 246, 0.92);
		font-size: 12px;
		letter-spacing: 0.06em;
		max-width: min(92vw, 560px);
		text-align: center;
	}

	.toast.warn {
		border-color: rgba(249, 209, 140, 0.55);
	}

	.toast.error {
		border-color: rgba(255, 100, 90, 0.6);
		color: rgba(255, 221, 214, 0.95);
	}

	.interaction {
		position: absolute;
		left: 50%;
		bottom: calc(92px + env(safe-area-inset-bottom));
		transform: translateX(-50%);
		z-index: 6;
		display: flex;
		align-items: center;
		gap: 10px;
		padding: 10px 12px;
		border-radius: 16px;
		background: rgba(12, 18, 22, 0.62);
		backdrop-filter: blur(10px);
		border: 1px solid rgba(143, 177, 185, 0.35);
		pointer-events: none;
	}

		.interaction-text {
			font-size: 12px;
			color: rgba(232, 243, 246, 0.9);
			letter-spacing: 0.06em;
		}

		.interaction-actions {
			display: flex;
			gap: 8px;
		}

		.interaction button {
			pointer-events: auto;
			appearance: none;
			border: 1px solid rgba(249, 209, 140, 0.7);
			background: rgba(249, 209, 140, 0.12);
		color: rgba(255, 250, 242, 0.95);
		border-radius: 999px;
		padding: 8px 12px;
		font-size: 11px;
		letter-spacing: 0.12em;
		text-transform: uppercase;
		cursor: pointer;
	}

		.interaction button:hover {
			background: rgba(249, 209, 140, 0.18);
		}

		.interaction button.alt {
			border-color: rgba(183, 241, 255, 0.55);
			background: rgba(183, 241, 255, 0.12);
			color: rgba(232, 243, 246, 0.92);
		}

		.interaction button.alt:hover {
			background: rgba(183, 241, 255, 0.18);
		}

		.touch-pad {
			appearance: none;
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

		.touch-pad.down {
			right: calc(124px + env(safe-area-inset-right));
			bottom: calc(18px + env(safe-area-inset-bottom));
		}

		.touch-pad.interact {
			right: calc(18px + env(safe-area-inset-right));
			bottom: calc(122px + env(safe-area-inset-bottom));
			width: 110px;
			height: 64px;
			border-radius: 18px;
		}

		.touch-pad.interact.alt {
			bottom: calc(196px + env(safe-area-inset-bottom));
			border-color: rgba(183, 241, 255, 0.55);
			background: rgba(183, 241, 255, 0.08);
			color: rgba(232, 243, 246, 0.9);
		}

		.touch-pad.exit {
			right: calc(18px + env(safe-area-inset-right));
			bottom: calc(270px + env(safe-area-inset-bottom));
			width: 92px;
			height: 56px;
			border-radius: 18px;
			border-color: rgba(255, 141, 126, 0.55);
			background: rgba(255, 100, 90, 0.08);
			color: rgba(255, 221, 214, 0.92);
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
