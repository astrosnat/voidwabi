# Wabi - A Self-Hosted, Extensible Chat Platform

Wabi is a private, self-hosted, and ephemeral real-time chat application designed for small to medium-sized communities (10-50 users). It runs as a web application and can be packaged as a native desktop application using Tauri. It prioritizes privacy and simplicity, with all chat data stored in-memory and disappearing when the server restarts.

## Features

- **Real-time Chat**: Instant messaging, typing indicators, user presence, and GIF support.
- **Voice & Video Calls**: WebRTC-based direct and group calls.
- **Screen Sharing**: Share your screen with multiple participants.
- **Native Desktop App**: Cross-platform desktop application built with Tauri.
- **Extensible via Plugins**: Add custom functionality to the backend via a simple plugin architecture.
- **File Sharing**: Upload and share files directly in chat.
- **Ephemeral & Private**: All data is stored in-memory on your own server. No database required, and no data persists after a restart.
- **Self-Hosted Relay Server**: Includes a pre-configured `coturn` TURN server to improve WebRTC connection reliability.
- **Collaborative Drawing**: Integrated Excalidraw whiteboard for team collaboration.

## Tech Stack

- **Backend**: Node.js (Bun Runtime), Socket.IO, TypeScript
- **Frontend**: SvelteKit, TypeScript
- **Desktop**: Tauri
- **Real-time Comms**: Socket.IO (Signaling), WebRTC (Media)
- **Relay Server**: Coturn, Docker & Docker Compose

## Project Structure

```
.
├── backend/         # Node.js Socket.IO backend
├── frontend/        # SvelteKit frontend & Tauri desktop wrapper
├── plugins/         # Directory for backend plugins
├── turn-server/     # Pre-configured Coturn TURN server for Docker
└── pureref-connector/ # Standalone tool for PureRef integration
```

## Getting Started

### Prerequisites

- [Bun](https://bun.sh/) v1.0 or higher
- [Docker](https://www.docker.com/get-started) & [Docker Compose](https://docs.docker.com/compose/install/) (for the TURN server)
- A modern web browser

### 1. Installation

Clone the repository and install all dependencies using Bun's workspace feature.

```bash
git clone <your-repo-url>
cd <repo-name>
bun install
```

### 2. Configure the Backend URL

The frontend needs to know where the backend server is running. For development, this is hardcoded.

- **Open `frontend/src/lib/socket.ts`**
- **Modify the `io()` connection string** to point to your backend's IP address or URL. By default, it may point to a development `ngrok` URL. Change it to `http://localhost:3000` for local development.

```typescript
// In frontend/src/lib/socket.ts
// Before
const socket: Socket = io('https://your-backend.ngrok.io');

// After (for local dev)
const socket: Socket = io('http://localhost:3000');
```

### 3. Configure and Run the TURN Relay Server

For voice/video calls to work reliably, you need the TURN relay server.

1.  Navigate to the `turn-server` directory: `cd turn-server`.
2.  **Open `turnserver.conf`** and change `external-ip=127.0.0.1` to your machine's local IP address (or public IP for production).
3.  Start the server using Docker Compose: `docker-compose up -d`.
4.  Navigate back to the root directory: `cd ..`

### 4. Run the Application

Start the backend and frontend development servers concurrently from the root directory.

```bash
bun run dev
```

- The frontend will be available at `http://localhost:5173`.
- The backend will be running on `http://localhost:3000`.

## Architecture

### Backend Plugin System

The backend features a simple yet powerful plugin system located in the `plugins/` directory.

- The server automatically loads any subdirectory in `plugins/` that contains a `plugin.json` manifest.
- Each plugin's entry point receives a `context` object, giving it access to the core `io` (Socket.IO server) instance and the application's `state`.
- This allows you to listen for events, emit new ones, and modify the shared server state, enabling deep integration and extension of core functionality.

### WebRTC and the TURN Server

For reliable WebRTC connections (especially across different networks), STUN and TURN servers are used.

- **STUN**: Public STUN servers are pre-configured to help clients discover their public IP address.
- **TURN**: This project includes a pre-configured `coturn` TURN server in the `/turn-server` directory. It relays media traffic when a direct peer-to-peer connection fails. The frontend is already configured to use it, but you must ensure it's running and its IP is correctly configured in `turnserver.conf` and `frontend/src/lib/(calling|webrtc).ts`.

## Building for Production

This application can be built into a native desktop application.

1.  Ensure all configurations (especially server URLs) are set for production.
2.  Run the Tauri build command from the `frontend/` directory:

```bash
cd frontend
bun run tauri build
```

This will generate a native executable for your operating system in `frontend/src-tauri/target/release/`.

## Contributing

This project is under active development. Key areas for contribution include:

- [ ] Add authentication and user accounts
- [ ] Persist messages to an optional database (e.g., SQLite)
- [ ] Add message reactions
- [ ] Admin/moderation controls
- [ ] Custom themes
- [ ] Message threading

## License

MIT License