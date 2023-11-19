<script lang="ts">
	import { onMount } from 'svelte';

	let canvas: HTMLCanvasElement;
	let ctx: CanvasRenderingContext2D;

	interface Ship {
		x: number;
		y: number;
		angle: number;
		thrust: number;
	}

	interface KeyState {
		left: boolean;
		right: boolean;
		up: boolean;
		down: boolean;
		shoot: boolean;
	}

	interface Asteroid {
		x: number;
		y: number;
		size: number;
		xVel: number;
		yVel: number;
		color: string;
	}

	interface Bullet {
		x: number;
		y: number;
		angle: number;
		speed: number;
	}

	let ship: Ship = { x: 400, y: 300, angle: 0, thrust: 0 };
	let keys: KeyState = { left: false, right: false, up: false, down: false, shoot: false };
	let asteroids: Asteroid[] = [];
	let bullets: Bullet[] = [];
	let isExploding: boolean = false;
	let respawnTime: number = 3000; // 5 seconds
	let lastRespawn: number = Date.now();
	let gameTimer: number = 0;
	let explosionSize: number = 0;
	let playerHasMoved = false;

	let lastShotTime: number = 0;
	const shootingDelay: number = 500; // Delay in milliseconds

	// let explosionSound = new Audio('hit.wav');

	// timer display
	let bestTimeY = 30; // Y-coordinate for the best time
	let currentTimeY = 60; // Y-coordinate for the current time

	const asteroidCount = 5;
	const bulletSpeed = 5;

	function createAsteroid(): Asteroid {
		let newX, newY, distance;
		const safeDistance = 100; // Safe distance from the player

		do {
			newX = Math.random() * canvas.width;
			newY = Math.random() * canvas.height;

			let dx = newX - ship.x;
			let dy = newY - ship.y;
			distance = Math.sqrt(dx * dx + dy * dy);
		} while (distance < safeDistance);

		return {
			x: newX,
			y: newY,
			size: 50 + Math.random() * 30,
			xVel: (Math.random() - 0.5) * 3,
			yVel: (Math.random() - 0.5) * 3,
			color: getRandomGray()
		};
	}

	function handleKeyDown(e: KeyboardEvent): void {
		if (e.key === 'ArrowLeft') keys.left = true;
		if (e.key === 'ArrowRight') keys.right = true;
		if (e.key === 'ArrowUp') keys.up = true;
		if (e.key === 'ArrowDown') keys.down = true;
		if (e.key === 'a') keys.left = true;
		if (e.key === 'd') keys.right = true;
		if (e.key === 'w') keys.up = true;
		if (e.key === 's') keys.down = true;
		if (e.key === ' ') keys.shoot = true;
	}

	function handleKeyUp(e: KeyboardEvent): void {
		if (e.key === 'ArrowLeft') keys.left = false;
		if (e.key === 'ArrowRight') keys.right = false;
		if (e.key === 'ArrowUp') keys.up = false;
		if (e.key === 'ArrowDown') keys.down = false;
		if (e.key === 'a') keys.left = false;
		if (e.key === 'd') keys.right = false;
		if (e.key === 'w') keys.up = false;
		if (e.key === 's') keys.down = false;
		if (e.key === ' ') keys.shoot = false;
	}

	onMount(() => {
		const resizeCanvas = () => {
			canvas.width = window.innerWidth;
			canvas.height = window.innerHeight;

			// You might need to redraw or reposition game elements here
		};
		spawnAsteroid();

		window.addEventListener('resize', resizeCanvas);

		// Initial setup
		resizeCanvas();

		ctx = canvas.getContext('2d')!;
		canvas.width = window.innerWidth;
		canvas.height = window.innerHeight;
		for (let i = 0; i < asteroidCount; i++) {
			asteroids.push(createAsteroid());
		}

		window.addEventListener('keydown', handleKeyDown);
		window.addEventListener('keyup', handleKeyUp);

		requestAnimationFrame(gameLoop);

		return () => {
			window.removeEventListener('resize', resizeCanvas);
		};
	});

	function updateShip(): void {
		if (keys.left) ship.angle -= 0.05;
		if (keys.right) ship.angle += 0.05;

		if (keys.left || keys.right || keys.up || keys.down) {
			playerHasMoved = true;
		}

		if (keys.up) {
			ship.thrust = 2;
		} else {
			ship.thrust = 0;
		}
		ship.x += Math.cos(ship.angle) * ship.thrust;
		ship.y += Math.sin(ship.angle) * ship.thrust;

		// Wrap around horizontally
		if (ship.x < 0) ship.x = canvas.width;
		else if (ship.x > canvas.width) ship.x = 0;

		// Wrap around vertically
		if (ship.y < 0) ship.y = canvas.height;
		else if (ship.y > canvas.height) ship.y = 0;
	}

	function drawShip(): void {
		ctx.save();
		ctx.translate(ship.x, ship.y);
		ctx.rotate(ship.angle);
		ctx.beginPath();
		ctx.moveTo(-10, -10);
		ctx.lineTo(10, 0);
		ctx.lineTo(-10, 10);
		ctx.closePath();
		ctx.stroke();
		ctx.restore();
	}

	function spawnAsteroid() {
		if (playerHasMoved) {
			asteroids.push(createAsteroid());
		}
		const nextSpawnTime = Math.random() * (5000 - 2000) + 2000; // Time in milliseconds
		setTimeout(spawnAsteroid, nextSpawnTime);
	}

	function getRandomGray(): string {
		let grayValue = Math.floor(Math.random() * 256);
		return `rgb(${grayValue}, ${grayValue}, ${grayValue})`;
	}

	function updateAsteroids(): void {
		if (!playerHasMoved) return;

		for (let asteroid of asteroids) {
			asteroid.x += asteroid.xVel;
			asteroid.y += asteroid.yVel;

			// Wrap asteroids around the canvas
			if (asteroid.x < 0) asteroid.x = canvas.width;
			if (asteroid.x > canvas.width) asteroid.x = 0;
			if (asteroid.y < 0) asteroid.y = canvas.height;
			if (asteroid.y > canvas.height) asteroid.y = 0;
		}
	}

	function drawAsteroids(): void {
		for (let asteroid of asteroids) {
			ctx.fillStyle = asteroid.color;
			ctx.shadowBlur = 10;
			ctx.shadowColor = 'rgba(0, 0, 0, 0.5)';
			ctx.fillRect(asteroid.x, asteroid.y, asteroid.size, asteroid.size);
			ctx.shadowBlur = 0;
		}
	}

	function shootBullet(): void {
		let currentTime = Date.now();
		if (keys.shoot && currentTime - lastShotTime > shootingDelay && !isExploding) {
			bullets.push({
				x: ship.x,
				y: ship.y,
				angle: ship.angle,
				speed: bulletSpeed
			});
			lastShotTime = currentTime;
		}
	}

	function updateBullets(): void {
		for (let i = bullets.length - 1; i >= 0; i--) {
			let bullet = bullets[i];
			bullet.x += Math.cos(bullet.angle) * bullet.speed;
			bullet.y += Math.sin(bullet.angle) * bullet.speed;

			// Remove bullets that go off screen
			if (bullet.x < 0 || bullet.x > canvas.width || bullet.y < 0 || bullet.y > canvas.height) {
				bullets.splice(i, 1);
			}
		}
	}

	function drawBullets(): void {
		ctx.fillStyle = 'black';
		for (let bullet of bullets) {
			ctx.beginPath();
			ctx.arc(bullet.x, bullet.y, 2, 0, Math.PI * 2);
			ctx.fill();
		}
	}

	function spawnNewAsteroids(count: number, delay: number): void {
		setTimeout(() => {
			for (let i = 0; i < count; i++) {
				asteroids.push(createAsteroid());
			}
		}, delay);
	}

	function checkCollisions(): void {
		for (let i = asteroids.length - 1; i >= 0; i--) {
			let asteroid = asteroids[i];
			for (let j = bullets.length - 1; j >= 0; j--) {
				let bullet = bullets[j];
				if (
					bullet.x >= asteroid.x &&
					bullet.x <= asteroid.x + asteroid.size &&
					bullet.y >= asteroid.y &&
					bullet.y <= asteroid.y + asteroid.size
				) {
					// Collision detected
					asteroids.splice(i, 1);
					bullets.splice(j, 1);
					// explosionSound.play();
					spawnNewAsteroids(2, 500); // Spawn 2 new asteroids after 500 ms
					break;
				}
			}
		}
	}

	function getShipVertices(ship: Ship): { x: number; y: number }[] {
		let points = [];
		let shipSize = 10; // Adjust as per the ship's size

		// Front vertex
		points.push({
			x: ship.x + Math.cos(ship.angle) * shipSize,
			y: ship.y + Math.sin(ship.angle) * shipSize
		});

		// Rear vertices
		let angle45 = Math.PI / 4;
		points.push({
			x: ship.x + Math.cos(ship.angle + Math.PI + angle45) * shipSize,
			y: ship.y + Math.sin(ship.angle + Math.PI + angle45) * shipSize
		});
		points.push({
			x: ship.x + Math.cos(ship.angle + Math.PI - angle45) * shipSize,
			y: ship.y + Math.sin(ship.angle + Math.PI - angle45) * shipSize
		});

		return points;
	}

	function checkShipCollision(): void {
		let shipVertices = getShipVertices(ship);

		for (let asteroid of asteroids) {
			for (let vertex of shipVertices) {
				if (
					vertex.x >= asteroid.x &&
					vertex.x <= asteroid.x + asteroid.size &&
					vertex.y >= asteroid.y &&
					vertex.y <= asteroid.y + asteroid.size &&
					!isExploding
				) {
					isExploding = true;
					explosionSize = 20;
					setTimeout(() => {
						respawnShip();
					}, respawnTime);
					return;
				}
			}
		}
	}

	function respawnShip(): void {
		let currentGameTime = Date.now() - lastRespawn;

		if (currentGameTime > getBestTime()) {
			localStorage.setItem('bestTime', currentGameTime.toString());
		}

		ship = { x: 400, y: 300, angle: 0, thrust: 0 };
		isExploding = false;
		lastRespawn = Date.now();
		gameTimer = 0;
		playerHasMoved = false;

		asteroids = [];
		for (let i = 0; i < asteroidCount; i++) {
			asteroids.push(createAsteroid());
		}
	}

	function drawExplosion(): void {
		ctx.save();
		ctx.translate(ship.x, ship.y);

		// Color change from red to orange
		let color = explosionSize < 35 ? 'red' : 'orange';
		ctx.fillStyle = color;

		ctx.beginPath();
		ctx.arc(0, 0, explosionSize, 0, Math.PI * 2);
		ctx.fill();
		ctx.restore();

		// Increase the size of the explosion until a certain limit
		if (explosionSize < 50) {
			explosionSize += 1;
		}
	}

	function displayAlignedText(text: string, x: number, y: number): void {
		let colonIndex = text.indexOf(':');
		let beforeColon = text.substring(0, colonIndex + 1);
		let afterColon = text.substring(colonIndex + 1);

		let beforeColonWidth = ctx.measureText(beforeColon).width;

		// Display the first part of the text
		ctx.fillText(beforeColon, x - beforeColonWidth, y);

		// Display the second part of the text
		ctx.fillText(afterColon, x, y);
	}

	function formatTime(ms: number): string {
		let minutes = Math.floor(ms / 60000);
		let seconds = Math.floor((ms % 60000) / 1000);
		let milliseconds = ms % 1000;

		return `${minutes}:${seconds.toString().padStart(2, '0')}.${milliseconds
			.toString()
			.padStart(3, '0')}`;
	}

	function getBestTime(): number {
		return parseInt(localStorage.getItem('bestTime') || '0');
	}

	function gameLoop(): void {
		let currentTime = Date.now();

		ctx.clearRect(0, 0, canvas.width, canvas.height);

		if (playerHasMoved && !isExploding) {
			gameTimer = currentTime - lastRespawn;
		}

		ctx.font = '26px Arial';
		let rightAlignX = canvas.width - 120; // X-coordinate for alignment

		if (!isExploding) {
			updateShip();
			checkShipCollision();
			gameTimer = currentTime - lastRespawn;
		} else {
			drawExplosion();
		}

		shootBullet();
		updateBullets();
		updateAsteroids();
		checkCollisions();

		if (!isExploding) {
			drawShip();
		}
		drawBullets();
		drawAsteroids();

		ctx.fillStyle = 'black';
		let bestTimeText = `Najbolje vrijeme: ${formatTime(getBestTime())}`;
		displayAlignedText(bestTimeText, rightAlignX, bestTimeY);

		//timer
		if (playerHasMoved) {
			let currentTimeText = `Vrijeme: ${formatTime(gameTimer)}`;
			displayAlignedText(currentTimeText, rightAlignX, currentTimeY);
		} else {
			let currentTimeText = `Vrijeme: 0:00.000`;
			displayAlignedText(currentTimeText, rightAlignX, currentTimeY);
		}

		requestAnimationFrame(gameLoop);
	}
</script>

<canvas bind:this={canvas}></canvas>

<style>
	body,
	html {
		margin: 0;
		padding: 0;
		overflow: hidden; /* Prevent scrollbars */
	}
	canvas {
		display: block; /* Remove default margin */
		width: 100%;
		height: 100%;
	}
</style>
