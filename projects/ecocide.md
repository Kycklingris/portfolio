---
layout: project
title: ECOCIDE
group_count: 13
source_url: https://futuregames.itch.io/ecocidegame
url_text: Itch.io
---
(We were 4 programmers)
# Ragdoll system
The ragdoll system consists a multitude of features, such as partial ragdoll for limbs, half ragdoll half animation and ragdoll while still using the capsule collider for physics.

{% include preload_img.html
  src="/assets/projects/ecocide/partial_ragdoll.gif"
  aspect_ratio="549/309"
  alt="Gif showing partial ragdoll"
%}

The core of the update logic is below. Looking back, the growing number of state checks and transitions is a clear case for a proper state machine, which is how I would structure it today.

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
The first version used a large number of raycasts uniformly distributed in a circle, which showed up as a noticeable FPS drop in Unreal's profiler. I replaced them with a much smaller number of sphere casts, after which the spikes were gone from the profiler.

{% include preload_img.html
  src="/assets/projects/ecocide/partial_windblast.gif"
  aspect_ratio="549/309"
  alt="Gif showing partial windblast"
%}

A further improvement would be a cone intersection test plus a single ray/sphere cast per enemy to check for strength or blockage.

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