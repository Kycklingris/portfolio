---
layout: home
title: Malte LN
---

{% include project_card.html 
	title="Jackbox style game" 
	date="June 2024 ~100 hours so far" 
	image=""
	read-more="/projects/jackbox style game"
	content=
"
My main personal project with a focus on development tools for the use of modding. It is a multi faceted project that requires work in backend, frontend and game development. Currently reworking the system that connects the Frontend and Host game.
- Backend built with **ASP.NET Core**, **CoTURN**, **Docker** and **Redis**
- Frontend built with **Lit** and **JQuery**
- Host game built with **Godot**
"
%}

{% include project_card.html 
	title="Rendering" 
	date="Started November 2023 ~120 hours so far" 
	image="smooth_shadows.png"
	read-more="/projects/rendering"
	content=
"
There isn't all that much to show currently. My goal with the project is voxel cone tracing similar to [The Tomorrow Children](https://www.gamedeveloper.com/programming/graphics-deep-dive-cascaded-voxel-cone-tracing-in-i-the-tomorrow-children-i-) where both direct and indirect lighting is calculated using vct.
- Working with **Rust** and **WGPU**
- Soft shadows
- Currently a lot of light leak
- Compute shader per triangle scanline based voxelization
- Basic voxel rendering for debugging
"
%}

{% include project_card.html 
	title="Rift Breaker" 
	date="2023 September - 4 weeks" 
	video="5gsygQafOQM" 
	read-more="/projects/riftbreaker"
	content=
"
Rift breaker is a first person shooter with two variations of the same world that can be swapped between. It was my fourth group game project at Futuregames.
- Working with **C++** in **Unreal Engine 5**
- Player weapons
- World switching
- Highscores using **Rust**
"
%}

{% include project_card.html 
	title="ECOCIDE" 
	date="2023 May - 7 weeks" 
	video="T7aXZeEGoUo"
	read-more="/projects/ecocide"
	content=
"
Ecocide is a first person ability/grab and throw based defensive action game. It was my third group game project at Futuregames.
- Working with **C++** in **Unreal Engine 5**
- Ragdoll System
- Windblast Ability
- Spike Wall
"
%}

{% include project_card.html 
	title="B34-N" 
	date="2023 Febuary - 4 weeks" 
	video="G3C1li83WUY"
	read-more="/projects/b34n"
	content=
"
B34-N is an online multiplayer top-down cooperative shooter. 
This was my second group game project at Futuregames, and can only be described as a failure. However, due to the unique aspect of online multiplayer I feel it is worth showcasing.
- Working with **C#** in **Unity**
- Online networking using the **Mirror** library
"
%}