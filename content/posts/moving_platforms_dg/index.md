---
title: "Creating Moving Platforms with Directional Gravity in Unreal Engine"
date: 2020-05-03
description: "Implementing moving platforms in my custom Unreal gravity system."
draft: false
---

## Why is directional gravity so difficult? 
In 9.8, one of the fundamental mechanics is the ability to walk around planets. This is not default behaviour in Unreal Engine, in which the built-in Character class has hard-coded vertical gravity. All the walking, jumping, crouching, swimming and falling movement code carries the assumption that the character's Capsule collision is vertical, and that the player is moving in the x-y plane. I will go into detail as to how I navigated through most of those issues, but there was still one flaw in my alterations.

When my player was on a rotating object, they would slide around on the surface, as though the object was made of ice:

{{< video src="BrokenBaseMovement">}}

## How the character interacts with moving objects in UE4
First, a bit about how UE4 deals with the character standing on a moving object.

The `ACharacter` Actor has, by default, an actor component called `UCharacterMovementComponent`. This class implements different movement modes: Walking, Falling, Swimming, Flying, etc, as well as networking. This component is responsible for moving the player based on a `Velocity` provided by the `PlayerController`. 

[//]: # (More details on the PlayerController input system in another post.)

In the `ACharacter` class, there is a property
```cpp
FBasedMovementInfo BasedMovement;
```
that stores information about the component the player is currently standing on. This is updated in the `PhysWalking(DeltaSeconds, Iterations)` method from the floor returned from the `FindFloor(...)` method. `PhysWalking(...)` calculates how the Capsule collision moves when walking, and is run multiple times per frame. This ensures that `BasedMovement` is up to date and can be used each frame.

Ok, so we know what object the character is standing on. How is this used to make sure we don't slide off moving platforms?

In the method `UpdateBasedMovement(DeltaSeconds)`{{< super >}}[1]({{< relref "moving_platforms_dg#Ref1">}}){{< /super >}} of UCharacterMovementComponent, UE4 will detect if the MovementBase primitive component has moved (translated or rotated) since the last frame. If it has, it will first rotate the character with the same delta (only in the z-axis, the yaw):
```cpp {lineNos=true}
TargetRotator.Pitch = 0.f; 
TargetRotator.Roll = 0.f; 
MoveUpdatedComponent(FVector::ZeroVector, TargetRotator, /*bSweep*/ false);
```
<a name="Code1"></a>
Then, it will attempt to translate the player along with the platform:

```cpp {lineNos=true}
const FVector BaseOffset(0.0f, 0.0f, HalfHeight);
const FVector LocalBasePos = OldLocalToWorld.InverseTransformPosition(UpdatedComponent->GetComponentLocation() - BaseOffset);
const FVector NewWorldPos = ConstrainLocationToPlane(NewLocalToWorld.TransformPosition(LocalBasePos) + BaseOffset);
DeltaPosition = ConstrainDirectionToPlane(NewWorldPos - UpdatedComponent->GetComponentLocation());
```

This dense little block transforms the position at the bottom of the player capsule (`UpdatedComponent->GetComponentLocation() - BaseOffset`) from world coordinates to the _old_ (previous frame) local space of the platform object. It then transforms that _local_ point into world space again using the current _local_ space of the platform object.  The difference between the new point and the character position is then found and applied to the character.

So, what's the problem with this approach? We have a comment from the dev team themselves to answer that:
```cpp {lineNos=true}
// hack - transforms between local and world space introducing slight error FIXMESTEVE - discuss with engine team: just skip the transforms if no rotation?
FVector BaseMoveDelta = NewBaseLocation - OldBaseLocation;
if (!bRotationChanged && (BaseMoveDelta.X == 0.f) && (BaseMoveDelta.Y == 0.f))
{
    DeltaPosition.X = 0.f;
    DeltaPosition.Y = 0.f;
}
```
It appears that this approach of converting between different coordinate systems is not particularly accurate, but for an upright player the sliding around can be nullified easily enough using this hack. For my directional gravity system though, this fix can't be used because the _x_ and _y_ translation is relative to the player, not the world.

To fix this, I decided to have another look at the MovementBase system.

`ACharacter.h` has a neat little namespace called `MovementBaseUtility`, for processing information about the current movement base. The ones we are interested in:

```cpp
GetMovementBaseTangentialVelocity(...) 
GetMovementBaseVelocity(...)
```

Both are only used in one method: `GetImpartedMovementBaseVelocity()`. This method, in turn, is only used once: in `OnMovementModeChanged(...)`, when the character transitions to falling mode. The combination of tangential velocity and linear velocity is applied on the moment of jumping/falling to simulate conservation of momentum.

## The maths of Rigid Body Rotation

It just so happened that this very week my physics class was studying rigid body rotation, and how to use angular velocity, radius, and tangential velocity for solving problems. For our needs, we just need to understand this small equation:

![VtMath](/moving_platforms_dg/vt_math.png)

Each of these values is a vector quantity, so the cross symbol is the vector cross product. The angular velocity vector is directed along the axis of rotation. The tangential velocity at this point is the output of this calculation.
To show how the vector quantities relate:

![TangentialVelocity](/moving_platforms_dg/TanVel.png)

And in 3D:
{{< video src="3DSphere1">}}

This behind-the-scenes maths was very useful, because it allowed me to understand how `GetMovementBaseTangentialVelocity(...)` calculates its value, most importantly that it will calculate the exact tangential velocity for any point on the surface on any rigid object{{< super >}}[2]({{< relref "moving_platforms_dg#Ref2">}}){{< /super >}}. This can be visualised by looking at a bounding sphere of any object you like, no matter how complex:

![NoodleSphere](/moving_platforms_dg/BoundingSphere.png)

Then, notice how the tangential velocity equation works for any point on the surface of the sphere. You can change the radius vector to point towards any position on the surface:

{{< video src="3DSphere2">}}

Extending this idea, the radius can also point to any position contained within the sphere. By definition of a bounding sphere, any point on the surface of the enclosed object is also within the sphere, and therefore the tangential velocity of any point on the surface of the rigid object can be calculated with this equation.

{{< video src="NoodleLeft">}}

The linear velocity function is more intuitive to understand. Because the object is rigid, all points on the surface of the object move in the same direction at the same speed. Adding the output of these two calculations will give the final velocity of a point on the surface of a rigid object.

## The Solution
So, the final code (modified from [here](#Code1)) to fix the sliding around:

```cpp {lineNos=true}
FVector TangentialVel = MovementBaseUtility::GetMovementBaseTangentialVelocity(MovementBase, CharacterOwner->GetBasedMovement().BoneName, UpdatedComponent->GetComponentLocation());
FVector LinearVel = MovementBaseUtility::GetMovementBaseVelocity(MovementBase, CharacterOwner->GetBasedMovement().BoneName);

const FVector BaseOffset = GetLocalAxisZ() * HalfHeight;
const FVector WorldBasePos = UpdatedComponent->GetComponentLocation() - BaseOffset;
const FVector MovedBasePos = WorldBasePos + TangentialVel * DeltaSeconds + LinearVel * DeltaSeconds;
const FVector NewWorldPos = MovedBasePos + BaseOffset;
```
This works perfectly:

{{< video src="FixedBaseMovement">}}

With this piece of the puzzle in place, I can be confident that UE4 has the capability to run this central component of the gameplay.

## Notes:
Source links only work if you have access to the [UnrealEngine Github](https://www.unrealengine.com/en-US/ue4-on-github).

1. {{< anchor "Ref1">}}A link to this [source block](https://github.com/EpicGames/UnrealEngine/blob/f8f4b403eb682ffc055613c7caf9d2ba5df7f319/Engine/Source/Runtime/Engine/Private/Components/CharacterMovementComponent.cpp#L1979).

2. {{< anchor "Ref2">}}Not simulated rigid body physics, just an object that doesn't bend or deform in any way. In UE4 and similar it means 'StaticMesh' not 'SkeletalMesh'.
