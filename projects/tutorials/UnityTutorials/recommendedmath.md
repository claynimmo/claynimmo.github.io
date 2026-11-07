---
layout: default
title: Recommended Math Knowledge for Game Programming
tags: [Unity, Math]
---

[Home](../../../../index.md) / [Tutorials](../index.md) /

## Overview

To effectively write code for video games using a Game Engine, there are several required math skills. This tutorial includes examples from the Unity Game Engine, but the concepts are universal.

## Coordinate Systems

Coordinate systems are the single most important concept for creating games. It is what determines where objects are, how they move, and how they interact. Every object encodes its global and local position, rotation, and scale, all represented as vectors. Vectors in Unity are encoded in the order X, Y, Z.

In Unity, the coordinate space sets the Z axis as forward, Y as up, and X as right. Euler rotation around an axis is clockwise when looking into the axis. An easy way to visualise this is by pointing your thumb into the axis, where the direction your fingers curl is positive rotation.

Rotation is not stored as Euler angles internally due to deadlock conditions. Unity instead uses quaternions. However, many game engines abstract quaternions and allow you to use Euler angles, so learning quaternions is not mandatory.

In Unity, rotation can be set using:

```csharp
transform.rotation = Quaternion.Euler(new Vector3(x, y, z));
```

In Unity, 1 unit is one metre. For example, the scale vector `(10,10,10)` represents a cube with 10-metre side lengths.

Many coordinate systems include both a local and global reference point:

- **Global:** True orientation relative to the scene's origin. The axis is the same regardless of rotation.
- **Local:** Orientation relative to the direct parent. The axes shift with the parent's rotation. For example, an object with a parent rotated 90 degrees may have its Z axis along the global Y axis. Grandparents only affect the local position of the parent, and not the child.

Local position, converted to world space at runtime, is better for static teleport positions such as initial spawn points, reducing maintenance.

## Unit Circle

The unit circle has high importance for geometrically placing objects at runtime. Very often, you would want to place objects randomly inside a circle. However, doing so based on random coordinates is heavily biased towards the centre. Instead, selecting a random radius provides better variance.

The most important properties of the unit circle are:

- Radius is 1.
- The coordinate at the edge of the circle for any given angle is $$ (\cos\theta,\sin\theta) $$.
- Proves the identity: $$ \cos^2\theta+\sin^2\theta=1 $$

Here is an example function returning as close as possible to a true random point within a circle, including a deadzone ring, adjusted to fit the terrain:

```csharp
Vector3 GetRandomSpawnPosition(){
    float randomAngleRadians = Mathf.Deg2Rad * Random.Range(0, 360);
    float randomRadius = Random.Range(donutSpawnRadius, donutSpawnRadius + donutInnerSpawnRadius);

    float xCoord = randomRadius * Mathf.Cos(randomAngleRadians);
    float zCoord = randomRadius * Mathf.Sin(randomAngleRadians);

    Vector3 rayPos = new Vector3(xCoord, 0, zCoord);
    rayPos += transform.position;
    Vector3 pos = rayPos;

    RaycastHit hit;
    if(Physics.Raycast(rayPos, -transform.up, out hit, 200)){
        pos = hit.point;
        pos = new Vector3(pos.x, pos.y + 1.5f, pos.z);
    }

    return pos;
}
```

# Vector Maths

Vectors are how you navigate the coordinate system, useful for movement, physics, direction vectors, surface normals, and more.

- **Magnitude:** The quantity pointing in the direction of a vector. For example, the velocity vector `(2,0,2)` has a magnitude of 2.8, pointing 45 degrees between the X and Z plane (forward-right).

- **Normalised vector:** A vector with a magnitude of 1. This is used for direction and input vectors. Normalising inputs is important to have consistent velocity. A fully pressed input is a 1 on that axis. With only one direction input, the magnitude is one. However, with two directions input, the magnitude becomes 1.4. This is why some older games have the player move faster diagonally.

- **Normal vector:** A vector pointing in the direction perpendicular to a plane. Surface normals point outward from a mesh face, which can be used to align objects to geometry, such as walking up slopes.

- **Dot product:** Calculates how much of one vector lies on another. It is useful for projecting vectors onto a plane, or calculating the angle between two vectors. Perpendicular vectors return 0, which can be used to determine if one vector is the normal to another. Dot product returns a scalar.

- **Cross product:** Computes the vector perpendicular to both inputs. This is the least useful, with its only major application being computing surface normals and back faces, which is often abstracted by the game engine.

Here are some useful applications of vector maths:

### Direction from A to B

```csharp
Vector3 dir = Vector3.Normalize(B.position - A.position);
```

### Horizontal Movement Relative to the Camera

```csharp
Vector3 cameraForward = Camera.main.transform.forward;
cameraForward.y = 0;
cameraForward.Normalize();

Vector3 cameraRight = Camera.main.transform.right;

targetVelocity = cameraRight * horizontal +
                 cameraForward * vertical;

targetVelocity.Normalize();
targetVelocity *= combinedSpeed;

Vector3 velocity = rbody.velocity;

Vector3 velocityChange = (targetVelocity - velocity);
velocityChange.y = 0;

rbody.AddForce(velocityChange, ForceMode.VelocityChange);
```

### Look Relative to Movement Input

(Combine with the previous code.)

```csharp
bool horizontalPressed = horizontal > deadZone || horizontal < -deadZone;

bool verticalPressed = vertical > deadZone || vertical < -deadZone;

targetAngle = Mathf.Atan2(horizontal, vertical) * Mathf.Rad2Deg + cam.eulerAngles.y;

float angle = Mathf.SmoothDampAngle(body.transform.eulerAngles.y, targetAngle, ref turnSmoothVelocity, turnSmoothTime);

if(updatedHorizontal != 0 || updatedVertical != 0) {
    body.transform.rotation = Quaternion.Euler(0, angle, 0);
}
```

# Time

Game programming is unique in how computation happens repeatedly inside frames. Physics (velocity, acceleration) are highly dependent on time to calculate the next value. However, framerate can vary significantly. Many game engines contain a delta time global variable that represents the difference in time between the current frame and the last.

Multiplying by delta time can be used to increase a value per second. For example, `x += Time.deltaTime;` in the main loop is equivalent to increasing `x` by 1 every second, regardless of framerate. This variable is used almost constantly for every dynamic object in the scene.

Time scale directly alters delta time, adjusting the time period for everything relying on the variable. For example, setting the time scale to `0.5` runs the game at half speed. Everything not coupled to delta time is not affected by time scale.

# Physics

Creating stylistic physics often relies on creating custom physics interactions instead of relying on the built-in Rigidbody systems. For example, accelerating physics objects with different masses by the same amount requires multiplying the applied force by the mass for it to cancel.

Knowledge of basic calculus (differentiation and integration) is required knowledge. Here are the most important physics formulas.

- F is force
- m is mass
- a is acceleration
- v is velocity
- d is distance
- g is gravity
- t is time

$$F = ma$$

$$F_{net} = \sum F$$

$$F_{normal} = mg\cos\theta$$, where $$\theta$$ is the angle between the incline and the ground plane.

$$F_{friction} = \mu F_{normal}$$, where $$\mu$$     is the coefficient of friction.

$$d = \int v(t)\,dt$$

$$v = \frac{d}{t}$$

$$v = \int a(t)\,dt$$

$$a = \frac{d}{dt}v(t)$$

$$a = \frac{F_{net}}{m}$$

The force of drag is dependent on the squared velocity, multiplied by the drag coefficient. Fluid density and frontal area can be assumed to cancel with the $$\frac{1}{2}$$ term to optimise game performance. However, it is generally better to let the built-in physics engine compute drag.