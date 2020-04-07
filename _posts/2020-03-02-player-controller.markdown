---
layout: post
title:  "Player Relocation"
categories: blue player
---

<div style="width:100%;height:0px;position:relative;padding-bottom:60%;"><iframe src="https://streamable.com/s/5t06q/vniffh" frameborder="0" width="100%" height="100%" allowfullscreen style="width:100%;height:100%;position:absolute;left:0px;top:0px;overflow:hidden;"></iframe></div>

A big part of my task in this sprint is to implement player relocation.
As is documented, when Io touches a hazard but is not dead, she should be relocated to her last safely grounded place.
If she dies, she should be re-spawned to the level's re-spawn point.

# Relocation Procedure
The relocation procedure is a coroutine in the player controller.
```c#
// Class CharacterController2D

IEnumerator RelocateCoroutine ()
{
    if (teleportProjectileReference)
        Destroy(teleportProjectileReference);

    ResetPlayer();
    ResetTimeScale();

    OnPreRelocation.Invoke();

    // Note that here we do not use WaitForSeconds to keep the values during the time.
    float count = 0.0f;
    while (count < preRelocationFreezeTime)
    {
        applyRelocationGravityScale = true;
        canMove = false;
        canApplySlowMo = false;
        canFire = false;

        count += Time.deltaTime;
        yield return null;
    }

    applyRelocationGravityScale = false;

    DoTeleportation(relocationTarget.position);

    slowmoGravityScaleCoroutine = StartCoroutine(PostTeleportationGravityScaleEffect());
    canApplySlowMo = true;
    slowmoAvailable = maxSlowmoLength;
    
    canFire = true;

    relocationTarget = null;
    relocationPriority = 0;

    OnPostRelocation.Invoke();
}
```
What this piece of code does is that it freezes Io for a while before and after relocating,
which gives player some time to accept the truth and to be prepared.

# Relocation Point
Another problem is where to place the relocation point in the scene.
The relocation point should be grounded and safe enough.

In `PlayerCollision` I find a piece of legacy code which detects the collision between the character's boundary box and the hazards.
This piece of code is not gonna to be used in hazard detection because it is done with the hurt box, which is designed to be smaller.
However I find that it is suitable for detecting whether a location is safe enough.

After implementing it, Matt says that the checkpoints in the scene is better to be hand-crafted.
To prevent the change of design from wasting my previous work, I add the concept of 'check zones':
the safe location is only recorded when the player touches check zone.

```c#
if (collInfo.onGround && !collInfo.touchedHazard)
{
    // Check if we are in check zone
    bool touchedCheckZone = TestLayerCollision(checkZoneLayers);

    bool groundSafety = Physics2D.OverlapBox((Vector2)transform.position + bottomOffset, overlapBox_Width * groundSafetyFactor, 0.0f, groundLayers);
    if (touchedCheckZone && groundSafety)
        // Ghost is an object recording the position of relocation point.
        ghost.position = transform.position;
    }
```

In the video above, the white box is the relocation point and the green boxes are check zones.

# Relocation vs. Re-spawning
A problem in Unity is that the order of script execution is undetermined.
The player relocation and death will be triggered in the same frame, but if death is triggered, it should perform a re-spawning instead of relocation.
In order to dealing with this issue, I give higher priority to re-spawning.
```c#
// Class CharacterController2D

[Header("Relocation")]
// ...
private Transform relocationTarget;
private int relocationPriority = 0;

// ...

public void Relocate(Transform target, int priority)
{
    if (priority > relocationPriority)
    {
        relocationTarget = target;
        relocationPriority = priority;

        if (relocationCoroutine != null)
            StopCoroutine(relocationCoroutine);
        relocationCoroutine = StartCoroutine(RelocateCoroutine());
    }
}
```