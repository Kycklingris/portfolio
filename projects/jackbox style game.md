---
layout: project
title: Riff Party
group_count: 1
---
<video class="project-video" src="/assets/projects/riff_party/demo.mp4" poster="/assets/projects/riff_party/demo_poster.jpg" autoplay muted loop playsinline controls preload="metadata" aria-label="Riff Party gameplay demo"></video>

Riff Party is a Jackbox-style party game. The game runs on a host machine, and players join through the browser. Every game mode is a mod, and the whole project is built around making new games easy to create.

The project spans three layers:
- **Game modes** are mods written in **GDScript**, each in its own Godot project. Mod authors never touch networking, Protobuf, TypeScript or C++.
- **The engine** is a **Godot 4** fork with a custom **C++** module. It defines the UI elements and resources mods use, and sends them to the players.
- **The frontend** is written in **TypeScript** with **Lit** and **XState**, and renders each element as a web component on the player's phone.

The backend is live, deployed with infrastructure as code, and the game has been playtested with real players.

*The architecture and the core of the project, including the network object system, backend, initial infrastrucure, the style system, buttons, text input and real-time collaborative drawing with undo/redo, are written by me. Since mid-2026 I've used AI tools for much of the newer feature work, with me doing the design and review. I am still not certain on how or even if I want to use AI for future projects.*

# Host to frontend connection
I decided early on that the connection should be done through WebRTC, for two reasons.
1. Cost: the less time a server is involved, the lower the running cost.
2. Latency: I want to keep the door open for real-time controller games when playing in the same room, for example a karting game or a fighting game.

Two Rust services facilitate the connection:
- A **lobby server** (Axum, Twirp) that creates and joins lobbies and hands out TURN credentials
- A **signaling server** (WebSockets) that relays the WebRTC offers, answers and ICE candidates between the host and each player

The services talk to each other over **ZeroMQ** and share state in **Redis**. The signaling server keeps the host's connection resident for the whole lobby, with heartbeats to detect dead hosts. Lobbies expire on their own through Redis TTLs, and idle hosts can be parked when the server is at capacity. Once WebRTC is connected, all game traffic goes peer-to-peer (or via CoTURN when NAT gets in the way) as binary Protobuf.

# Controlling the frontend from the host
The host acts as a highly opinionated server-side rendering library. Mods build their UI from a set of element nodes in Godot (buttons, text fields, images, sliders and more), and each one maps directly to a Lit web component on the player's phone. I have setup a embedded browser in godot to preview any changes in real time.

{% include preload_img.html
  src="/assets/projects/riff_party/visual_editor.gif"
  aspect_ratio="2558/1390"
  alt="Animation of the visual editor in Godot being used to build player UI"
%}

Every element and resource is a **network object** that exists in all three layers at once. The messages between them are defined with Protobuf, so changing a property in GDScript updates the right element on every connected phone. A style system on top of this lets mods style elements (including animated transitions) without writing CSS.

Players can also draw together. The collaborative drawing system syncs strokes between players in real time and supports undo/redo.

{% include preload_img.html
  src="/assets/projects/riff_party/collaborative_drawing.gif"
  aspect_ratio="2558/1390"
  alt="Animation of two browser windows drawing on the same shared canvas in real time"
%}

# Infrastructure
![Diagram of the Riff Party deployment: phones and the host game reach the lobby and signaling server through Cloudflare, the server reaches Redis over Tailscale, game data goes peer-to-peer with CoTURN relays as a fallback, and traces and logs go to Grafana Cloud](/assets/projects/riff_party/architecture.svg)

The whole backend is defined in **OpenTofu** (Terraform) and runs on a few small cloud servers. Players and the host only ever talk to the backend through Cloudflare, to create and join lobbies and to set up the WebRTC connection. After that, game data goes directly between the host and the phones, with CoTURN relays as a fallback when NAT blocks a direct connection.

## Why this setup
- **Virtual machines and Docker Compose instead of Kubernetes.** At this scale, a handful of small servers is cheaper and much simpler to run and debug than a cluster.
- **Several cloud providers.** TURN relays use a lot of bandwidth but little CPU, so they run on Oracle Cloud's free tier, while the main server runs on a cheap Hetzner VPS.
- **Cloudflare in front.** It handles DNS, TLS and rate limiting, so the servers themselves don't have to.

## Placing services on servers
Servers and the services running on them are declared separately. A `server` module turns each node into a VM on its provider (Oracle Cloud, Hetzner or DigitalOcean), so moving a service, or a whole server, to a different provider is a small config change:

```terraform
nodes = {
  "node-main" = { provider = "hetzner", instance_type = "cx23", location = "nbg1" }
  "node-1"    = { provider = "oracle",  instance_type = "VM.Standard.E2.1.Micro" }
  "node-2"    = { provider = "oracle",  instance_type = "VM.Standard.E2.1.Micro" }
}

apps = {
  "redis-main" = { role = "redis",  node = "node-main" }
  "coturn-1"   = { role = "coturn", node = "node-1" }
  "coturn-2"   = { role = "coturn", node = "node-2" }
  # ...
}
```

```terraform
module "nodes" {
  for_each = var.nodes
  source   = "./modules/server"

  provider_name = each.value.provider
  instance_type = each.value.instance_type
  location      = each.value.location
  server_id     = each.key

  # A cloud-init script stitched together from a shared base setup
  # and one template per app placed on this node
  user_data = data.cloudinit_config.setup[each.key].rendered
}
```

## Private networking
Internal services are never exposed to the internet. Every node, and every service on it, joins a private **Tailscale** network with its own auth key. Redis only listens on localhost, and a Tailscale sidecar forwards traffic to it from the private network, so the backend reaches Redis by its private hostname while the public internet can't reach it at all:

```yaml
services:
  ts-sidecar:
    image: tailscale/tailscale:v${tailscale_version}
    network_mode: host
    environment:
      - TS_AUTHKEY=${tailscale_api_key}   # one key per app, generated by Terraform
      - TS_HOSTNAME=${app_id}             # e.g. redis-main.<tailnet>.ts.net
      - TS_SERVE_CONFIG=/config/serve.json  # forwards tailnet:6379 -> 127.0.0.1:local_port
    healthcheck:
      test: ["CMD-SHELL", "tailscale status --json | grep -q '\"BackendState\":\"Running\"'"]

  redis:
    image: redis:8-alpine
    network_mode: host
    command: >
      redis-server
      --bind 127.0.0.1
      --port ${local_port}
      --requirepass ${redis_password}
      --protected-mode yes
```

# Deployment pipeline
Everything is deployed by **GitLab CI**:
1. **Build:** multi-stage Docker builds for amd64 and arm64, with cargo-chef dependency caching and static binaries in `FROM scratch` images.
2. **Plan:** merge requests that touch the infrastructure get a Terraform plan report.
3. **Deploy:** releases (`release-vX.Y.Z` tags) are applied to production, with GitLab managing the Terraform state and locking.

# Observability
The backend services export traces and logs with **OpenTelemetry** to **Grafana** and **Loki**, and the frontend reports errors with **Grafana Faro**.
