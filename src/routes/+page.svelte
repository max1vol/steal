<script lang="ts">
	import { onMount } from 'svelte';
	import * as THREE from 'three';
	import RAPIER from '@dimforge/rapier3d-compat';

	let container: HTMLDivElement | null = null;
	let joystickEl: HTMLDivElement | null = null;
	let joystickThumbEl: HTMLDivElement | null = null;
	let lookPadEl: HTMLDivElement | null = null;

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

			const camera = new THREE.PerspectiveCamera(65, 1, 0.1, 300);
			const cameraRig = new THREE.Group();
			const cameraOffset = new THREE.Vector3(0, 4.6, 7.5);
			const defaultCameraDistance = cameraOffset.length();
			let cameraDistance = defaultCameraDistance;
			const collisionDampIn = 8;
			const collisionDampOut = 3.5;
			camera.position.copy(cameraOffset);
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

			const grassTopTex = loadTexture('/textures/grass_top.png');
			const grassSideTex = loadTexture('/textures/grass_side.png');
			const dirtTex = loadTexture('/textures/dirt.png');
			const stoneTex = loadTexture('/textures/stone.png');

			const grassTopMat = new THREE.MeshStandardMaterial({ map: grassTopTex, roughness: 0.95 });
			const grassSideMat = new THREE.MeshStandardMaterial({ map: grassSideTex, roughness: 0.95 });
			const dirtMat = new THREE.MeshStandardMaterial({ map: dirtTex, roughness: 1 });
			const stoneMat = new THREE.MeshStandardMaterial({ map: stoneTex, roughness: 1 });

			const blockGeo = new THREE.BoxGeometry(1, 1, 1);
			const grassMats = [grassSideMat, grassSideMat, grassTopMat, dirtMat, grassSideMat, grassSideMat];
			const dirtMats = [dirtMat, dirtMat, dirtMat, dirtMat, dirtMat, dirtMat];
			const stoneMats = [stoneMat, stoneMat, stoneMat, stoneMat, stoneMat, stoneMat];

			const world = new RAPIER.World({ x: 0, y: -9.81, z: 0 });

			const terrainSize = 24;
			const half = Math.floor(terrainSize / 2);
			const heights: number[][] = [];
			const terrainMeshes: THREE.Mesh[] = [];
			const cameraOccluders: THREE.Mesh[] = [];

			const heightNoise = (x: number, z: number) => {
				return (
					Math.sin(x * 0.35) * 0.9 +
					Math.cos(z * 0.27) * 0.9 +
					Math.sin((x + z) * 0.2) * 0.6
				);
			};

			const computeHeight = (x: number, z: number) => {
				const base = 3;
				const height = base + heightNoise(x, z) * 2.1;
				return THREE.MathUtils.clamp(Math.round(height), 1, 7);
			};

			for (let ix = 0; ix < terrainSize; ix += 1) {
				heights[ix] = [];
				for (let iz = 0; iz < terrainSize; iz += 1) {
					const worldX = ix - half;
					const worldZ = iz - half;
					const height = computeHeight(worldX, worldZ);
					heights[ix][iz] = height;

					for (let y = 0; y < height; y += 1) {
						let mats = dirtMats;
						if (y === height - 1) {
							mats = grassMats;
						} else if (y < height - 3) {
							mats = stoneMats;
						}

						const block = new THREE.Mesh(blockGeo, mats);
						block.position.set(worldX, y + 0.5, worldZ);
						terrainMeshes.push(block);
						cameraOccluders.push(block);
						scene.add(block);
					}

					const columnBody = world.createRigidBody(
						RAPIER.RigidBodyDesc.fixed().setTranslation(worldX, height / 2, worldZ)
					);
					world.createCollider(RAPIER.ColliderDesc.cuboid(0.5, height / 2, 0.5), columnBody);
				}
			}

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

			const getHeightAt = (x: number, z: number) => {
				const ix = THREE.MathUtils.clamp(Math.round(x + half), 0, terrainSize - 1);
				const iz = THREE.MathUtils.clamp(Math.round(z + half), 0, terrainSize - 1);
				return heights[ix][iz];
			};

			player.position.set(0, getHeightAt(0, 0), 0);
			scene.add(player);

			const input = {
				forward: false,
				back: false,
				left: false,
				right: false,
				jump: false
			};

			const handleKeyDown = (event: KeyboardEvent) => {
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
				lookDelta: 0,
				mouseDown: false,
				lastMouseX: 0
			};

			const handleMouseDown = (event: PointerEvent) => {
				if (event.pointerType !== 'mouse' || event.button !== 0) {
					return;
				}
				pointerState.mouseDown = true;
				pointerState.lastMouseX = event.clientX;
				renderer.domElement.setPointerCapture(event.pointerId);
			};

			const handleMouseMove = (event: PointerEvent) => {
				if (event.pointerType !== 'mouse' || !pointerState.mouseDown) {
					return;
				}
				const deltaX = event.clientX - pointerState.lastMouseX;
				pointerState.lastMouseX = event.clientX;
				pointerState.lookDelta += deltaX * 0.003;
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
				lastX: 0
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
				if (event.pointerType !== 'touch' || !joystickEl) {
					return;
				}
				joystickState.pointerId = event.pointerId;
				joystickEl.setPointerCapture(event.pointerId);
				updateJoystickBounds();
				handleJoystickMove(event);
				event.preventDefault();
			};

			const handleJoystickMove = (event: PointerEvent) => {
				if (event.pointerId !== joystickState.pointerId) {
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
				if (event.pointerId !== joystickState.pointerId) {
					return;
				}
				joystickState.pointerId = null;
				moveAxis.set(0, 0);
				updateJoystickThumb(0, 0);
				joystickEl?.releasePointerCapture(event.pointerId);
			};

			const handleLookDown = (event: PointerEvent) => {
				if (event.pointerType !== 'touch' || !lookPadEl) {
					return;
				}
				lookState.pointerId = event.pointerId;
				lookState.lastX = event.clientX;
				lookPadEl.setPointerCapture(event.pointerId);
				event.preventDefault();
			};

			const handleLookMove = (event: PointerEvent) => {
				if (event.pointerId !== lookState.pointerId) {
					return;
				}
				const dx = event.clientX - lookState.lastX;
				lookState.lastX = event.clientX;
				pointerState.lookDelta += dx * 0.004;
				event.preventDefault();
			};

			const handleLookUp = (event: PointerEvent) => {
				if (event.pointerId !== lookState.pointerId) {
					return;
				}
				lookState.pointerId = null;
				lookPadEl?.releasePointerCapture(event.pointerId);
			};

			joystickEl?.addEventListener('pointerdown', handleJoystickDown, { passive: false });
			joystickEl?.addEventListener('pointermove', handleJoystickMove, { passive: false });
			joystickEl?.addEventListener('pointerup', handleJoystickUp);
			joystickEl?.addEventListener('pointercancel', handleJoystickUp);
			lookPadEl?.addEventListener('pointerdown', handleLookDown, { passive: false });
			lookPadEl?.addEventListener('pointermove', handleLookMove, { passive: false });
			lookPadEl?.addEventListener('pointerup', handleLookUp);
			lookPadEl?.addEventListener('pointercancel', handleLookUp);

			const fallingBlocks: Array<{ mesh: THREE.Mesh; body: RAPIER.RigidBody }> = [];

			const spawnFallingBlock = () => {
				const spawnX = THREE.MathUtils.randFloat(-half + 2, half - 2);
				const spawnZ = THREE.MathUtils.randFloat(-half + 2, half - 2);
				const spawnY = THREE.MathUtils.randFloat(10, 16);
				const block = new THREE.Mesh(blockGeo, stoneMats);
				block.position.set(spawnX, spawnY, spawnZ);
				cameraOccluders.push(block);
				scene.add(block);

				const bodyDesc = RAPIER.RigidBodyDesc.dynamic().setTranslation(spawnX, spawnY, spawnZ);
				const bodyInstance = world.createRigidBody(bodyDesc);
				const colliderDesc = RAPIER.ColliderDesc.cuboid(0.5, 0.5, 0.5);
				colliderDesc.setFriction(0.9);
				colliderDesc.setRestitution(0.1);
				world.createCollider(colliderDesc, bodyInstance);

				fallingBlocks.push({ mesh: block, body: bodyInstance });
			};

			const forwardBase = new THREE.Vector3(0, 0, -1);
			const headOffset = new THREE.Vector3(0, 1.6, 0);
			const tempVec = new THREE.Vector3();
			const tempVec2 = new THREE.Vector3();
			const tempVec3 = new THREE.Vector3();
			const raycaster = new THREE.Raycaster();
			
			const clock = new THREE.Clock();
			let frame = 0;
			let spawnTimer = 0.6;
			const gravity = -18;
			let verticalVelocity = 0;
			let grounded = false;
			

			const tick = () => {
				const delta = Math.min(clock.getDelta(), 0.05);
				const time = clock.elapsedTime;

				const analogTurn = moveAxis.x;
				const analogMove = -moveAxis.y;
				const turnInput = (input.left ? -1 : 0) + (input.right ? 1 : 0) + analogTurn;

				let moveInput = analogMove * 0.9;
				if (input.forward) {
					moveInput = 1.2;
				} else if (input.back) {
					moveInput = -0.4;
				}

				moveInput = THREE.MathUtils.clamp(moveInput, -0.6, 1.2);
				player.rotation.y += turnInput * 1.6 * delta + pointerState.lookDelta;
				pointerState.lookDelta = 0;

				tempVec.copy(forwardBase).applyQuaternion(player.quaternion);
				player.position.addScaledVector(tempVec, 4.2 * moveInput * delta);

				const clampLimit = half - 2;
				player.position.x = THREE.MathUtils.clamp(player.position.x, -clampLimit, clampLimit);
				player.position.z = THREE.MathUtils.clamp(player.position.z, -clampLimit, clampLimit);

				const targetHeight = getHeightAt(player.position.x, player.position.z);
				if (player.position.y <= targetHeight + 0.02 && verticalVelocity <= 0) {
					grounded = true;
					player.position.y = targetHeight;
					verticalVelocity = 0;
				} else {
					grounded = false;
				}

				if (input.jump && grounded) {
					verticalVelocity = 7.2;
					grounded = false;
					input.jump = false;
				}

				verticalVelocity += gravity * delta;
				player.position.y += verticalVelocity * delta;
				if (player.position.y < targetHeight) {
					player.position.y = targetHeight;
					verticalVelocity = 0;
					grounded = true;
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
				cameraRig.updateMatrixWorld();

				const target = tempVec.copy(player.position).add(headOffset);
				const desiredWorld = tempVec2.copy(cameraOffset);
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

				spawnTimer -= delta;
				if (spawnTimer <= 0) {
					spawnTimer = THREE.MathUtils.randFloat(0.6, 1.4);
					spawnFallingBlock();
				}

				world.timestep = delta;
				world.step();

				for (let i = fallingBlocks.length - 1; i >= 0; i -= 1) {
					const block = fallingBlocks[i];
					const position = block.body.translation();
					block.mesh.position.set(position.x, position.y, position.z);

					if (position.y < -10) {
						scene.remove(block.mesh);
						const occluderIndex = cameraOccluders.indexOf(block.mesh);
						if (occluderIndex >= 0) {
							cameraOccluders.splice(occluderIndex, 1);
						}
						world.removeRigidBody(block.body);
						fallingBlocks.splice(i, 1);
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
				lookPadEl?.removeEventListener('pointerdown', handleLookDown);
				lookPadEl?.removeEventListener('pointermove', handleLookMove);
				lookPadEl?.removeEventListener('pointerup', handleLookUp);
				lookPadEl?.removeEventListener('pointercancel', handleLookUp);
				container?.removeChild(renderer.domElement);

				blockGeo.dispose();
				bodyMat.dispose();
				limbMat.dispose();
				grassTopMat.dispose();
				grassSideMat.dispose();
				dirtMat.dispose();
				stoneMat.dispose();
				grassTopTex.dispose();
				grassSideTex.dispose();
				dirtTex.dispose();
				stoneTex.dispose();
				(body.geometry as THREE.BufferGeometry).dispose();
				(head.geometry as THREE.BufferGeometry).dispose();
				(legLeft.geometry as THREE.BufferGeometry).dispose();
				(legRight.geometry as THREE.BufferGeometry).dispose();
				(armLeft.geometry as THREE.BufferGeometry).dispose();
				(armRight.geometry as THREE.BufferGeometry).dispose();

				for (const mesh of terrainMeshes) {
					scene.remove(mesh);
				}
				cameraOccluders.length = 0;

				for (const block of fallingBlocks) {
					scene.remove(block.mesh);
					world.removeRigidBody(block.body);
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
	<title>Runner Field</title>
	<link rel="preconnect" href="https://fonts.googleapis.com" />
	<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin />
	<link
		href="https://fonts.googleapis.com/css2?family=Space+Grotesk:wght@400;600&display=swap"
		rel="stylesheet"
	/>
</svelte:head>

<div class="page">
	<div class="hud">
		<h1>Runner Field</h1>
		<p>W / A / S / D or Arrow keys + mouse drag. Space to jump. Touch: left stick move, right pad look.</p>
	</div>
	<div class="scene" bind:this={container}></div>
	<div class="touch-controls">
		<div class="touch-pad joystick" bind:this={joystickEl}>
			<div class="thumb" bind:this={joystickThumbEl}></div>
		</div>
		<div class="touch-pad look" bind:this={lookPadEl}></div>
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
		width: 136px;
		height: 136px;
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
		bottom: calc(20px + env(safe-area-inset-bottom));
	}

	.touch-pad.look {
		right: calc(20px + env(safe-area-inset-right));
		bottom: calc(20px + env(safe-area-inset-bottom));
	}

	.touch-pad.look::after {
		content: 'Look';
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
