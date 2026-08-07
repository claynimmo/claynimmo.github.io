---
layout: default
title: Portfolio | Subatomic
---

[Home](../../../index.md) / [Game Development](../index.md) / [Games](games.md) /

# Subatomic

Subatomic is fully 3D team based battle royal game against AI enemies, with the mechanics taking inspiration from quantum theory. This game was developed specifically to maintain strong programming practices, and to create custom enemy AI for the first time. The source code of the project is made [available](https://github.com/claynimmo/Source-Code-for-Atomic).

The game was published on May 27th, 2024.

<iframe frameborder="0" src="https://itch.io/embed/2734025" width="552" height="167"><a href="https://nimclay.itch.io/subatomic">Subatomic by nimclay</a></iframe>

<iframe width="560" height="315" src="https://www.youtube.com/embed/QMv9lFlxzvU?si=9ECP9qmeqiSN56Hx" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe>

## Movement System

With the game set at a subatomic scale, the movement system needed to reflect this, giving fully 3D flight controls to the user (since particles have no ground to stand on). 

The movement was achieved by simply applying force to the player relative to the camera's look rotation, separating the vertical and horizontal movement by different speeds. Splitting the speed was required to limit vertical movement, otherwise the player was too sensitive to the camera making simply looking around to target an enemy unintuitive. However, since the vertical movement was made much slower, additional keybinds specifically for moving up and down was added, allowing a much less restrictive movement, especially through allowing moving vertical whilst looking forward.

To give the smooth, low friction feel to the movement, this was done by dynamically adjusting the rigidbody's drag property with respect to its velocity to limit speed.

``` C#
// get the inputs
horizontal = Input.GetAxis("Horizontal");
vertical = Input.GetAxis("Vertical");

// get the movement direction
forward = cameraTransform.forward;
forward = Vector3.Normalize(forward);
Vector3 moveDirection = (horizontal * right + vertical * forward) * speed;

// perform the movement
rbody.AddForce(moveDirection*rbody.mass+transform.up*space*speed);
rbody.drag = rbody.velocity.magnitude/maxdragvelocity;
```

## AI Controls

The AI for the game was constructed based on the player model, taking the same movement, attack, and ability functions where the AI would inject its own inputs. The movement was achieved by taking a random direction, biased from the direction to its target, then move along it with a further randomized speed. This process is described in the below code snippets: 
```C#
void GetForwardDirection(){
    movementMan.forward = RandomForward();
    movementMan.randomSpeed = Random.Range(0.3f,1);
}

Vector3 RandomForward(){
    float x = 0;
    float y = 0;
    float z = 0;
    //make the AI more likely to move in the opposite direction so it does not tend to move in only one direction
    if(movementMan.forward.x>0)
        x = Random.Range(-10,5);
    else
        x = Random.Range(-5,10);
    if(movementMan.forward.z>0)
        z = Random.Range(-10,5);
    else
        z = Random.Range(-5,10);
    if(movementMan.forward.y>0)
        y = Random.Range(-10,5);
    else
        y = Random.Range(-5,10);
    Vector3 dir = new Vector3(x,y,z);
    Vector3 bias = Vector3.zero;
    dir = Vector3.Normalize(dir);
    if(movementMan.weapon.publicenemynumber1!=null){
        //get direction of target relative to this object
        Vector3 targetDirRel = Vector3.Normalize(movementMan.weapon.publicenemynumber1.transform.position - this.transform.position);
        //get the direction between the current random forward and the target position through vector addition
        bias = dir + targetDirRel;
        //multiply the vector by the bias, then normalize once again
        bias = Vector3.Normalize(new Vector3(bias.x*moveBias,bias.y*moveBias,bias.z*moveBias));
    }
    dir = Vector3.Normalize(dir + bias);
    return dir;
}
```

The attacks and abilities were carried out using semi-random cooldowns, where the AI will simply pick a random ability then wait for that abilities cooldown plus a random factor before attacking again. This mimics the player's capabilities, since all abilities use the same shared cooldown. This gives a somewhat believable attack pattern.

```C#
void GetTicks(){
    shootTick += Time.deltaTime;
    swapModeTick += Time.deltaTime;
    useAbilityTick += Time.deltaTime;

    //error prevention algorithm to cap the tick rate, reseting to 0 when it is above 10
    shootTick %= 10 * shootTickModifyer;
    swapModeTick %= 10 * swapModeTickModifyer;
    useAbilityTick %= 10 * useAbilityTickModifyer;

    if(shootTick>=randomShoot){
        RandomShoot();
        shootTick = 0;
    }
    if(swapModeTick>=randomSwap){
        RandomSwap();
        swapModeTick = 0;
    }
    if(useAbilityTick>=randomUse){
        RandomUseAbility();
        useAbilityTick = 0;
    }
}
```