**Repository Overview**

- **Purpose**: small Node.js multiplayer game server (Flappy Penguin) serving a static frontend in `public/` and real-time multiplayer via `socket.io`.
- **Main entry**: `server.js` (uses `express` + `socket.io`).
- **Containerization**: `Dockerfile` uses a two-stage build and switches to a non-root user in the runtime image.
- **Kubernetes**: `k8sdeploy.yaml` provides Deployment, Service and Ingress; imagePullPolicy is `Always` and the container listens on port `3000`.

**Key runtime patterns**

- **In-memory state**: `players`, `gameStates`, and `leaderboard` are plain in-memory maps/arrays inside `server.js`. Restarting the process will lose all game/leaderboard data.
- **Leader logic**: leadership is computed with `getCurrentLeader()` by scanning `players` and selecting the active player with the highest score.
- **Leaderboard update**: `updateLeaderboard(name, score)` keeps a sorted top-10 array — used on `gameOver` events.
- **Socket event names**: `playerJoin`, `gameStart`, `gameStateUpdate`, `scoreUpdate`, `gameOver`, `leaderboardUpdate`, `leaderGameState`, `playerScoreUpdate`. Use these exact names when adding/removing listeners in client code under `public/`.

**Where to change behavior**

- To persist scores across restarts: replace in-memory `leaderboard` with a DB call in `updateLeaderboard()` and on server start. See `server.js` lines that call `updateLeaderboard` and `getCurrentLeader`.
- To proxy or scale sockets: the app currently uses in-process Socket.IO. For multi-replica Kubernetes deployments, integrate a Socket.IO adapter (Redis) and make `k8sdeploy.yaml` use a scalable image and resource requests.

**Developer workflows & commands**

- Local dev server: `npm install` then `npm run dev` (uses `nodemon` to restart on changes).
- Run production locally: `npm start` (runs `node server.js`).
- Build Docker image (from `game/`):

```
docker build -t <registry>/<repo>:<tag> .
```

- Push and deploy to Kubernetes (examples):

```
docker push <registry>/<repo>:<tag>
kubectl apply -f k8sdeploy.yaml
kubectl rollout status deployment/k8sgame-deployment
```

- Port/Env: the server listens on `process.env.PORT || 3000`. Kubernetes forwards port 80 -> 3000 per `k8sdeploy.yaml`.

**Project conventions & shortcuts**

- Static frontend lives in `public/` — edit `game.html`, `game.js`, `index.html`, `leaderboard.html` here. The server serves them via `express.static()`.
- Keep socket event names and payload shapes stable: other parts of the app assume a leader's `gameState` object format and player objects with `id`, `name`, `score`, `isPlaying`.
- Use the `Dockerfile` multi-stage build pattern: dependencies installed in `builder` stage, runtime stage copies `node_modules` and app. Keep changes compatible with this layout.

**Safety and testing notes discovered in code**

- No automated tests in repo — changes to leaderboard, scoring, or leader-election logic should be validated manually or with unit tests you add.
- Because state is in-memory, running multiple replicas without a Socket.IO adapter will break real-time consistency.

**Useful files to inspect when working on features/bugs**

- `server.js` — core game/socket logic and in-memory state handling.
- `package.json` — scripts: `start`, `dev`, and declared dependencies.
- `Dockerfile` — build/runtime image structure and non-root user setup.
- `k8sdeploy.yaml` — Kubernetes deployment/service/ingress configuration.
- `public/game.js` and `public/game.html` — client-side event usage and expected payloads.

If anything here is unclear or you'd like extra examples (unit tests for `getCurrentLeader()` or an example Socket.IO Redis adapter integration), tell me which area and I will extend this doc.
