# Monolith War Concept

Monolith War is a real-time strategy focused on fast iteration and clear data flow. The world is a 400x240 grid represented with flat `Uint8Array` buffers, ensuring deterministic updates on the server and predictable rendering on the client. The map is split into 16x16 chunks that double as Socket.IO rooms, giving us efficient fog-of-war replication without manual filtering.

Core gameplay revolves around painting territory and commanding units. The server runs an authoritative 15 TPS loop that processes player commands, updates unit movement, handles painting/capture logic, and broadcasts chunk-scoped snapshots. Clients render on HTML5 Canvas, interpolating snapshots for smooth motion while keeping UI overlays in plain HTML/CSS.

The stack is intentionally minimal: Node.js 22 with strict TypeScript on the backend, Vite-powered TypeScript on the frontend, and shared types/constants in a workspace package. Assets live under `client/public/assets`, and deployment targets a container-friendly build pipeline.
