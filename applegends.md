<!DOCTYPE html>
<html>
<head>
  <title>Shell Racing Legends – Bluetooth RC Remote</title>
  <meta charset="utf-8">
  <style>
    body { font-family:sans-serif; }
    label { display:block; margin:6px 0; }
  </style>
</head>

<body>
  <button id="connectBtn">Connect car</button>

  <label><input type="checkbox" id="turboBox" disabled> Turbo</label>
  <label><input type="checkbox" id="lightBox" disabled> Lights</label>
  <label><input type="checkbox" id="donutBox" disabled> Donut Spin</label>
  <div>Battery %: <span id="battSpan">0</span></div>

  <script>
  /* ----------  CONSTANTS  ---------- */
  const CONTROL_SERVICE_UUID = '0000fff0-0000-1000-8000-00805f9b34fb';
  const CONTROL_CHAR_UUID    = '0000fff1-0000-1000-8000-00805f9b34fb';
  const BATTERY_SERVICE_UUID = 'battery_service';  // Standard 0x180F
  const BATTERY_CHAR_UUID    = 'battery_level';    // Standard 0x2A19

  /* ----------  STATE  ---------- */
  let bleDevice  = null;
  let ctrlChar   = null;
  let lastPacket = new Uint8Array(8);  // Track changes for event-driven sends
  let up=false, down=false, left=false, right=false, turbo=false, lights=false, donut=false, mode=0;

  /* ----------  CONNECT  ---------- */
  async function connect() {
    bleDevice = await navigator.bluetooth.requestDevice({
      filters: [{namePrefix: 'SL-'}],
      optionalServices: [CONTROL_SERVICE_UUID, BATTERY_SERVICE_UUID]
    });
    bleDevice.addEventListener('gattserverdisconnected', () => bleDevice = null);
    const gatt = await bleDevice.gatt.connect();

    const ctrlSvc = await gatt.getPrimaryService(CONTROL_SERVICE_UUID);
    ctrlChar      = await ctrlSvc.getCharacteristic(CONTROL_CHAR_UUID);

    const battSvc = await gatt.getPrimaryService(BATTERY_SERVICE_UUID);
    const battChar = await battSvc.getCharacteristic(BATTERY_CHAR_UUID);
    await battChar.startNotifications();
    battChar.addEventListener('characteristicvaluechanged', handleBattery);
    // Initial read for current battery level
    const initialBatt = await battChar.readValue();
    handleBattery({target: {value: initialBatt}});

    // Send initial zero packet
    await sendCmd();
  }

  /* ----------  BATTERY  ---------- */
  function handleBattery(ev) {
    const value = ev.target.value.getUint8(0);  // Single byte: 0-100%
    document.getElementById('battSpan').textContent = value;
  }

  /* ----------  COMMAND BUILD  ---------- */
  function buildPacket() {
    const packet = new Uint8Array(8);
    packet[0] = mode;    // 0: Normal mode (toggle for UI modes if needed)
    packet[1] = up ? 1 : 0;
    packet[2] = down ? 1 : 0;
    packet[3] = left ? 1 : 0;
    packet[4] = right ? 1 : 0;
    packet[5] = lights ? 1 : 0;
    packet[6] = turbo ? 1 : 0;
    packet[7] = donut ? 1 : 0;
    return packet;
  }

  /* ----------  SEND  ---------- */
  async function sendCmd(force = false) {
    if (!bleDevice || !ctrlChar) return;
    const packet = buildPacket();
    const hasMotion = packet[1] || packet[2];  // up or down active
    const isBoostPress = packet[6] && !lastPacket[6];  // turbo just pressed
    const isSpecialPress = packet[5] || packet[7];  // lights or donut

    // Send if changed, or force for turbo/special (even if no motion change)
    if (force || !arraysEqual(packet, lastPacket) || (isBoostPress && hasMotion) || isSpecialPress) {
      await ctrlChar.writeValueWithoutResponse(packet);
      console.log('Sent packet:', Array.from(packet).map(b => b.toString(16).padStart(2, '0')).join(' '), 
                  `| Turbo: ${packet[6] ? 'ON' : 'OFF'}`);
      lastPacket = new Uint8Array(packet);  // Copy for comparison
    }
  }

  function arraysEqual(a, b) {
    if (a.length !== b.length) return false;
    for (let i = 0; i < a.length; i++) if (a[i] !== b[i]) return false;
    return true;
  }

  /* ----------  GAMEPAD  ---------- */
  function pollPad() {
    const gp = navigator.getGamepads()[0];
    if (gp) {
      const prevUp = up;
      const prevTurbo = turbo;
      up     = gp.buttons[0]?.pressed || false;  // Accelerate (A/X)
      down   = gp.buttons[2]?.pressed || false;  // Brake (X/Square)
      const dL = gp.buttons[14]?.pressed || false,
            dR = gp.buttons[15]?.pressed || false,
            ax  = gp.axes[0] || 0;
      left   = dL || ax < -0.3;  // Steer left (D-pad + stick)
      right  = dR || ax >  0.3;  // Steer right
      turbo  = gp.buttons[5]?.pressed || false;  // Right bumper (RB)
      lights = gp.buttons[4]?.pressed || false;  // Left bumper (LB)
      donut  = gp.buttons[3]?.pressed || false;  // Y/Triangle for donut spin
      mode   = gp.buttons[7]?.pressed ? 1 : 0;   // Start button for mode toggle (hold to switch)

      // Update UI
      document.getElementById('turboBox').checked = turbo;
      document.getElementById('lightBox').checked = lights;
      document.getElementById('donutBox').checked = donut;

      // Send on change (event-driven); force if turbo pressed with motion
      const turboWithMotion = turbo && (up || down);
      sendCmd(turboWithMotion || turbo !== prevTurbo || up !== prevUp);
    }
    requestAnimationFrame(pollPad);
  }
  pollPad();

  /* ----------  UI  ---------- */
  document.getElementById('connectBtn').onclick = connect;
  </script>
</body>
</html>
