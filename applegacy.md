<!DOCTYPE html>
<html>
<head>
    <title>Bluetooth RC Remote</title>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/crypto-js/4.2.0/crypto-js.min.js" integrity="sha512-a+SUDuwNzXDvz4XrIcXHuCf089/iJAoN4lmrXJg18XnduKK6YlDHNRalv4yd1N40OKI80tFidF+rqTFKGPoWFQ==" crossorigin="anonymous" referrerpolicy="no-referrer"></script>
</head>
<body>
    <button id="connect_device" onclick="connectButton()">Connect to device</button>
    <div>
        <input type="checkbox" id="turbo_mode" disabled> Turbo (RB)
    </div>
    <div>
        <input type="checkbox" id="light_mode" disabled> Lights (LB)
    </div>
    <div>
        Battery: <span id="battery_status">0</span>%
    </div>
    <div>
        <b>Controller Mapping:</b>
        <ul>
            <li>A / X: Accelerate</li>
            <li>X / Square: Brake</li>
            <li>D-pad Left/Right: Steer</li>
            <li>RB: Turbo</li>
            <li>LB: Lights</li>
        </ul>
    </div>
    <script>
        var bluetooth = null;

        const CONTROL_SERVICE_UUID = '0000fff0-0000-1000-8000-00805f9b34fb'
        const BATTERY_SERVICE_UUID = '0000fff0-0000-1000-8000-00805f9b34fb'
        const CONTROL_CHARACTERISTICS_UUID = 'd44bc439-abfd-45a2-b575-925416129600'
        const BATTERY_CHARACTERISTICS_UUID = 'd44bc439-abfd-45a2-b575-925416129601'

        const DECRYPT_KEY = "34522a5b7a6e492c08090a9d8d2a23f8";

        // Control states
        let bt_up = false;
        let bt_down = false;
        let bt_left = false;
        let bt_right = false;
        let turboPressed = false;
        let lightsPressed = false;

        const turboEl = document.getElementById('turbo_mode');
        const lightEl = document.getElementById('light_mode');
        const batteryEl = document.getElementById('battery_status');

        function connectButton() {
            requestDevice();
        }

        async function requestDevice() {
            console.log('Requesting any Bluetooth Device...');
            var device = await navigator.bluetooth.requestDevice({
                filters: [
                    { namePrefix: "QCAR-" },
                ],
                optionalServices: [
                    CONTROL_SERVICE_UUID,
                    BATTERY_SERVICE_UUID,
                    CONTROL_CHARACTERISTICS_UUID,
                    BATTERY_CHARACTERISTICS_UUID,
                ]
            });
            await connectDevice(device);
            console.log("Device connected");
        }

        async function onDisconnected() {
            console.log('> Bluetooth Device disconnected');
            bluetooth = null;
        }

        function decryptAES(cipherText) {
            const encryptedHex = CryptoJS.enc.Hex.parse(cipherText.tohex());
            const keyHex = CryptoJS.enc.Hex.parse(DECRYPT_KEY);

            const decrypted = CryptoJS.AES.decrypt({ ciphertext: encryptedHex }, keyHex, {
                mode: CryptoJS.mode.ECB,
                padding: CryptoJS.pad.NoPadding
            });

            return decrypted.toString().toUint8Array();
        }

        function encryptAES(plainBytes) {
            const valueHex = CryptoJS.enc.Hex.parse(plainBytes.tohex());
            const keyHex = CryptoJS.enc.Hex.parse(DECRYPT_KEY);
            const encrypted = CryptoJS.AES.encrypt(valueHex, keyHex, {
                mode: CryptoJS.mode.ECB,
                padding: CryptoJS.pad.NoPadding
            });
            return encrypted.ciphertext.toString().toUint8Array();
        }

        async function sendMessage() {
            const cmd = calculateMove();
            const encryptCmd = encryptAES(cmd);
            var service = await bluetooth.getPrimaryService(CONTROL_SERVICE_UUID);
            var characteristic = await service.getCharacteristic(CONTROL_CHARACTERISTICS_UUID);
            await characteristic.writeValue(encryptCmd);
        }

        async function connectDevice(device) {
            device.addEventListener('gattserverdisconnected', onDisconnected);
            bluetooth = await device.gatt.connect();
            return new Promise(async (resolve) => {
                const BatteryService = await bluetooth.getPrimaryService(BATTERY_SERVICE_UUID);
                const BatteryCharacteristic = await BatteryService.getCharacteristic(BATTERY_CHARACTERISTICS_UUID);
                await BatteryCharacteristic.startNotifications();
                BatteryCharacteristic.addEventListener('characteristicvaluechanged', handleNotificationsBattery);
            });
        }

        ArrayBuffer.prototype.tohex = function () {
            return [...new Uint8Array(this)]
                .map(x => x.toString(16).padStart(2, '0'))
                .join('');
        }

        Uint8Array.prototype.tohex = function () {
            return [...new Uint8Array(this)]
                .map(x => x.toString(16).padStart(2, '0'))
                .join('');
        }

        String.prototype.toUint8Array = function () {
            return Uint8Array.from(this.match(/.{1,2}/g).map((byte) => parseInt(byte, 16)));
        }

        function calculateMove() {
            const turbo = turboPressed;
            const light = lightsPressed;

            const up = bt_up;
            const down = bt_down;
            const left = bt_left;
            const right = bt_right;

            const cmd = new Uint8Array(16);

            cmd[1] = 0x43; // 'C'
            cmd[2] = 0x54; // 'T'
            cmd[3] = 0x4c; // 'L'
            cmd[8] = 1;    // Lights default OFF (1)
            cmd[9] = 0x3C; // Normal speed

            if (up) cmd[4] = 1;
            if (down) cmd[5] = 1;
            if (left) cmd[6] = 1;
            if (right) cmd[7] = 1;

            // Lights byte: 0 = ON, 1 = OFF
            cmd[8] = light ? 0 : 1;

            if (turbo) cmd[9] = 0x64; // Turbo speed if pressed

            // Update UI checkboxes
            turboEl.checked = turbo;
            lightEl.checked = light;

            return cmd;
        }

        function handleNotificationsBattery(event) {
            let value = event.target.value;
            let decrypt = decryptAES(value.buffer)
            batteryEl.innerHTML = decrypt[4];
        }

        // Gamepad polling
        function pollGamepad() {
            const gamepads = navigator.getGamepads ? navigator.getGamepads() : [];

            if (gamepads.length > 0) {
                const gp = gamepads[0]; // Use first connected gamepad

                if (gp) {
                    // Buttons: pressed = true/false
                    bt_up = gp.buttons[0]?.pressed || false;      // A / X - acceleration
                    bt_down = gp.buttons[2]?.pressed || false;    // X / Square - braking

                    // D-pad left/right
                    const dpadLeft = gp.buttons[14]?.pressed || false;
                    const dpadRight = gp.buttons[15]?.pressed || false;

                    // Left stick horizontal axis (-1 left, +1 right)
                    const axisX = gp.axes[0] || 0;
                    const deadzone = 0.3;

                    // Determine left/right from stick with deadzone
                    const stickLeft = axisX < -deadzone;
                    const stickRight = axisX > deadzone;

                    // Combine D-pad and stick inputs
                    bt_left = dpadLeft || stickLeft;
                    bt_right = dpadRight || stickRight;

                    turboPressed = gp.buttons[5]?.pressed || false;  // Right bumper
                    lightsPressed = gp.buttons[4]?.pressed || false; // Left bumper
                }
            }

            requestAnimationFrame(pollGamepad);
        }


        window.addEventListener('gamepadconnected', (e) => {
            console.log('Gamepad connected:', e.gamepad);
            pollGamepad();
        });

        window.addEventListener('gamepaddisconnected', (e) => {
            console.log('Gamepad disconnected:', e.gamepad);
            // Reset controls on disconnect
            bt_up = bt_down = bt_left = bt_right = false;
            turboPressed = lightsPressed = false;
            turboEl.checked = false;
            lightEl.checked = false;
        });

        // Start polling immediately in case gamepad is already connected
        pollGamepad();

        // Send BLE command every 100ms
        setInterval(async function () {
            if (!bluetooth) return;
            await sendMessage();
        }, 100);

    </script>
</body>
</html>

