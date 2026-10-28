<script>
  import { onMount } from 'svelte';

  let alpha = $state(null);
  let beta = $state(null);
  let gamma = $state(null);

  let number = $state('-');
  let message = $state('Press Start Sensor');

  let x = $state(0);
  let y = $state(0);
  let z = $state(0);

  let shaking = false;
  let lastPosition = null;
  let positionTime = 0;
  let lastRoll = 0;
  let lastForce = 0;

  function handleOrientation(event) {
    alpha = event.alpha == null ? null : Math.round(event.alpha);
    beta = event.beta == null ? null : Math.round(event.beta);
    gamma = event.gamma == null ? null : Math.round(event.gamma);
  }

  function handleMotion(event) {
    const g = event.accelerationIncludingGravity;
    if (g?.x == null || g?.y == null || g?.z == null) return;

    x = Math.round(g.x);
    y = Math.round(g.y);
    z = Math.round(g.z);

    // Detect shaking
    const a = event.acceleration;

    const force = a?.x != null && a?.y != null && a?.z != null
      ? Math.hypot(a.x, a.y, a.z)
      : Math.hypot(g.x - xPrevious, g.y - yPrevious, g.z - zPrevious);

    xPrevious = g.x;
    yPrevious = g.y;
    zPrevious = g.z;

    if (force > 6 && Date.now() - lastRoll > 1000) {
      shaking = true;
      number = '-';
      lastPosition = null;
      message = 'Shake detected! Hold your phone still.';
    }

    if (!shaking) return;

    // Detect the six sides
    let position = null;

    if (g.z > 7) position = 6;
    else if (g.z < -7) position = 1;
    else if (g.x < -7) position = 2;
    else if (g.x > 7) position = 5;
    else if (g.y > 7) position = 3;
    else if (g.y < -7) position = 4;

    // Check if the phone stays in one position
    if (position !== lastPosition) {
      lastPosition = position;
      positionTime = Date.now();
    }

    if (position !== null && Date.now() - positionTime > 700 && force < 3) {
      number = position;
      shaking = false;
      lastRoll = Date.now();
      message = 'Shake again to roll!';
    }
  }

  let xPrevious = 0;
  let yPrevious = 0;
  let zPrevious = 0;

  async function startSensor() {
    if (!window.isSecureContext) {
      message = 'HTTPS is required.';
      return;
    }

    if (!('DeviceMotionEvent' in window)) {
      message = 'Motion sensors not supported.';
      return;
    }

    try {
      // Request both permissions directly from the button click
      const motion = typeof DeviceMotionEvent.requestPermission === 'function'
        ? DeviceMotionEvent.requestPermission()
        : Promise.resolve('granted');

      const orientation = typeof DeviceOrientationEvent !== 'undefined' &&
        typeof DeviceOrientationEvent.requestPermission === 'function'
        ? DeviceOrientationEvent.requestPermission()
        : Promise.resolve('granted');

      const [motionResult, orientationResult] = await Promise.all([motion, orientation]);

      if (motionResult !== 'granted') {
        message = 'Motion permission denied.';
        return;
      }

      window.removeEventListener('devicemotion', handleMotion);
      window.removeEventListener('deviceorientation', handleOrientation);

      window.addEventListener('devicemotion', handleMotion);
      if (orientationResult === 'granted') {
        window.addEventListener('deviceorientation', handleOrientation);
      }

      message = 'Ready! Shake your phone.';
    } catch {
      message = 'Could not access sensors.';
    }
  }

  onMount(() => {
    return () => {
      window.removeEventListener('devicemotion', handleMotion);
      window.removeEventListener('deviceorientation', handleOrientation);
    };
  });
</script>

<div class="flex min-h-screen flex-col items-center justify-center gap-6 bg-gray-100 p-6">

  <h1 class="text-3xl font-bold">Digital Dice 🎲</h1>

  <div class="flex h-40 w-40 items-center justify-center rounded-3xl bg-white text-8xl font-bold shadow-lg">
    {number}
  </div>

  <p class="text-center">{message}</p>

  <button
    onclick={startSensor}
    class="rounded-xl bg-blue-600 px-6 py-3 text-white"
  >
    Start Sensor
  </button>

  <div class="text-center text-gray-500">
    <p>Alpha: {alpha ?? '-'}</p>
    <p>Beta: {beta ?? '-'}</p>
    <p>Gamma: {gamma ?? '-'}</p>

    <p class="mt-4">X: {x}</p>
    <p>Y: {y}</p>
    <p>Z: {z}</p>
  </div>

</div>