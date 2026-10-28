<script>
  import { onMount } from 'svelte';

  let alpha = $state(null);
  let beta = $state(null);
  let gamma = $state(null);

  let number = $state('-');
  let message = $state('Click Start Sensor');
  let shaking = false;
  let lastPosition = null;
  let positionTime = 0;
  let lastRoll = 0;

  function handleOrientation(event) {
    alpha = event.alpha === null ? null : Math.round(event.alpha);
    beta = event.beta === null ? null : Math.round(event.beta);
    gamma = event.gamma === null ? null : Math.round(event.gamma);
  }

  function handleMotion(event) {
    const g = event.accelerationIncludingGravity;
    if (g?.x == null || g?.y == null || g?.z == null) return;

    const { x, y, z } = g;
    const a = event.acceleration;

    // Detect shaking
    const force = a?.x != null && a?.y != null && a?.z != null
      ? Math.hypot(a.x, a.y, a.z)
      : Math.abs(Math.hypot(x, y, z) - 9.8);

    if (!shaking && Date.now() - lastRoll > 1000 && force > 13) {
      shaking = true;
      number = '-';
      lastPosition = null;
      message = 'Shake detected! Hold your phone in a position.';
    }

    if (!shaking) return;

    // Detect the six sides
    let position = null;

    if (z > 7.5 && Math.abs(x) < 5 && Math.abs(y) < 5) position = 6;
    else if (z < -7.5 && Math.abs(x) < 5 && Math.abs(y) < 5) position = 1;
    else if (x < -7.5 && Math.abs(y) < 5 && Math.abs(z) < 5) position = 2;
    else if (x > 7.5 && Math.abs(y) < 5 && Math.abs(z) < 5) position = 5;
    else if (y > 7.5 && Math.abs(x) < 5 && Math.abs(z) < 5) position = 3;
    else if (y < -7.5 && Math.abs(x) < 5 && Math.abs(z) < 5) position = 4;

    // Wait until the position is stable
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

  async function startSensor() {
    if (!window.isSecureContext) {
      message = 'HTTPS is required.';
      return;
    }

    if (!('DeviceMotionEvent' in window)) {
      message = 'Motion sensors are not supported.';
      return;
    }

    try {
      if (typeof DeviceMotionEvent.requestPermission === 'function') {
        if (await DeviceMotionEvent.requestPermission() !== 'granted') {
          message = 'Permission denied.';
          return;
        }
      }

      if (typeof DeviceOrientationEvent !== 'undefined' &&
          typeof DeviceOrientationEvent.requestPermission === 'function') {
        if (await DeviceOrientationEvent.requestPermission() !== 'granted') {
          message = 'Permission denied.';
          return;
        }
      }

      window.removeEventListener('devicemotion', handleMotion);
      window.removeEventListener('deviceorientation', handleOrientation);

      window.addEventListener('devicemotion', handleMotion);
      window.addEventListener('deviceorientation', handleOrientation);

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
  </div>

</div>