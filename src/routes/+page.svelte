<script>
  import { onMount } from 'svelte';

  let alpha = $state(null);
  let beta = $state(null);
  let gamma = $state(null);

  let x = $state(null);
  let y = $state(null);
  let z = $state(null);

  let number = $state('-');
  let message = $state('Click Start Sensor');

  let shaking = false;
  let lastPosition = null;
  let positionTime = 0;
  let lastRoll = 0;
  let previous = null;

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

    if (message === 'Waiting for sensor data...') {
      message = 'Ready! Shake your phone.';
    }

    const force = previous
      ? Math.hypot(g.x - previous.x, g.y - previous.y, g.z - previous.z)
      : 0;

    previous = { x: g.x, y: g.y, z: g.z };

    if (force > 5 && Date.now() - lastRoll > 1000) {
      shaking = true;
      number = '-';
      lastPosition = null;
      message = 'Shake detected! Hold your phone still.';
    }

    if (!shaking) return;

    let position = null;

    if (g.z > 7) position = 6;
    else if (g.z < -7) position = 1;
    else if (g.x < -7) position = 2;
    else if (g.x > 7) position = 5;
    else if (g.y > 7) position = 3;
    else if (g.y < -7) position = 4;

    if (position !== lastPosition) {
      lastPosition = position;
      positionTime = Date.now();
    }

    if (position !== null && Date.now() - positionTime > 700 && force < 3) {
      number = position;
      shaking = false;
      lastRoll = Date.now();
      message = 'Shake again!';
    }
  }

  async function startSensor() {
    message = 'Starting sensors...';

    if (!window.isSecureContext) {
      message = 'HTTPS required!';
      return;
    }

    if (!window.DeviceMotionEvent) {
      message = 'Motion sensors not supported!';
      return;
    }

    try {
      // Ask permission on iPhone
      if (typeof window.DeviceMotionEvent.requestPermission === 'function') {
        const result = await window.DeviceMotionEvent.requestPermission();

        if (result !== 'granted') {
          message = 'Permission denied!';
          return;
        }
      }

      window.removeEventListener('devicemotion', handleMotion);
      window.addEventListener('devicemotion', handleMotion);

      window.removeEventListener('deviceorientation', handleOrientation);
      window.addEventListener('deviceorientation', handleOrientation);

      message = 'Waiting for sensor data...';
    } catch (error) {
      message = 'Error: ' + error.message;
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

  <p class="text-center font-semibold">{message}</p>

  <button
    onclick={startSensor}
    class="rounded-xl bg-blue-600 px-6 py-3 text-white"
  >
    Start Sensor
  </button>

  <div class="text-center text-gray-600">
    <p>Alpha: {alpha ?? '-'}</p>
    <p>Beta: {beta ?? '-'}</p>
    <p>Gamma: {gamma ?? '-'}</p>

    <div class="mt-4">
      <p>X: {x ?? '-'}</p>
      <p>Y: {y ?? '-'}</p>
      <p>Z: {z ?? '-'}</p>
    </div>
  </div>

</div>