---
layout: project
title: Jackbox style game
group_count: 1
---
# Host to frontend connection
To start of with, I decided that the connection should be done through WebRTC, due to two reasons.
1. Cost, if I can minimize the amount of time a server is involved, then that means a lower cost for me.
2. Latency, mainly as I would like to maximize the possibility of doing a real time controller game (for example a karting game or fighting game).

WebRTC signaling as well as lobbies are handled through a Rust + Axum server as well as a Redis instance for horizontal scalability albeit that is so far untested and I am so far only planning for the future.

# Controlling the frontend from the host
The host kind of acts as a highly opinionated server side rendering library, where a set of C++ components in Godot directly map to a set of Lit components on the frontend.

{% include preload_img.html
  src="/assets/projects/jackbox_style_game/TPG_Editor_2025-07-25.jpg"
  aspect_ratio="1394/1020"
  alt="Image showing the godot editor with the Elements main panel and properties panel"
%}

The messages between the two is defined using Protobufs.