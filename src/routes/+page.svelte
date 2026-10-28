<script>
  import { onMount } from 'svelte';

  let alpha = $state(null);
  let beta = $state(null);
  let gamma = $state(null);
  let message = $state('');

 function handleDeviceOrientation(event) {
  alpha = event.alpha === null ? null : Math.round(event.alpha);
  beta = event.beta === null ? null : Math.round(event.beta);
  gamma = event.gamma === null ? null : Math.round(event.gamma);
}

  async function startSensor() {
    if (!window.isSecureContext) {
      message = 'Please use HTTPS.';
      return;
    }

    if (!('DeviceOrientationEvent' in window)) {
      message = 'Device orientation is not supported.';
      return;
    }

    try {
      if (typeof DeviceOrientationEvent.requestPermission === 'function') {
        const permission = await DeviceOrientationEvent.requestPermission();

        if (permission !== 'granted') {
          message = 'Permission denied.';
          return;
        }
      }

      window.addEventListener('deviceorientation', handleDeviceOrientation);
      message = 'Sensor started. Move your phone!';
    } catch (error) {
      message = 'Could not access the sensor.';
    }
  }

  onMount(() => {
    return () => {
      window.removeEventListener('deviceorientation', handleDeviceOrientation);
    };
  });
</script>

<h1>Device Orientation</h1>

<button onclick={startSensor}>Start Sensor</button>

<p>{message}</p>

<p>Alpha: {alpha}</p>
<p>Beta: {beta}</p>
<p>Gamma: {gamma}</p>