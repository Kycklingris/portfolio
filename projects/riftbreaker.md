---
layout: project
title: Rift Breaker
group_count: 7
source_url: https://futuregames.itch.io/rift-breaker
url_text: Itch.io
---
(We were 4 programmers)
# Highscores
There are two parts to it
- A simple server that allows for requesting the highscores and submitting your own (I would have called it a REST server, but you can't edit or remove scores), it is written in Rust with [rusqlite](https://github.com/rusqlite/rusqlite) and [actix](https://actix.rs/), and hosted using Docker. [Github](https://github.com/Kycklingris/GP4_Team5_Highscore)
- The client side that also stores the current local highscore for any given weapon and level combination locally incase there is no internet connection.

{% include preload_img.html
  src="/assets/projects/riftbreaker/highscore.gif"
  aspect_ratio="549/309"
  alt="Gif showing a highscore being set"
%}

# Player weapons
Repeatable recoil resets after a specified amount of time; it is fully customizable without touching any code. 

{% include preload_img.html
  src="/assets/projects/riftbreaker/spray_patterns.gif"
  aspect_ratio="549/309"
  alt="Gif showing repeatable recoil patterns"
%}

The animations are easily customizable and allows for both *one bullet at a time* and magazine based reloads.

{% include preload_img.html
  src="/assets/projects/riftbreaker/partial_reload.gif"
  aspect_ratio="549/309"
  alt="Gif showing partial reloads"
%}

As well as easily replacable hit registration through unreal blueprints or c++; mainly used for the shotgun.

# World Switching
The world switching is implemented rather naively, purely switching on and off the rendering of half the world. There were two main problems with my approach, the first being the was a noticable lag spike when switching, and level design was difficult as there wasn't any button to swap the visible objects in editor. It did however allow for easier AI development as unlike other approaches the world position of everything stays the same and almost all of the colliders are static, which was the main reason for my decision.

The lag spike was caused by lumen recalculating all lighting the first time the world was switched. This was fixed by setting all lights to stationary and baking the light (Due to the time limit only the first level has baked lighting). I also experimented with using lighting scenarios, but the previously mentioned time constraints and there seemingly being no noticable difference between scenarios and just stationary lighting in performance caused me to choose the stationary lights instead.

{% include preload_img.html
  src="/assets/projects/riftbreaker/lagspike.gif"
  aspect_ratio="549/309"
  alt="Gif showing world switch lag spike"
%}

<!-- 
Mark
Bryan
Kim
Teddy
Ruben
Malte
Kristoffer 
-->