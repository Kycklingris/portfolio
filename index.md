---
layout: home
title: Malte LN
---

{% include project_card.html 
	title="Deck Of Haunts" 
	date="June 2024 - January 2025"
	video="1ADA3Ntrr7U"
	read-more="/projects/deck of haunts"
	content=
"
Deck Of Haunts is a roguelike deckbuilder, [released on Steam](https://store.steampowered.com/app/3179730/Deck_of_Haunts/) a few months after my ~8 month internship at Mantis Games. I built the card system and many of the cards. Since the team was small I also worked on other features across the game.
- Working with **C++** in **Unreal Engine 5**
- **Plastic SCM** for versioning
- The card system, built from composable requirements and effects
- Many of the cards
- Saving, settings UI and card effects
- Debug menu
- A year after launch, the team's senior programmer described the card system as 'very robust and flexible when it comes to adding new functionality'
"
%}

{% include project_card.html 
	title="Riff Party" 
	date="Started March 2024, 700+ hours"
	local-video="/assets/projects/riff_party/demo_card.mp4"
	poster="/assets/projects/riff_party/demo_poster.jpg"
	image-alt="Riff Party gameplay demo"
	read-more="/projects/jackbox style game"
	content=
"
My main personal project: a Jackbox-style party game where players join from their phones, designed from the start to be easily modded, by for example making sure the phone is controllable through single threaded GDScript. It covers the whole stack, from a custom Godot engine module to a Rust backend and the cloud infrastructure it runs on. The backend is live and the game has been playtested. The source is private as I want to release the game, but I'm happy to show more in a private setting.
- Host game built on a **Godot 4** fork with a custom **C++** engine module. Game modes are written as **GDScript** mods
- Frontend built with **TypeScript**, **Lit** and **XState**, talking to the host over **WebRTC** and **Protobuf**
- Backend built with **Rust**, **Axum**, **Redis**, **ZeroMQ** and **CoTURN**
- Infrastructure as code with **OpenTofu/Terraform**, deployed by **GitLab CI** and **Docker**
- **Git** for versioning
"
%}

{% include project_card.html 
	title="Rendering" 
	date="Started November 2023, ~120 hours" 
	image="smooth_shadows.png"
	read-more="/projects/rendering"
	content=
"
A from-scratch renderer in Rust where I explored GPU voxelization as a base for voxel cone traced lighting. It's paused for now; it taught me a lot about managing GPU resources, and if I pick it up again I'll build on an existing engine such as Bevy so I can focus on the rendering techniques.
- Working with **Rust** and **WGPU**
- **Git** for versioning
- Compute shader per triangle scanline based voxelization
- Voxel rendering for debugging
- Early soft shadows
- A helper crate with derive macros that cuts down WGPU boilerplate
"
%}

{% include project_card.html 
	title="Rift Breaker" 
	date="2023 September - 4 weeks fulltime" 
	video="5gsygQafOQM" 
	read-more="/projects/riftbreaker"
	content=
"
Rift breaker is a first person shooter with two variations of the same world that can be swapped between. It was my fourth group game project at Futuregames.
- Working with **C++** in **Unreal Engine 5**
- **Perforce** for versioning
- Player weapons
- World switching
- Highscores using **Rust**
"
%}

{% include project_card.html 
	title="ECOCIDE" 
	date="2023 May - 7 weeks fulltime" 
	video="T7aXZeEGoUo"
	read-more="/projects/ecocide"
	content=
"
Ecocide is a first person ability/grab and throw based defensive action game. It was my third group game project at Futuregames.
- Working with **C++** in **Unreal Engine 5**
- **Perforce** for versioning
- Ragdoll System
- Windblast Ability
- Spike Wall
"
%}
