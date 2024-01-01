---
layout: project
title: ECOCIDE
group_count: 13
url: https://futuregames.itch.io/ecocidegame
url_text: Itch.io
---
(We were 4 programmers)
# Ragdoll system
The ragdoll system consists of a couple of features, such as partial ragdoll for limbs, half ragdoll half animation and ragdoll while still using the capsule collider.

![Gif showing partial ragdoll](/assets/projects/ecocide/partial_ragdoll.gif)

Now, I can't say I am particularly happy with the code quality as I should been using a state machine,
as can be seen below, there a multiple checks for states and possible transistions. Which would have been easier to manage if I were to have used a state machine.

```cpp
// Check if physics should be turned off
if (PhysicsOn)
{
	RequiredFlyingTimeLeft -= DeltaTime;

	if ((DistanceFromGround <= 2.f || VelocityLength <= 400.f) && RequiredFlyingTimeLeft <= 0.f)
	{
		Collider->SetSimulatePhysics(false);
		PhysicsOn = false;
		GetOwner()->SetActorRotation(FRotator(0.f, 0.f, 0.f));
	}
}

// Have the capsule collider follow the ragdoll.
if (IsRagdolling)
{
	GetOwner()->SetActorLocation(Mesh->GetComponentLocation(), false, nullptr, ETeleportType::None);
}

if (!IsRagdolling && (VelocityLength >= RagdollRequiredForce) && !JustEnteredPhysics)
{
	EnterRagdoll(Velocity * RagdollTransitionMultiplier);
}
else if (!ExitingRagdoll && IsRagdolling && VelocityLength <= 40.0f && DistanceFromGround < 50.f)
{
	StartEaseOut();
}
else if (ExitingRagdoll)
{
	EaseOutRagdoll(DeltaTime);
}

if (JustEnteredPhysics)
{
	JustEnteredPhysics = false;
}
```

# Windblast ability
Originally it was implemented using a large amount of raycasts uniformly distributed in a circle, this however lead to a noticable drop of fps in unreals profiler. As such it was later replaced with a much smaller amount of sphere casts which was performant enough to where I was unable to find any drops in fps. 

The reason for not using just checking for intersection with a cone in front of the player was granularity, some examples being
1. If the windblast is partially blocked by a rock, any enemies behind the rock should not be affected.
2. If a enemy is behind another enemy the strength of the windblast should be lowered.

![Gif showing partial windblast](/assets/projects/ecocide/partial_windblast.gif)

Now, in hindsight it likely would have been enough to check for intersection in a cone and running a single ray/sphere cast between every enemy and the player to check for strength or blockage.

<!-- 
Dennis Larson  GD
Filip Ekström  GD
Mark Victor Laguitan  GD
Lea Koinberg GA BOD
Moa Jerneholt GA   SKE
Gautham Satheesh  GA BOD
Malte Linde Neveling GP BOD
Kristoffer Saxmo GP BOD
Max Pålsson GP SKE
Simon Persson GP BOD
Leon Laszlo QA
Fredrik Modin QA
Christoffer Siltanen QA 
-->