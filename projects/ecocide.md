---
layout: project
title: ECOCIDE
permalink: /ecocide.html
---
I believe that I did quite well during this project, of course I could have been better at commenting, the eternal struggle, but considering the non-documented, but as I remember it small number of bugs reported and changes requested regarding my code, I am happy with my work.

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "PhysicsEngine/PhysicalAnimationComponent.h"
#include "NiagaraFunctionLibrary.h"
#include "FMODBlueprintStatics.h"
#include "RagdollComponent.generated.h"

class UFMODEvent;

UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
class GP3_API URagdollComponent : public UPhysicalAnimationComponent
{
	GENERATED_BODY()

public:
	// Sets default values for this component's properties
	URagdollComponent();

protected:
	// Called when the game starts
	virtual void BeginPlay() override;

public:
	// Called every frame
	virtual void TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction) override;

	UFUNCTION(BlueprintCallable)
	void AddImpulse(FVector Impulse, FVector Location, FName BoneName, bool bVelChange);

	UFUNCTION(BlueprintCallable)
	void SetRagdoll(bool bRagdollChange);

	UFUNCTION(CallInEditor, BlueprintCallable)
	void SetPermanentRagdoll();

	UFUNCTION(CallInEditor, BlueprintCallable)
	void SetSemiRagdoll(bool bRagdollChange);

	UPROPERTY(EditAnywhere, BlueprintReadOnly)
	FName RootSocketName = FName();

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
	bool IsRagdolling = false;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
	bool IsPermaRagdoll = false;

	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	bool CanTakeDamage = false;

	UPROPERTY(EditAnywhere)
	UNiagaraSystem* BloodAsset;

	UPROPERTY(EditAnywhere)
	UFMODEvent* HitWallSoundEvent;

	UPROPERTY(EditAnywhere)
	UFMODEvent* HitEnemySoundEvent;

	FVector LastLastFrameVelocity;
	FVector LastFrameVelocity;

protected:
	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	float MinimumFlyingTime = 0.5f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	float EaseOutTime = 0.0f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	float RagdollRequiredForce = 1500.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	float RagdollTransitionMultiplier = 6.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	float ForceToDamageThreshold = 3000.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite)
	float ForceToDamageDivider = 40.f;

	class USkeletalMeshComponent*	   Mesh;
	class UPrimitiveComponent*		   Collider;
	class UCharacterMovementComponent* CharMovComp;
	class UPhysicsAsset*			   PhysAsset;

private:
	FVector	 OrgRelativeLocation;
	FRotator OrgRelativeRotation;

	TMap<FName, float>				BoneTimes;
	TMap<URagdollComponent*, float> HasHit;

	bool HasTakenDamage = false;

	bool IsSemiRagdoll = false;

	bool  PhysicsOn = false;
	bool  JustEnteredPhysics = false;
	float RequiredFlyingTimeLeft = 0.f;

	bool  ExitingRagdoll = false;
	float TimeLeftInRagdoll = -1.f;
	void  StartEaseOut();
	void  EaseOutRagdoll(float DeltaTime);

	void  CalcForces(float DeltaTime);
	void  CheckHitTimes(float DeltaTime);
	float CheckHeight();

	void EnterRagdoll(FVector Impulse);
	void ExitRagdoll();

	UFUNCTION()
	void OnHit(UPrimitiveComponent* HitComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit);
};
```
```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#include "RagdollComponent.h"
#include "Components/SkeletalMeshComponent.h"
#include "Components/PrimitiveComponent.h"
#include "Components/CapsuleComponent.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Math/NumericLimits.h"
#include "Kismet/KismetSystemLibrary.h"
#include "Kismet/GameplayStatics.h"
#include "NavigationSystem.h"
#include "NiagaraFunctionLibrary.h"
#include "DamageTypes.h"
#include "Particles/ParticleSystem.h"
#include "DrawDebugHelpers.h"

// Sets default values for this component's properties
URagdollComponent::URagdollComponent()
{
	// Set this component to be initialized when the game starts, and to be ticked every frame.  You can turn these features
	// off to improve performance if you don't need them.
	PrimaryComponentTick.bCanEverTick = true;
}

// Called when the game starts
void URagdollComponent::BeginPlay()
{
	Super::BeginPlay();

	Mesh = GetOwner()->FindComponentByClass<USkeletalMeshComponent>();
	Collider = Cast<UPrimitiveComponent>(GetOwner()->GetRootComponent());
	CharMovComp = GetOwner()->FindComponentByClass<UCharacterMovementComponent>();

	check(IsValid(Mesh));
	check(IsValid(Collider));
	check(IsValid(CharMovComp));

	SetSkeletalMeshComponent(Mesh);

	Collider->SetNotifyRigidBodyCollision(true);
	Mesh->OnComponentHit.AddDynamic(this, &URagdollComponent::OnHit);

	Collider->BodyInstance.bLockXRotation = true;
	Collider->BodyInstance.bLockYRotation = true;
	Collider->BodyInstance.CreateDOFLock();

	Collider->SetNotifyRigidBodyCollision(true);
	Collider->OnComponentHit.AddDynamic(this, &URagdollComponent::OnHit);

	OrgRelativeLocation = Mesh->GetRelativeLocation();
	OrgRelativeRotation = Mesh->GetRelativeRotation();
}

void URagdollComponent::AddImpulse(FVector Impulse, FVector Location, FName BoneName, bool bVelChange)
{
	if (BoneName == RootSocketName || BoneName == "None" || BoneName == "")
	{
		BoneName = RootSocketName;
	}
	FName OrgBoneName = BoneName;

	if (!IsRagdolling)
	{
		Collider->SetSimulatePhysics(true);
		PhysicsOn = true;
		JustEnteredPhysics = true;
		RequiredFlyingTimeLeft = MinimumFlyingTime;

		Collider->AddImpulse(Impulse, FName(), bVelChange);
		Collider->AddImpulse(FVector(0.f, 0.f, Impulse.Length() / 100.f), FName(), true);

		if (BoneName == RootSocketName)
		{
			return;
		}

		while (true)
		{
			FName Parent = Mesh->GetParentBone(BoneName);
			if (Parent == RootSocketName || Parent == "None" || Parent == "")
			{
				break;
			}

			BoneName = Parent;
		}

		float& BoneTime = BoneTimes.FindOrAdd(BoneName, 1.0f);
		if (BoneTime < 1.0f)
		{
			BoneTime = 1.0f;
		}

		Mesh->SetAllBodiesBelowSimulatePhysics(BoneName, true, true);
		ApplyPhysicalAnimationProfileBelow(BoneName, "Wind", true, false);
	}

	Mesh->AddImpulse(Impulse, OrgBoneName, bVelChange);
}

// Called every frame
void URagdollComponent::TickComponent(float DeltaTime, ELevelTick TickType, FActorComponentTickFunction* ThisTickFunction)
{
	Super::TickComponent(DeltaTime, TickType, ThisTickFunction);

	// Get Largest current velocity.
	FVector Velocity = Collider->GetComponentVelocity();
	float	VelocityLength = Velocity.Length();
	{
		FVector MeshVelocity = Mesh->GetComponentVelocity();
		float	MeshVelocityLength = MeshVelocity.Length();
		if (MeshVelocityLength > VelocityLength)
		{
			Velocity = MeshVelocity;
			VelocityLength = MeshVelocityLength;
		}
	}

	float DistanceFromGround = CheckHeight();

	CalcForces(DeltaTime);
	CheckHitTimes(DeltaTime);

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

	LastLastFrameVelocity = LastFrameVelocity;
	LastFrameVelocity = Velocity;
}

// Outward facing setragdoll function.
void URagdollComponent::SetRagdoll(bool bRagdollChange)
{
	if (bRagdollChange)
	{
		EnterRagdoll(LastFrameVelocity);
	}
	else
	{
		ExitingRagdoll = false;
		TimeLeftInRagdoll = -1.f;
		ExitRagdoll();
	}
}

void URagdollComponent::SetPermanentRagdoll()
{
	CanTakeDamage = false;
	IsPermaRagdoll = true;
	Mesh->SetNotifyRigidBodyCollision(false);
	Collider->SetNotifyRigidBodyCollision(false);
	SetComponentTickEnabled(false);
	if (!IsRagdolling)
	{
		EnterRagdoll(LastFrameVelocity);
	}
}

void URagdollComponent::SetSemiRagdoll(bool bRagdollChange)
{
	// Budget exit ragdoll
	if (bRagdollChange)
	{
		Mesh->bPauseAnims = false;
		CharMovComp->SetMovementMode(EMovementMode::MOVE_Walking);
		Mesh->SetAllBodiesSimulatePhysics(false);
		Mesh->SetAllBodiesPhysicsBlendWeight(0.0, false);

		FAttachmentTransformRules Rules = FAttachmentTransformRules::SnapToTargetIncludingScale;
		Mesh->AttachToComponent(GetOwner()->GetRootComponent(), Rules, FName("None"));
		Mesh->SetRelativeLocationAndRotation(OrgRelativeLocation, OrgRelativeRotation, false, nullptr, ETeleportType::None);
		GetOwner()->SetActorRotation(FRotator(0.f, 0.f, 0.f));
	}

	IsSemiRagdoll = bRagdollChange;
	SetComponentTickEnabled(!bRagdollChange);
	IsRagdolling = bRagdollChange;
	Mesh->SetAllBodiesBelowSimulatePhysics(RootSocketName, bRagdollChange, false);
	CanTakeDamage = false;

	if (bRagdollChange)
	{
		Collider->SetSimulatePhysics(false);
		PhysicsOn = false;
		Collider->SetCollisionProfileName("OverlapAll");
		Mesh->SetAllBodiesBelowPhysicsBlendWeight(RootSocketName, 0.2f, false, false);
		ApplyPhysicalAnimationProfileBelow(RootSocketName, "", false, false);
	}
	else
	{
		Collider->SetCollisionProfileName("Pawn");
		Mesh->SetAllBodiesBelowPhysicsBlendWeight(RootSocketName, 0.0, false, false);
		ApplyPhysicalAnimationProfileBelow(RootSocketName, "", false, false);
	}
}

void URagdollComponent::CalcForces(float DeltaTime)
{
	// if ragdolling don't modify bones
	if (IsRagdolling)
	{
		return;
	}

	TArray<FName> BonesToRemove;

	// Update bone simulation.
	for (TPair<FName, float>& BoneTime : BoneTimes)
	{
		BoneTime.Value -= DeltaTime;
		if (BoneTime.Value <= 0.f)
		{
			Mesh->SetAllBodiesBelowSimulatePhysics(BoneTime.Key, false, true);
			Mesh->SetAllBodiesBelowPhysicsBlendWeight(BoneTime.Key, 0.0, false, true);
			ApplyPhysicalAnimationProfileBelow(BoneTime.Key, "", true, false);
			BonesToRemove.Add(BoneTime.Key);
			continue;
		}

		if (BoneTime.Value < 1.0f)
		{
			Mesh->SetAllBodiesBelowPhysicsBlendWeight(BoneTime.Key, BoneTime.Value, false, true);
			continue;
		}

		Mesh->SetAllBodiesBelowPhysicsBlendWeight(BoneTime.Key, 1.0f, false, true);
	}

	for (FName Bone : BonesToRemove)
	{
		BoneTimes.Remove(Bone);
	}
}

void URagdollComponent::CheckHitTimes(float DeltaTime)
{
	TArray<URagdollComponent*> ToRemove;

	for (TPair<URagdollComponent*, float>& HitTime : HasHit)
	{
		HitTime.Value -= DeltaTime;

		if (HitTime.Value <= 0.f)
		{
			ToRemove.Add(HitTime.Key);
		}
	}

	for (URagdollComponent* Key : ToRemove)
	{
		HasHit.Remove(Key);
	}
}

void URagdollComponent::EnterRagdoll(FVector Impulse)
{
	HasTakenDamage = false;

	Collider->SetSimulatePhysics(false);
	PhysicsOn = false;

	Mesh->bPauseAnims = true;

	IsRagdolling = true;
	CharMovComp->SetMovementMode(EMovementMode::MOVE_None, 0);
	Collider->SetCollisionProfileName("OverlapAll");
	Mesh->SetAllBodiesSimulatePhysics(true);
	Mesh->SetAllBodiesPhysicsBlendWeight(1.0, false);
	ApplyPhysicalAnimationProfileBelow(RootSocketName, "dead", true, false);

	Mesh->AddImpulse(Impulse, RootSocketName, true);
}

void URagdollComponent::StartEaseOut()
{
	ExitingRagdoll = true;
	TimeLeftInRagdoll = EaseOutTime;
}

void URagdollComponent::EaseOutRagdoll(float DeltaTime)
{
	TimeLeftInRagdoll -= DeltaTime;

	Mesh->SetAllBodiesPhysicsBlendWeight(TimeLeftInRagdoll / EaseOutTime, false);

	if (TimeLeftInRagdoll <= 0.f)
	{
		ExitingRagdoll = false;
		ExitRagdoll();
	}
}

void URagdollComponent::ExitRagdoll()
{
	CanTakeDamage = false;
	Mesh->bPauseAnims = false;
	IsRagdolling = false;
	CharMovComp->SetMovementMode(EMovementMode::MOVE_Walking);
	Collider->SetCollisionProfileName("Pawn");
	Mesh->SetAllBodiesSimulatePhysics(false);
	Mesh->SetAllBodiesPhysicsBlendWeight(0.0, false);
	ApplyPhysicalAnimationProfileBelow(RootSocketName, "", true, false);

	FAttachmentTransformRules Rules = FAttachmentTransformRules::SnapToTargetIncludingScale;
	Mesh->AttachToComponent(GetOwner()->GetRootComponent(), Rules, FName("None"));
	Mesh->SetRelativeLocationAndRotation(OrgRelativeLocation, OrgRelativeRotation, false, nullptr, ETeleportType::None);
	GetOwner()->SetActorRotation(FRotator(0.f, 0.f, 0.f));

	FNavLocation		NavLoc;
	FVector				QueryingExtent = FVector(250.0f, 250.0f, 250.0f);
	FNavAgentProperties NavAgentProps;
	NavAgentProps.AgentHeight = Cast<UCapsuleComponent>(Collider)->GetScaledCapsuleHalfHeight() * 4.f;
	NavAgentProps.AgentRadius = Cast<UCapsuleComponent>(Collider)->GetScaledCapsuleRadius() * 2.f;
	NavAgentProps.AgentStepHeight = -1.f;

	// Set you NavAgentProps properties here (radius, height, etc)
	bool bProjectedLocationValid = Cast<UNavigationSystemV1>(GetWorld()->GetNavigationSystem())->ProjectPointToNavigation(GetOwner()->GetActorLocation(), NavLoc, QueryingExtent);
	if (bProjectedLocationValid)
	{
		FVector NewLocation = NavLoc.Location;
		NewLocation.Z += Cast<UCapsuleComponent>(Collider)->GetScaledCapsuleHalfHeight();
		GetOwner()->SetActorLocation(NewLocation, false, nullptr, ETeleportType::None);
	}
	else
	{
		UGameplayStatics::ApplyDamage(GetOwner(), 9999999.f, nullptr, GetOwner(), nullptr);
	}

	BoneTimes = TMap<FName, float>();
}

float URagdollComponent::CheckHeight()
{
	FVector BoxExtent;
	FVector Origin;
	GetOwner()->GetActorBounds(false, Origin, BoxExtent);

	FVector Start;
	FVector End;
	FVector Feet;
	if (IsRagdolling)
	{
		Start = Feet = Mesh->GetComponentLocation();
		Start.Z += BoxExtent.Z;

		End = Start;
		End.Z -= BoxExtent.Z * 4.f;
	}
	else
	{
		Start = Feet = GetOwner()->GetActorLocation();
		Start.Z += BoxExtent.Z;
		Feet.Z -= BoxExtent.Z;

		End = Start;
		End.Z -= BoxExtent.Z * 4.f;
	}

	TArray<AActor*> ActorsToIgnore;
	ActorsToIgnore.Add(GetOwner());

	FHitResult Hit;

	if (UKismetSystemLibrary::LineTraceSingleByProfile(
			GetWorld(),
			Start,
			End,
			FName("BlockAll"),
			false,
			ActorsToIgnore,
			EDrawDebugTrace::None,
			Hit,
			true,
			FLinearColor::Red,
			FLinearColor::Green,
			5.f))
	{

		return Feet.Z - Hit.Location.Z;
	}
	else
	{
		return TNumericLimits<float>::Max();
	}
}

void URagdollComponent::OnHit(UPrimitiveComponent* HitComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
{
	if ((!PhysicsOn && !IsRagdolling && !CanTakeDamage) || OtherActor == GetOwner())
		return;

	if (OtherActor == nullptr || OtherActor->IsActorBeingDestroyed())
		return;

	// Check for the largest velocity length.
	FVector Velocity = LastLastFrameVelocity;
	if (LastFrameVelocity.Length() < Velocity.Length())
	{
		Velocity = LastFrameVelocity;
	}
	{
		FVector MeshVelocity = Mesh->GetComponentVelocity();

		FVector CurrentVelocity = Collider->GetComponentVelocity();
		if (MeshVelocity.Length() > CurrentVelocity.Length())
		{
			CurrentVelocity = MeshVelocity;
		}

		if (CurrentVelocity.Length() > Velocity.Length())
		{
			Velocity = CurrentVelocity;
		}
	}

	URagdollComponent* OtherRagdoll = OtherActor->FindComponentByClass<URagdollComponent>();
	if (OtherRagdoll != nullptr)
	{
		if (!HasHit.Contains(OtherRagdoll) && !OtherRagdoll->HasHit.Contains(this) && !IsSemiRagdoll && !OtherRagdoll->IsSemiRagdoll)
		{
			OtherRagdoll->AddImpulse(Velocity / 2.f, FVector(), Hit.BoneName, true);
			AddImpulse(-(Velocity / 2.f), FVector(), Hit.MyBoneName, true);

			HasHit.Add(OtherRagdoll, 0.1f);
			OtherRagdoll->HasHit.Add(this, 0.1f);
			if (HitEnemySoundEvent != nullptr)
			{
				UFMODBlueprintStatics::PlayEventAttached(HitEnemySoundEvent, Collider, NAME_None, GetOwner()->GetActorLocation(), EAttachLocation::KeepWorldPosition, false, true, true);
			}
		}

		if (!CanTakeDamage)
			return;

		if (Velocity.Length() > 650)
		{
			UGameplayStatics::ApplyDamage(OtherActor, 999999.f, nullptr, GetOwner(), UExplodeDamage::StaticClass());
			UGameplayStatics::ApplyDamage(GetOwner(), 999999.f, nullptr, OtherActor, UExplodeDamage::StaticClass());

			UNiagaraFunctionLibrary::SpawnSystemAtLocation(GetWorld(), BloodAsset, GetOwner()->GetActorLocation(), FRotator::ZeroRotator, FVector(1),
				true, true, ENCPoolMethod::AutoRelease);
			UNiagaraFunctionLibrary::SpawnSystemAtLocation(GetWorld(), BloodAsset, OtherActor->GetActorLocation(), FRotator::ZeroRotator, FVector(1),
				true, true, ENCPoolMethod::AutoRelease);

			OtherRagdoll->GetOwner()->Destroy();
			GetOwner()->Destroy();
		}
	}
	else
	{ // Wall
		if (NormalImpulse.Length() > ForceToDamageThreshold && !HasTakenDamage && !IsSemiRagdoll && IsRagdolling)
		{
			HasTakenDamage = true;
			UGameplayStatics::ApplyDamage(GetOwner(), 1.f, nullptr, OtherActor, UThrowDamage::StaticClass());
			if (HitWallSoundEvent != nullptr)
			{
				UFMODBlueprintStatics::PlayEventAttached(HitWallSoundEvent, Collider, NAME_None, GetOwner()->GetActorLocation(), EAttachLocation::KeepWorldPosition, true, true, true);
			}
		}
	}
}
```

Of course, there are many changes I would like to make if I had more time for the same amount of work, one of which would be that the ragdoll component would be easier to manage if partially implemented using a state machine RagdollComponent.cpp, RagdollComponent.h, especially considering that most of the functionality of the ragdoll component is switching which bones are simulated and which are animated at what times, in this case I ended up using Boolean checks that run every tick and also when requested to change states.

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "Components/ActorComponent.h"
#include "WindPushComponent.generated.h"

UCLASS(ClassGroup = (Custom), meta = (BlueprintSpawnableComponent))
class GP3_API UWindPushComponent : public UActorComponent
{
	GENERATED_BODY()

public:
	// Sets default values for this component's properties
	UWindPushComponent();

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush|Force")
	float Force = 10000.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush|Force")
	float EndingForce = 6000.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush|Force", meta = (ClampMin = "0.0", ClampMax = "1.0", UIMin = "0.0", UIMax = "1.0"))
	float CarryThroughPercent = 0.5f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush")
	float MaxRange = 6000.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush", meta = (ClampMin = "0.1"))
	float StartingRadius = 60.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush", meta = (ClampMin = "0.2"))
	float EndingRadius = 350.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush", meta = (ClampMin = "2"))
	int TraceAmount = 250;

	// UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush|Collision")
	// bool VelocityChange = true;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush|Collision")
	FName OwnerTag = FName("Player");

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush|Collision")
	TSet<FName> IgnoreTags;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "WindPush|Collision")
	FName CollisionPreset = FName("WindPush");

	UPROPERTY(EditAnywhere, Category = "WindPush|Debug")
	bool DrawDebugLines = false;

	UFUNCTION(BlueprintCallable, CallInEditor, Category = "WindPush|Debug")
	void UseWindPush();

private:
	// https://stackoverflow.com/questions/28567166/uniformly-distribute-x-points-inside-a-circle
	TArray<FVector2f> Sunflower(int N, float Alpha = 0.f, bool Geodesic = false);
};
```


```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#include "WindPushComponent.h"
#include "DrawDebugHelpers.h"
#include "Kismet/KismetSystemLibrary.h"
#include "../RagdollComponent.h"
#include "GameFramework/CharacterMovementComponent.h"
#include "Kismet/GameplayStatics.h"
#include "GameFramework/DamageType.h"

// Sets default values for this component's properties
UWindPushComponent::UWindPushComponent()
{
	// Set this component to be initialized when the game starts, and to be ticked every frame.  You can turn these features
	// off to improve performance if you don't need them.
	PrimaryComponentTick.bCanEverTick = false;
}

void UWindPushComponent::UseWindPush()
{
	TArray<FVector2f> Points = Sunflower(TraceAmount, 2.f, true);

	FVector	 Location;
	FRotator Rotation;
	GetOwner()->GetActorEyesViewPoint(Location, Rotation);

	FVector ForwardVector = Rotation.Vector();
	FVector RightVector = GetOwner()->GetActorRightVector();
	FVector UpVector = FVector::CrossProduct(ForwardVector, RightVector);

	TSet<UPrimitiveComponent*> HasHit;
	for (auto Point : Points)
	{
		FVector TransformedPoint = (Point.X * RightVector) + (Point.Y * UpVector);
		FVector Start = Location + (TransformedPoint * StartingRadius);
		FVector End = Location + (TransformedPoint * EndingRadius + (ForwardVector * MaxRange));

		TArray<AActor*> ActorsToIgnore;
		ActorsToIgnore.Add(GetOwner());

		EDrawDebugTrace::Type DebugTraceType = EDrawDebugTrace::None;
		if (DrawDebugLines)
		{
			DebugTraceType = EDrawDebugTrace::ForDuration;
		}

		TArray<FHitResult> Hits;
		UKismetSystemLibrary::SphereTraceMultiByProfile(GetWorld(), Start, End, 5.f /* Radius */,
			CollisionPreset, true /* TraceComplexMesh */, ActorsToIgnore, DebugTraceType, Hits, true /* IgnoreSelf */,
			FLinearColor::Red /* TraceColor */, FLinearColor::Green /* TraceColorHit */, 5.0f /* DrawTime */);

		float TraceForce = Force / float(TraceAmount);
		float EndingTraceForce = EndingForce / float(TraceAmount);
		float CarryThrough = 1.0f;
		for (auto Hit : Hits)
		{
			if (Hit.bBlockingHit)
				break;

			auto Component = Hit.GetComponent();

			bool ShouldContinue = false;
			for (auto Tag : IgnoreTags)
			{
				if (Component->ComponentHasTag(Tag))
				{
					ShouldContinue = true;
					break;
				}
			}
			if (ShouldContinue)
				continue;

			float RealTraceForce = TraceForce * CarryThrough;
			if (EndingTraceForce != TraceForce)
			{
				float Min = EndingTraceForce / TraceForce;
				float Max = 1.0f;
				float Alpha = Hit.Distance / MaxRange;
				RealTraceForce *= FMath::Lerp(Min, Max, Alpha);
			}

			URagdollComponent* RagdollComp = Cast<URagdollComponent>(Component->GetOwner()->GetComponentByClass(URagdollComponent::StaticClass()));
			if (Component->IsSimulatingPhysics())
			{
				Component->AddVelocityChangeImpulseAtLocation(ForwardVector * RealTraceForce, Hit.Location, Hit.BoneName);
			}
			else if (RagdollComp)
			{
				if (!HasHit.Contains(Component))
				{
					HasHit.Add(Component);
					UGameplayStatics::ApplyDamage(Component->GetOwner(), 1.f, nullptr, GetOwner(), UDamageType::StaticClass());
				}
				RagdollComp->AddImpulse(ForwardVector * RealTraceForce, Hit.Location, Hit.BoneName, true);
				break;
			}

			CarryThrough -= CarryThroughPercent;
		}
	}
}

// https://stackoverflow.com/questions/28567166/uniformly-distribute-x-points-inside-a-circle
TArray<FVector2f> UWindPushComponent::Sunflower(int N, float Alpha, bool Geodesic)
{
	const float Phi = (1 + FMath::Sqrt(5.f)) / 2.f; // golden ratio
	const float AngleStride = 360.f * Phi;

	auto Radius = [](float K, float N, float B) {
		return K > N - B ? 1 : FMath::Sqrt(K - 0.5f) / FMath::Sqrt(N - (B + 1.f) / 2.f);
	};

	int B = FMath::FloorToInt(Alpha * FMath::Sqrt(float(N))); // Number of boundary points

	TArray<FVector2f> Points;

	for (int K = 0; K < N; K++)
	{
		float R = Radius(K, N, B);
		float Theta = Geodesic ? K * 360.f * Phi : K * AngleStride;
		float X = !FMath::IsNaN(R * FMath::Cos(Theta)) ? R * FMath::Cos(Theta) : 0.f;
		float Y = !FMath::IsNaN(R * FMath::Sin(Theta)) ? R * FMath::Sin(Theta) : 0.f;
		Points.Add(FVector2f(X, Y));
	}

	return Points;
}
```

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#pragma once

#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "GP3/MainAttack.h"
#include "SpikeWall.generated.h"

class UFMODEvent;

struct BodyStorage
{
	TMap<FName, class UPhysicsHandleComponent*> Bones;
	float										TimeLeft;
};

UCLASS()
class GP3_API ASpikeWall : public AActor
{
	GENERATED_BODY()

public:
	// Sets default values for this actor's properties
	ASpikeWall();

	UPROPERTY(EditAnywhere, BlueprintReadOnly)
	class UBoxComponent* DefaultRoot;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "SpikeWall|Gameplay")
	int MaxBodies = -1;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "SpikeWall|Gameplay")
	float BodyTime = -1.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "SpikeWall|Gameplay")
	float HealthPerBody = 10.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "SpikeWall|Gameplay")
	bool HasToBeFullyRagdoll = true;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "SpikeWall|Physics")
	float AngularDamping = 5000.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "SpikeWall|Physics")
	float AngularStiffness = 7500.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "SpikeWall|Physics")
	float InterpolationSpeed = 500.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "SpikeWall|Physics")
	float LinearDamping = 7500.f;

	UPROPERTY(EditAnywhere, BlueprintReadWrite, Category = "SpikeWall|Physics")
	float LinearStiffness = 7500.f;

	UPROPERTY(EditAnywhere)
	class UNiagaraSystem* BloodAsset;

	UPROPERTY(EditAnywhere)
	UFMODEvent* EnemyImpaledSoundEvent;

	UPROPERTY(EditAnywhere)
	UFMODEvent* EnemyBleedingSoundEvent;

	UPROPERTY()
	AMainAttack* MainAttack;

protected:
	// Called when the game starts or when spawned
	virtual void BeginPlay() override;

public:
	// Called every frame
	virtual void Tick(float DeltaTime) override;

	UFUNCTION(CallInEditor, BlueprintCallable)
	void Reset();

	UFUNCTION()
	void OnHit(UPrimitiveComponent* HitComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit);

	void KillActor(class USkeletalMeshComponent* Mesh);

private:
	TMap<class USkeletalMeshComponent*, BodyStorage> Held;
};
```

```cpp
// Fill out your copyright notice in the Description page of Project Settings.

#include "SpikeWall.h"
#include "Components/BoxComponent.h"
#include "../RagdollComponent.h"
#include "Components/SkeletalMeshComponent.h"
#include "PhysicsEngine/PhysicsHandleComponent.h"
#include "UObject/UObjectGlobals.h"
#include "../GP3GameMode.h"
#include "Kismet/GameplayStatics.h"
#include "../DamageTypes.h"
#include "GP3/MainAttack.h"
#include "../Grabbable.h"
#include "Particles/ParticleSystem.h"
#include "FMODBlueprintStatics.h"
#include "NiagaraFunctionLibrary.h"

// Sets default values
ASpikeWall::ASpikeWall()
{
	// Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
	PrimaryActorTick.bCanEverTick = true;

	UBoxComponent* Collider = CreateDefaultSubobject<UBoxComponent>(TEXT("BoxCollider"));
	DefaultRoot = Collider;
	RootComponent = DefaultRoot;

	Collider->SetCollisionProfileName(TEXT("BlockAll"));
	Collider->SetNotifyRigidBodyCollision(true);

	Collider->OnComponentHit.AddDynamic(this, &ASpikeWall::OnHit);
}

// Called when the game starts or when spawned
void ASpikeWall::BeginPlay()
{
	Super::BeginPlay();
	TArray<AActor*> FoundActors;
	UGameplayStatics::GetAllActorsOfClass(GetWorld(), AMainAttack::StaticClass(), FoundActors);
	if (FoundActors.Num() > 0)
	{
		if (AMainAttack* PlayerCharacter = Cast<AMainAttack>(FoundActors[0]))
			MainAttack = PlayerCharacter;
	}
}

// Called every frame
void ASpikeWall::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);

	TArray<USkeletalMeshComponent*> ToRemove;

	for (TPair<USkeletalMeshComponent*, BodyStorage>& Outer : Held)
	{
		// if (BodyTime < 0.f)
		// {
		// 	return;
		// }

		Outer.Value.TimeLeft -= DeltaTime;

		if (Outer.Key == NULL || Outer.Key->GetOwner()->IsActorBeingDestroyed())
		{
			for (TPair<FName, UPhysicsHandleComponent*>& Bones : Outer.Value.Bones)
			{
				if (Bones.Value != NULL)
				{
					Bones.Value->DestroyComponent();
				}
			}

			ToRemove.Add(Outer.Key);
		}
	}

	for (USkeletalMeshComponent* Key : ToRemove)
	{
		// KillActor(Key);
		Held.Remove(Key);
	}
}

void ASpikeWall::Reset()
{
	for (TPair<USkeletalMeshComponent*, BodyStorage>& Outer : Held)
	{
		for (TPair<FName, UPhysicsHandleComponent*>& Bones : Outer.Value.Bones)
		{
			if (Bones.Value != NULL)
			{
				Bones.Value->DestroyComponent();
			}
		}

		KillActor(Outer.Key);
	}

	Held = TMap<USkeletalMeshComponent*, BodyStorage>();
}

void ASpikeWall::KillActor(class USkeletalMeshComponent* Mesh)
{
	if (Mesh != NULL)
	{
		Mesh->GetOwner()->Destroy();
		AGP3GameMode* GameMode = Cast<AGP3GameMode>(GetWorld()->GetAuthGameMode());
		if (IsValid(GameMode))
		{
			if (GEngine)
				GEngine->AddOnScreenDebugMessage(-1, 15.0f, FColor::Yellow, TEXT("ABOAWDS!"));
			GameMode->TreeHeal(HealthPerBody);
		}
	}
}

void ASpikeWall::OnHit(UPrimitiveComponent* HitComponent, AActor* OtherActor, UPrimitiveComponent* OtherComp, FVector NormalImpulse, const FHitResult& Hit)
{
	URagdollComponent* Ragdoll = OtherActor->FindComponentByClass<URagdollComponent>();
	if (Ragdoll)
	{
		if (MainAttack->GrabbedItem == OtherActor)
		{
			MainAttack->GrabbedItem->FindComponentByClass<UGrabbable>()->EndGrab();
			MainAttack->GrabbedItem = nullptr;
		}
		USkeletalMeshComponent* Mesh = Cast<USkeletalMeshComponent>(OtherComp);

		// Check if too many are already on the spikewall.
		if (!Held.Contains(Mesh))
		{
			if (MaxBodies >= 0 && Held.Num() >= MaxBodies)
			{
				return;
			}

			if (HasToBeFullyRagdoll)
			{
				if (!Ragdoll->IsRagdolling)
				{
					return;
				}
			}
			UGameplayStatics::ApplyDamage(OtherActor, 99999.f, nullptr, this, USpikeDamage::StaticClass());
			AGP3GameMode* GameMode = Cast<AGP3GameMode>(GetWorld()->GetAuthGameMode());
			if (IsValid(GameMode))
			{
				GameMode->TreeHeal(HealthPerBody);
			}

			BodyStorage TMPStorage;
			TMPStorage.Bones = TMap<FName, UPhysicsHandleComponent*>();
			TMPStorage.TimeLeft = BodyTime;

			UNiagaraFunctionLibrary::SpawnSystemAtLocation(GetWorld(), BloodAsset, Hit.Location, Hit.ImpactNormal.Rotation(), FVector(1),
				true, true, ENCPoolMethod::AutoRelease);

			Held.Add(Mesh, TMPStorage);

			if (EnemyImpaledSoundEvent != nullptr)
			{
				UFMODBlueprintStatics::PlayEventAttached(EnemyImpaledSoundEvent, GetRootComponent(), NAME_None, OtherActor->GetActorLocation(), EAttachLocation::KeepWorldPosition, true, true, true);
			}

			if (EnemyImpaledSoundEvent != nullptr)
			{
				UFMODBlueprintStatics::PlayEventAttached(EnemyBleedingSoundEvent, OtherActor->GetRootComponent(), NAME_None, OtherActor->GetActorLocation(), EAttachLocation::KeepWorldPosition, true, true, true);
			}
		}
		BodyStorage& Storage = Held[Mesh];


		Ragdoll->SetPermanentRagdoll();
		if (Hit.BoneName == "" || Hit.BoneName == "None")
		{
			return;
		}

		if (Storage.Bones.Contains(Hit.BoneName))
		{
			return;
		}

		FVector	 Loc;
		FRotator Rot;
		Mesh->GetSocketWorldLocationAndRotation(Hit.BoneName, Loc, Rot);

		// Create a unique name for the physics handle component.
		FString HandleName = FString::Printf(TEXT("Handle_%d_"), OtherActor->GetUniqueID());
		HandleName += Hit.BoneName.ToString();

		UPhysicsHandleComponent* Handle = NewObject<UPhysicsHandleComponent>(this, UPhysicsHandleComponent::StaticClass(), *HandleName);
		Handle->RegisterComponent();
		AddInstanceComponent(Handle);

		Handle->GrabComponentAtLocationWithRotation(Mesh, Hit.BoneName, Hit.Location, Rot);
		Handle->SetAngularDamping(AngularDamping);
		Handle->SetAngularStiffness(AngularStiffness);
		Handle->SetInterpolationSpeed(InterpolationSpeed);
		Handle->SetLinearDamping(LinearDamping);
		Handle->SetLinearStiffness(LinearStiffness);

		Storage.Bones.Add(Hit.BoneName, Handle);
	}
}
```

I would also have liked to change the wind push ability to use a dynamically generated cone rather than a bunch of sphere traces, entirely for performance reasons, although I of course did check using the in built profiler if the current method causes any lag spikes before leaving it as it is. WindPushComponent.cpp, WindPushComponent.h.

I of course also have a lot to improve on when it comes to programming ability, as can be seen in the time keeping data, in which I spent over 30 hours on the ragdolls, purely writing code, that does not include research or checking things in unreal engine. But as that is more of an experience problem, it is not particularly important to mention.