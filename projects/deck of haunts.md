---
layout: project
title: Deck Of Haunts
group_count: 8
source_url: https://store.steampowered.com/app/3179730/Deck_of_Haunts/
url_text: Steam
---
<iframe class="project-video" src="https://www.youtube.com/embed/1ADA3Ntrr7U" title="Deck of Haunts teaser" allowfullscreen></iframe>

Deck Of Haunts is a roguelike deckbuilder by Mantis Games. I worked on it for about 8 months as my Futuregames internship, remotely from Sweden while the studio is based in Ghent. The game was released on Steam a few months after my internship ended.

(We were 3 programmers including me, 2 artists plus 2 art interns for part of the time, and a project manager who also did VFX, with consultants for game design and music.)

- Working with **C++** in **Unreal Engine 5**
- **Plastic SCM** for versioning

# The card system
My main responsibility was the card system that every card in the game runs on, along with implementing most of the cards themselves.

The system is written in C++, and each card is composed from reusable pieces, requirements that decide when a card can be played, effects that decide what it does, and card text that is tied to both. New cards can be built by combining existing pieces, and new behaviour only needs a new requirement or effect rather than a new card class.

A year after launch, the team's senior programmer wrote in the [anniversary update post](https://store.steampowered.com/news/app/3179730/view/516367055196259433?l=english) that the card system "has proven to be very robust and flexible when it comes to adding new functionality".

## Looking back
I kept the system close to the game's previous card system, which was based on Data Assets. Linking text, requirements and effects together as instanced objects made the system flexible for programmers, but more complicated than it needed to be for authoring cards. Today I would build it on Blueprints, so designers could create and tweak cards more directly and adding a requirement for an effect is as simple as an if statement (or well branch in Blueprints).

# Other work
Since the team was small, I also worked on features across the rest of the game:
- Saving
- Settings UI
- Effects
- A debug menu
- Smaller features, such as a cat that roams the entrance room
- Bug fixing wherever it was needed

# What I took from it
Working remotely and mostly independently meant taking full responsibility for my own work, while staying in sync with the rest of the team on how work was split up and delegated.
