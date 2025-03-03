---
layout: project
title: Jackbox style game
group_count: 1
---
# Host to frontend connection
To start of with, I decided that the connection should be done through WebRTC, due to two reasons.
1. Cost, if I can minimize the amount of time a server is involved, then that means a lower cost for me.
2. Latency, mainly as I would like to maximize the possibility of doing a real time controller game.

WebRTC signaling as well as lobbies are handled through a Rust + Axum server as well as a Redis instance for horizontal scalability (untested).

# Controlling the frontend from the host
The host kind of acts as a highly opinionated server side rendering library, where a set of C++ components in Godot directly map to a set of Lit components on the frontend.