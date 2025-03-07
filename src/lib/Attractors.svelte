<script>
	import {
		BufferGeometry,
		Vector3,
		Vector4,
		PointsMaterial,
		BufferAttribute,
		BoxGeometry,
		Points,
		GridHelper,
		Vector2,
		Raycaster,
		PlaneGeometry,
		Mesh,
		SphereGeometry,
		DoubleSide
	} from 'three';

	import { T, useTask, useThrelte } from '@threlte/core';
	import { OrbitControls } from '@threlte/extras';
	import { onMount } from 'svelte';
	import { Spring } from 'svelte/motion';

	// Mouse raycasting...
	const { dom, camera, scene } = useThrelte();
	let mousePoint = new Vector2();
	let lastMousePoint = new Vector2();
	let mouseTravel = 0.0;

	let mousePosition = new Vector3(0.1, 0.1, 0.1);
	const planeGeo = new PlaneGeometry(360, 360);
	const mesh = new Mesh(planeGeo);

	const f = new SphereGeometry();
	const fMesh = new Mesh(f);

	onMount(() => {
		const raycaster = new Raycaster();

		dom.addEventListener('pointermove', (e) => {
			// console.log(e);
			mousePoint = new Vector2(
				(e.clientX / window.innerWidth) * 2.0 - 1,
				(e.clientY / window.innerHeight) * -2.0 + 1
			);

			// console.log(e);
			raycaster.setFromCamera(mousePoint, $camera);
			const intersects = raycaster.intersectObject(mesh);

			let p = intersects[0]?.point;
			if (p) {
				mousePosition = p;
			}
		});

		dom.addEventListener('pointerleave', (e) => {
			mousePoint = new Vector2();
		});
	});

	// this should work.
	const particleCount = 1200;

	let point = new Vector3(10, 10, 10);
	// These are the ideal points of the particles.
	const pointArray = new Float32Array(particleCount * 3);
	// These are the actual points of the particles.
	const particleArray = new Float32Array(particleCount * 3);

	const pointGeometry = new BufferGeometry();

	let tick = 0.0;
	const msClock = 0.4;
	const speed = 4;
	let dt = 0.03;

	let explosionPoint = new Vector3(0.2, 0.1, 0.8);
	const clamp = (num, min, max) => Math.min(Math.max(num, min), max);

	// This should only be calculated for the first point.
	useTask((delta) => {
		const intensity = 20.0;
		const maxSpeed = 4.0;
		let influence = 0.0;

		const eInfluence = 0.04;
		let speedFactor = 1.0;
		fMesh.position.set(mousePosition.x, mousePosition.y, mousePosition.z);
		mesh.rotation.set($camera.rotation.x, $camera.rotation.y, $camera.rotation.z);

		// console.log($camera);

		mouseTravel +=
			Math.max(
				Math.abs(mousePoint.x - lastMousePoint.x),
				Math.abs(mousePoint.y - lastMousePoint.y)
			) * 8.0;

		lastMousePoint = mousePoint;
		for (let i = 0; i < particleCount * 3; i++) {
			// explosives
			if (i % 3 === 0 && mousePosition) {
				// console.log(mousePosition);
				const x = particleArray[i];
				const y = particleArray[i + 1];
				const z = particleArray[i + 2];

				const distanceToExplostion = Math.sqrt(
					Math.pow(x - mousePosition.x, 2) +
						Math.pow(y - mousePosition.y, 2) +
						Math.pow(z - mousePosition.z, 2)
				);

				influence = (1 / distanceToExplostion) * intensity * mouseTravel;

				const eXDir = (x - mousePosition.x) / distanceToExplostion;
				const eYDir = (y - mousePosition.y) / distanceToExplostion;
				const eZDir = (z - mousePosition.z) / distanceToExplostion;

				particleArray[i] = x + eXDir * influence;
				particleArray[i + 1] = y + eYDir * influence;
				particleArray[i + 2] = z + eZDir * influence;

				speedFactor = Math.min(1.0, distanceToExplostion / eInfluence);
			}

			// clamp this value.
			const dir = clamp(pointArray[i] - particleArray[i], -maxSpeed, maxSpeed) * speedFactor;
			particleArray[i] = clamp(particleArray[i] + delta * dir * (speed * 2.2), -280, 280);
		}

		for (let i = 0; i < speed; i++) {
			const P = 10.0;
			const R = 28.0;
			const B = 8.0 / 3.0;

			// Now this should probably only happen every couple frames.
			// this is called for the particles to then, take the position of the first
			// particle and duplicate through the array.
			tick += delta;
			if (tick > msClock) {
				for (let i = particleCount * 3 - 1; i > 2; i--) {
					pointArray[i] = pointArray[i - 3];
				}
				tick = 0.0;
			}

			// when the end of the array is reached, the first entry should just copy from the last entry
			// this way the loop closes.
			pointArray[0] = point.x + dt * delta * (P * (point.y - point.x));
			pointArray[1] = point.y + dt * delta * (point.x * (R - point.z) - point.y);
			pointArray[2] = point.z + dt * delta * (point.x * point.y - B * point.z);

			// console.log(pointArray);
			point = new Vector3(pointArray[0], pointArray[1], pointArray[2]);
		}

		pointGeometry.setAttribute('position', new BufferAttribute(particleArray, 3));
		mouseTravel -= delta;

		mouseTravel = clamp(mouseTravel, -0.3, 1.5);
	});
</script>

<T.AmbientLight intensity={4.0} />

<T.PerspectiveCamera position={[0, 0, 120]} fov={75} makeDefault>
	<OrbitControls enableDamping enableZoom={false} />
</T.PerspectiveCamera>

<T is={mesh}>
	<T.MeshStandardMaterial color={'#001122'} visible={false} />
</T>

<T.Points geometry={pointGeometry} position={[0, 0, -20]} frustumCulled={false}>
	<T.PointsMaterial size={0.5} color={'white'} fog />
</T.Points>

<T is={fMesh} />

<!-- <T.GridHelper args={[10, 10]} /> -->
