# folio-2019 Multiplayer — Implementation Plan

Based on: https://github.com/brunosimon/folio-2019

---

## Verdict: FULLY VIABLE

The repo's architecture is well-suited for multiplayer. The physics→rendering pipeline is already decoupled, controls use an action-state boolean pattern (network-friendly), and Three.js can render N instances of the same model with minimal changes.

---

## Current Architecture (Single Player)

```
User Input (WASD / Arrow keys / Touch)
  → Controls.actions { up, down, left, right, brake, boost }
  → Physics.js — Cannon.js world.step() each frame
  → Car chassis + 4 wheel bodies (Cannon RigidBodies)
  → Three.js meshes — mirror physics body positions
  → Camera — follows chassis
```

**Key insight:** meshes only READ physics body positions. They don't own state. This is exactly the pattern needed for multiplayer remote cars.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend rendering | Three.js (WebGL) |
| Frontend physics | Cannon.js (local prediction) |
| Build tool | Vite |
| Audio | Howler.js |
| Animations | GSAP |
| Networking client | Socket.IO client |
| Networking server | Node.js + Socket.IO |
| Server physics | Cannon.js (authoritative) |
| Deploy | VPS (victorgalvez.dev) — PM2 + nginx |

---

## Multiplayer Architecture

```
┌──────────────────────────────────────────────────┐
│  CLIENT (Browser)                                │
│                                                  │
│  ┌──────────┐  inputs   ┌────────────────────┐   │
│  │ Controls │ ────────► │ Socket.IO Client   │   │
│  └──────────┘           └────────┬───────────┘   │
│                                  │ snapshots @20Hz│
│  ┌─────────────────┐             ▼               │
│  │  LocalCar        │   ┌────────────────────┐   │
│  │  (full Cannon.js)│   │  RemoteCarManager  │   │
│  │  client predict  │   │  lerp pos/rot      │   │
│  └─────────────────┘   └────────┬───────────┘   │
│                                  │               │
│              ┌───────────────────┘               │
│              ▼                                   │
│  ┌──────────────────────────────────┐            │
│  │  RemoteCar[B]  RemoteCar[C] ...  │            │
│  │  Three.js meshes only, no physics│            │
│  └──────────────────────────────────┘            │
└──────────────────────────────────────────────────┘
                │                │
          inputs│                │snapshots
                ▼                ▼
        ┌──────────────────────────────┐
        │  Node.js Server              │
        │                              │
        │  Socket.IO rooms             │
        │  ┌────────────────────────┐  │
        │  │  Cannon.js World       │  │
        │  │   car[A] physics body  │  │
        │  │   car[B] physics body  │  │
        │  │   car[C] physics body  │  │
        │  └────────────────────────┘  │
        │                              │
        │  Broadcast @ 20 Hz:          │
        │  { id, pos, quat, vel,       │
        │    wheels, actions }         │
        └──────────────────────────────┘
```

### Why server-authoritative physics?
- Prevents desync between clients
- Prevents cheating
- Single source of truth for all car positions
- Client still runs local physics for responsive feel (client prediction), then corrects to server state

---

## Network Payload

Per player state snapshot — ~60 bytes:

```json
{
  "id": "player_abc123",
  "pos": [x, y, z],
  "quat": [x, y, z, w],
  "vel": [x, y, z],
  "angVel": [x, y, z],
  "wheels": [rot0, rot1, rot2, rot3],
  "actions": 5
}
```

`actions` is a bitmask: `up(1) | down(2) | left(4) | right(8) | brake(16) | boost(32)`

**Bandwidth estimate:**
- 10 players × 60 bytes × 20 Hz = ~12 KB/s per client — negligible.

---

## Repository Structure

```
folio-bruno-multiplayer/
├── client/                        # Vite + Three.js frontend
│   ├── src/
│   │   ├── index.html
│   │   ├── index.js
│   │   └── javascript/
│   │       ├── Application.js     # Main orchestrator — add network init here
│   │       ├── Camera.js          # No changes needed
│   │       ├── Network.js         # NEW — Socket.IO client wrapper
│   │       └── World/
│   │           ├── index.js       # Add RemoteCarManager init
│   │           ├── Car.js         # Minor: expose setFromSnapshot()
│   │           ├── Controls.js    # Add: emit input changes to server
│   │           ├── RemoteCar.js   # NEW — remote player car (no physics)
│   │           └── RemoteCarManager.js  # NEW — manages remote car instances
│   ├── static/                    # Models, textures, audio (unchanged)
│   └── package.json
│
├── server/                        # Node.js authoritative server
│   ├── src/
│   │   ├── index.js               # Entry point — Socket.IO setup
│   │   ├── GameRoom.js            # Room logic, player management
│   │   ├── PhysicsWorld.js        # Cannon.js world (mirrors client physics)
│   │   └── PlayerCar.js          # Per-player physics body on server
│   └── package.json
│
├── shared/
│   └── constants.js               # Physics params, tick rate, shared config
│
├── MULTIPLAYER_PLAN.md
└── README.md
```

---

## Phase 1 — Base Setup (Days 1–2)

### 1.1 Clone and restructure

```bash
# Clone original repo into client/
git clone https://github.com/brunosimon/folio-2019 client

# Init server
mkdir server && cd server && npm init -y
npm install socket.io cannon express
npm install -D nodemon

# Init shared
mkdir shared
```

### 1.2 Server entry point — `server/src/index.js`

```js
import { createServer } from 'http'
import { Server } from 'socket.io'
import { GameRoom } from './GameRoom.js'

const httpServer = createServer()
const io = new Server(httpServer, {
  cors: { origin: '*' }
})

const room = new GameRoom(io)

httpServer.listen(3001, () => {
  console.log('Server running on :3001')
})
```

### 1.3 Vite proxy config — `client/vite.config.js`

```js
export default {
  server: {
    proxy: {
      '/socket.io': {
        target: 'http://localhost:3001',
        ws: true
      }
    }
  }
}
```

---

## Phase 2 — Authoritative Physics Server (Days 2–4)

### 2.1 `server/src/PhysicsWorld.js`

Mirror the Cannon.js setup from the client's `Physics.js`. Must be identical so positions match.

```js
import * as CANNON from 'cannon'

export class PhysicsWorld {
  constructor() {
    this.world = new CANNON.World()
    this.world.gravity.set(0, 0, -3.25 * 4)  // Same as client
    this.world.broadphase = new CANNON.SAPBroadphase(this.world)
    this.world.allowSleep = true
    this.world.defaultContactMaterial.restitution = 0.2

    this.cars = new Map()  // playerId → PlayerCar

    this._setupFloor()
  }

  _setupFloor() {
    // Add the same floor plane as the client
    const floorBody = new CANNON.Body({ mass: 0 })
    floorBody.addShape(new CANNON.Plane())
    this.world.addBody(floorBody)
  }

  addCar(playerId, spawnPosition) {
    const car = new PlayerCar(this.world, spawnPosition)
    this.cars.set(playerId, car)
    return car
  }

  removeCar(playerId) {
    const car = this.cars.get(playerId)
    if (car) {
      car.removeFromWorld(this.world)
      this.cars.delete(playerId)
    }
  }

  applyInputs(playerId, actions) {
    const car = this.cars.get(playerId)
    if (car) car.applyActions(actions)
  }

  step(delta) {
    this.world.step(1 / 60, delta, 3)
  }

  getSnapshot() {
    const snapshot = []
    for (const [id, car] of this.cars) {
      snapshot.push({ id, ...car.getState() })
    }
    return snapshot
  }
}
```

### 2.2 `server/src/PlayerCar.js`

```js
import * as CANNON from 'cannon'

export class PlayerCar {
  constructor(world, spawnPos = { x: 0, y: 0, z: 0 }) {
    // Chassis body — same shape/mass as client Car.js
    this.chassis = new CANNON.Body({ mass: 500 })
    this.chassis.addShape(new CANNON.Box(new CANNON.Vec3(1, 2, 0.5)))
    this.chassis.position.set(spawnPos.x, spawnPos.y, spawnPos.z)
    world.addBody(this.chassis)

    this.vehicle = new CANNON.RaycastVehicle({
      chassisBody: this.chassis,
      indexRightAxis: 0,
      indexUpAxis: 2,
      indexForwardAxis: 1
    })

    this._addWheels()
    this.vehicle.addToWorld(world)
  }

  _addWheels() {
    const wheelOptions = {
      radius: 0.38,
      directionLocal: new CANNON.Vec3(0, 0, -1),
      suspensionStiffness: 30,
      suspensionRestLength: 0.3,
      frictionSlip: 1.4,
      dampingRelaxation: 2.3,
      dampingCompression: 4.4,
      maxSuspensionForce: 100000,
      rollInfluence: 0.01,
      axleLocal: new CANNON.Vec3(-1, 0, 0),
      chassisConnectionPointLocal: new CANNON.Vec3(),
      maxSuspensionTravel: 0.3,
      customSlidingRotationalSpeed: -30,
      useCustomSlidingRotationalSpeed: true
    }

    const positions = [
      { x:  0.85, y:  1.3, z: -0.2 },  // front-right
      { x: -0.85, y:  1.3, z: -0.2 },  // front-left
      { x:  0.85, y: -1.15, z: -0.2 }, // rear-right
      { x: -0.85, y: -1.15, z: -0.2 }  // rear-left
    ]

    for (const pos of positions) {
      wheelOptions.chassisConnectionPointLocal.set(pos.x, pos.y, pos.z)
      this.vehicle.addWheel(wheelOptions)
    }
  }

  applyActions(actions) {
    const maxForce    = actions.boost ? 1600 : 800
    const maxBrake    = 50
    const maxSteer    = 0.3
    const engineForce = actions.up ? maxForce : actions.down ? -maxForce * 0.5 : 0
    const brakeForce  = actions.brake ? maxBrake : 0
    const steer       = actions.left ? maxSteer : actions.right ? -maxSteer : 0

    this.vehicle.applyEngineForce(engineForce, 2)
    this.vehicle.applyEngineForce(engineForce, 3)
    this.vehicle.setBrake(brakeForce, 0)
    this.vehicle.setBrake(brakeForce, 1)
    this.vehicle.setBrake(brakeForce, 2)
    this.vehicle.setBrake(brakeForce, 3)
    this.vehicle.setSteeringValue(steer, 0)
    this.vehicle.setSteeringValue(steer, 1)
  }

  getState() {
    const p = this.chassis.position
    const q = this.chassis.quaternion
    const v = this.chassis.velocity
    const av = this.chassis.angularVelocity

    return {
      pos:    [p.x, p.y, p.z],
      quat:   [q.x, q.y, q.z, q.w],
      vel:    [v.x, v.y, v.z],
      angVel: [av.x, av.y, av.z],
      wheels: this.vehicle.wheelInfos.map(w => w.rotation)
    }
  }

  removeFromWorld(world) {
    this.vehicle.removeFromWorld(world)
    world.remove(this.chassis)
  }
}
```

### 2.3 `server/src/GameRoom.js`

```js
import { PhysicsWorld } from './PhysicsWorld.js'

const TICK_RATE = 20          // Hz — broadcast frequency
const PHYSICS_STEP = 1 / 60  // physics runs at 60 Hz internally
const MAX_PLAYERS = 20

export class GameRoom {
  constructor(io) {
    this.io = io
    this.physics = new PhysicsWorld()
    this.players = new Map()  // socketId → { id, name, actions, carColor }

    this._startLoop()
    this._setupSocketEvents()
  }

  _setupSocketEvents() {
    this.io.on('connection', (socket) => {
      if (this.players.size >= MAX_PLAYERS) {
        socket.emit('room:full')
        socket.disconnect()
        return
      }

      console.log(`Player connected: ${socket.id}`)

      socket.on('player:join', ({ name, carColor }) => {
        const spawnPos = this._getSpawnPosition()
        this.physics.addCar(socket.id, spawnPos)

        this.players.set(socket.id, {
          id: socket.id,
          name: name || 'Anonymous',
          carColor: carColor || 0,
          actions: { up: false, down: false, left: false, right: false, brake: false, boost: false }
        })

        // Send existing players to new joiner
        const existingPlayers = [...this.players.values()]
          .filter(p => p.id !== socket.id)
          .map(p => ({ id: p.id, name: p.name, carColor: p.carColor }))
        socket.emit('room:joined', { id: socket.id, existingPlayers })

        // Notify others of new player
        socket.broadcast.emit('player:joined', {
          id: socket.id,
          name: name || 'Anonymous',
          carColor: carColor || 0
        })
      })

      socket.on('player:input', (actions) => {
        const player = this.players.get(socket.id)
        if (player) player.actions = actions
      })

      socket.on('disconnect', () => {
        this.physics.removeCar(socket.id)
        this.players.delete(socket.id)
        this.io.emit('player:left', { id: socket.id })
        console.log(`Player disconnected: ${socket.id}`)
      })
    })
  }

  _startLoop() {
    let last = Date.now()
    let tickAccumulator = 0
    const tickInterval = 1000 / TICK_RATE

    setInterval(() => {
      const now = Date.now()
      const delta = (now - last) / 1000
      last = now

      // Apply inputs from all players
      for (const [id, player] of this.players) {
        this.physics.applyInputs(id, player.actions)
      }

      // Step physics at 60 Hz
      this.physics.step(delta)

      // Broadcast snapshot at TICK_RATE
      tickAccumulator += delta * 1000
      if (tickAccumulator >= tickInterval) {
        tickAccumulator -= tickInterval
        const snapshot = this.physics.getSnapshot()
        if (snapshot.length > 0) {
          this.io.emit('world:snapshot', { t: now, cars: snapshot })
        }
      }
    }, 1000 / 60)  // Loop at 60 Hz, broadcast at 20 Hz
  }

  _getSpawnPosition() {
    const angle = Math.random() * Math.PI * 2
    const radius = 5
    return {
      x: Math.cos(angle) * radius,
      y: Math.sin(angle) * radius,
      z: 1
    }
  }
}
```

---

## Phase 3 — Client Network Layer (Days 4–6)

### 3.1 `client/src/javascript/Network.js`

```js
import { io } from 'socket.io-client'
import { EventEmitter } from '../utils/EventEmitter.js'

export class Network extends EventEmitter {
  constructor() {
    super()
    this.socket = null
    this.localId = null
    this.latency = 0
    this._pingInterval = null
  }

  connect(serverUrl = '/') {
    this.socket = io(serverUrl)

    this.socket.on('connect', () => {
      console.log('Connected to server:', this.socket.id)
      this.trigger('connected')
    })

    this.socket.on('disconnect', () => {
      console.warn('Disconnected from server')
      this.trigger('disconnected')
    })

    this.socket.on('room:joined', ({ id, existingPlayers }) => {
      this.localId = id
      this.trigger('room:joined', [{ id, existingPlayers }])
    })

    this.socket.on('room:full', () => {
      this.trigger('room:full')
    })

    this.socket.on('player:joined', (data) => {
      this.trigger('player:joined', [data])
    })

    this.socket.on('player:left', (data) => {
      this.trigger('player:left', [data])
    })

    this.socket.on('world:snapshot', (snapshot) => {
      this.trigger('world:snapshot', [snapshot])
    })

    // Latency measurement
    this._pingInterval = setInterval(() => {
      const start = Date.now()
      this.socket.emit('ping', () => {
        this.latency = Date.now() - start
      })
    }, 2000)
  }

  join(name, carColor) {
    this.socket.emit('player:join', { name, carColor })
  }

  sendInput(actions) {
    this.socket.emit('player:input', actions)
  }

  disconnect() {
    clearInterval(this._pingInterval)
    this.socket?.disconnect()
  }
}
```

### 3.2 Modify `Controls.js` — send inputs to server

Add to the `setKeyboardAction` and touch handlers:

```js
// In Controls.js constructor
this.network = null  // Set from outside after Network is ready

// After any action state change:
_notifyNetwork() {
  if (this.network) {
    this.network.sendInput({ ...this.actions })
  }
}
```

Call `_notifyNetwork()` whenever `this.actions` changes.

### 3.3 `client/src/javascript/World/RemoteCar.js`

Remote cars don't run physics — they just interpolate between received snapshots.

```js
import * as THREE from 'three'

export class RemoteCar {
  constructor({ scene, resources, id, name, carColor }) {
    this.id = id
    this.name = name

    // Snapshot buffer for interpolation
    this.snapshots = []          // [{ t, pos, quat, wheels }]
    this.interpolationDelay = 100  // ms — render 100ms behind server

    this._buildMesh(scene, resources, carColor)
    this._buildNameLabel(name)
  }

  _buildMesh(scene, resources, carColor) {
    // Clone the same car model used by the local player
    // resources.items.carDefaultChassis is the loaded GLTF
    this.chassis = resources.items.carDefaultChassis.scene.clone()
    scene.add(this.chassis)

    this.wheels = []
    const wheelModel = resources.items.carDefaultWheel.scene
    for (let i = 0; i < 4; i++) {
      const wheel = wheelModel.clone()
      scene.add(wheel)
      this.wheels.push(wheel)
    }

    // Apply a tint to distinguish players
    const hue = (carColor / 8) * 360
    this.chassis.traverse(child => {
      if (child.isMesh && child.material) {
        child.material = child.material.clone()
        child.material.color.setHSL(hue / 360, 0.6, 0.5)
      }
    })
  }

  _buildNameLabel(name) {
    // CSS2DObject for floating name tag
    const div = document.createElement('div')
    div.className = 'remote-player-label'
    div.textContent = name

    // Requires CSS2DRenderer added to Application.js
    const { CSS2DObject } = await import('three/addons/renderers/CSS2DRenderer.js')
    this.label = new CSS2DObject(div)
    this.label.position.set(0, 0, 1.5)
    this.chassis.add(this.label)
  }

  addSnapshot(snapshot) {
    this.snapshots.push({
      t:      snapshot.t,
      pos:    snapshot.pos,
      quat:   snapshot.quat,
      wheels: snapshot.wheels
    })
    // Keep only last 3 seconds of snapshots
    const cutoff = Date.now() - 3000
    this.snapshots = this.snapshots.filter(s => s.t > cutoff)
  }

  update() {
    if (this.snapshots.length < 2) return

    // Render at (now - interpolationDelay) to always have two points to lerp between
    const renderTime = Date.now() - this.interpolationDelay

    // Find two snapshots that surround renderTime
    let older = null
    let newer = null
    for (let i = 0; i < this.snapshots.length - 1; i++) {
      if (this.snapshots[i].t <= renderTime && this.snapshots[i + 1].t >= renderTime) {
        older = this.snapshots[i]
        newer = this.snapshots[i + 1]
        break
      }
    }

    if (!older || !newer) return

    const alpha = (renderTime - older.t) / (newer.t - older.t)

    // Interpolate position
    this.chassis.position.lerpVectors(
      new THREE.Vector3(...older.pos),
      new THREE.Vector3(...newer.pos),
      alpha
    )

    // Slerp quaternion
    const q1 = new THREE.Quaternion(...older.quat)
    const q2 = new THREE.Quaternion(...newer.quat)
    this.chassis.quaternion.slerpQuaternions(q1, q2, alpha)

    // Update wheel rotations
    this.wheels.forEach((wheel, i) => {
      if (older.wheels && newer.wheels) {
        const rot = older.wheels[i] + (newer.wheels[i] - older.wheels[i]) * alpha
        wheel.rotation.x = rot
      }
    })
  }

  destroy(scene) {
    scene.remove(this.chassis)
    this.wheels.forEach(w => scene.remove(w))
    this.chassis.remove(this.label)
    this.label.element.remove()
  }
}
```

### 3.4 `client/src/javascript/World/RemoteCarManager.js`

```js
import { RemoteCar } from './RemoteCar.js'

export class RemoteCarManager {
  constructor({ scene, resources, network }) {
    this.scene = scene
    this.resources = resources
    this.network = network
    this.cars = new Map()  // id → RemoteCar

    network.on('player:joined', ({ id, name, carColor }) => {
      this._addCar(id, name, carColor)
    })

    network.on('player:left', ({ id }) => {
      this._removeCar(id)
    })

    network.on('world:snapshot', ({ t, cars }) => {
      for (const carState of cars) {
        // Skip local player — we handle that separately
        if (carState.id === network.localId) continue

        const car = this.cars.get(carState.id)
        if (car) car.addSnapshot({ t, ...carState })
      }
    })
  }

  _addCar(id, name, carColor) {
    if (this.cars.has(id)) return
    const car = new RemoteCar({
      scene: this.scene,
      resources: this.resources,
      id, name, carColor
    })
    this.cars.set(id, car)
  }

  _removeCar(id) {
    const car = this.cars.get(id)
    if (car) {
      car.destroy(this.scene)
      this.cars.delete(id)
    }
  }

  update() {
    for (const car of this.cars.values()) {
      car.update()
    }
  }
}
```

---

## Phase 4 — Wiring Everything Together (Day 6)

### 4.1 Modify `Application.js`

```js
import { Network } from './Network.js'

// In constructor, after resources are loaded:
this.network = new Network()
this.network.connect()

this.network.on('connected', () => {
  // Show lobby UI — ask for name
  this._showLobby()
})
```

### 4.2 Modify `World/index.js`

```js
// After car is set up:
this.remoteCarManager = new RemoteCarManager({
  scene: this.scene,
  resources: this.resources,
  network: this.application.network
})

// In tick handler:
this.remoteCarManager.update()

// Connect controls to network:
this.controls.network = this.application.network
```

### 4.3 Lobby UI (HTML overlay)

Simple overlay before joining:

```html
<!-- In index.html -->
<div id="lobby" class="lobby">
  <h1>Enter the Portfolio</h1>
  <input type="text" id="playerName" placeholder="Your name" maxlength="20" />
  <button id="joinBtn">Join</button>
  <p id="playerCount"></p>
</div>
```

```js
// lobby.js
document.getElementById('joinBtn').addEventListener('click', () => {
  const name = document.getElementById('playerName').value || 'Anonymous'
  network.join(name, Math.floor(Math.random() * 8))
  document.getElementById('lobby').style.display = 'none'
})
```

---

## Phase 5 — Polish (Days 7–8)

### 5.1 Player count HUD

```js
// Floating counter top-right
const hud = document.createElement('div')
hud.id = 'player-count'
hud.style.cssText = 'position:fixed;top:20px;right:20px;color:white;font-family:monospace'
document.body.appendChild(hud)

network.on('player:joined', () => updateHUD())
network.on('player:left',   () => updateHUD())

function updateHUD() {
  hud.textContent = `${remoteCarManager.cars.size + 1} online`
}
```

### 5.2 Reconnection handling

Socket.IO handles reconnection automatically. Add a visual indicator:

```js
network.on('disconnected', () => {
  showToast('Connection lost — reconnecting...')
})
network.on('connected', () => {
  hideToast()
})
```

### 5.3 CSS for name labels

```css
.remote-player-label {
  background: rgba(0, 0, 0, 0.6);
  color: white;
  padding: 2px 8px;
  border-radius: 4px;
  font-family: monospace;
  font-size: 12px;
  pointer-events: none;
  user-select: none;
}
```

### 5.4 Different car colors

Assign a color index (0–7) per player on join. Map to HSL hues:

```js
const CAR_COLORS = [
  '#e74c3c', // red
  '#3498db', // blue
  '#2ecc71', // green
  '#f39c12', // orange
  '#9b59b6', // purple
  '#1abc9c', // teal
  '#e91e63', // pink
  '#ffffff'  // white
]
```

---

## Phase 6 — Deploy to VPS (Days 8–9)

### 6.1 Build client

```bash
cd client && npm run build
# Output: client/dist/
```

### 6.2 Server setup on VPS

```bash
# On VPS: install Node 20 + PM2
npm install -g pm2

# Clone repo
git clone <repo> /var/www/folio-multiplayer
cd /var/www/folio-multiplayer/server && npm install

# Start server
pm2 start src/index.js --name folio-server
pm2 save && pm2 startup
```

### 6.3 Nginx config

```nginx
server {
    listen 80;
    server_name folio.victorgalvez.dev;

    # Serve static client build
    root /var/www/folio-multiplayer/client/dist;
    index index.html;

    # Proxy WebSocket to Node server
    location /socket.io/ {
        proxy_pass http://localhost:3001;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }

    # SPA fallback
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### 6.4 HTTPS (required for wss://)

```bash
certbot --nginx -d folio.victorgalvez.dev
```

---

## Risks and Mitigations

| Risk | Probability | Impact | Mitigation |
|------|------------|--------|------------|
| Client/server physics desync | Medium | Medium | Server is authoritative; client smoothly corrects to server state via lerp |
| Visible lag on remote cars | High | Low | 100ms interpolation buffer + snapshot lerp makes it smooth |
| Cannon.js not deterministic cross-environment | Medium | Low | Doesn't matter — server is sole source of truth |
| World objects (ramps, pins) out of sync | Low | Low | Static objects are local; only cars sync |
| High player count performance | Low | Medium | 20 Hz broadcast + tiny payloads + instanced rendering if needed |
| Mobile WebSocket disconnects | Medium | Low | Socket.IO auto-reconnect + exponential backoff |

---

## What Does NOT Need to Change

- Three.js rendering pipeline — untouched
- Cannon.js local simulation for the local player — untouched
- All models, textures, shaders — untouched
- Camera system — follows local car as before
- Sounds, GSAP animations, sections, zones — untouched
- Controls — only add one `sendInput()` call on state change

---

## Implementation Order (Critical Path)

```
Day 1   Clone repo → restructure into client/ + server/
Day 2   Server: PhysicsWorld + PlayerCar + GameRoom skeleton
Day 3   Server: Tick loop + broadcast snapshots
Day 4   Client: Network.js + join flow + lobby UI
Day 5   Client: RemoteCar.js (mesh clone + interpolation)
Day 6   Client: RemoteCarManager + wire into World/index.js
Day 7   Client: Controls → sendInput() + local car correction
Day 8   Polish: name labels, colors, HUD, reconnection
Day 9   Deploy: VPS + nginx + HTTPS + smoke test
```

---

## Total Estimated Effort: 8–10 focused days

The project is architecturally clean. The decoupled physics→rendering pattern, action-state controls, and EventEmitter-based communication make this a straightforward layering of a network synchronization system on top of an already well-structured codebase. No major rewrites required.
