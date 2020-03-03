---
layout: post
title:  "The Worst Thing in OOP"
categories: blue parallax
---

The worst thing in OOP is that everything just happens elsewhere, I am always repeating this sentence these days.

## Taking Damage versus Applying Damage
In unity, components are attached to game objects, each of which has its own data and logic.
When two objects want to interact, for example, handling collisions, one must be the message sender and the other must be the receiver.
Since logic is bound with data of each object, there is no way to deal with both objects in the same piece of logic.
To decide which is which remains an issue throughout the whole project.

The first time I encountered this problem was when working on hurting and relocating the player on touching the hazards.
The one who applies damage is undoubtedly the hazard, but the one who takes damage is not the player, but the player's hurt box, one of its child,
and the receiver of damage message is the `Health` component, which is normally attached to the player.

# Hurt Box Detects Damage
This solution says that the hurt box actively detects the damage, and then forwards the message to the `Health` component.
What the hazards (or enemy attacks) hold is pure data relating to the damage.
```c#
// Class DoDamage.
// Contains only data. Attached to hazards or attacks.

[SerializedField] private int damage;
public int Damage { get { return damage; } }
```

```c#
// Class TakeDamage.
// Attached to player's hurt box.

public DamageEvent OnApplyDamage;

void OnTriggerEnter2D(Collider2D collision) {
    // Only responds to objects have DoDamage as a component.
    DoDamage doDamage = collision.gameObject.GetComponent<DoDamage>();
    if (doDamage) {
        OnApplyDamage.Invoke(doDamage.damage);
    }
}
```
Then the receiver of `OnApplyDamage` event is set to `Health.ApplyDamage` method.

Another feature of the system is to relocate the player after touching the hazards (but not normal enemy attacks).
I found it actually does the same as `TakeDamage` so I wrote the logic in it too.
For more information on player relocation and re-spawning, I have another post on that stuff.

This implementation was rejected by Max.
He said that it's better to separate the code by behaviors
(it makes more sense to let the hazards have `DoDamage` and `DoRelocate` as separate behaviors),
and that this implementation is inconsistent with the existing code.

# Consistency
Actually we have an existed `DamageOnTrigger` script inherited from Dream Willow, which is attached to the objects which can cause damage to player or enemy.
```c#
// Class DamageOnTrigger.
// Attached to hazards or attacks.

[SerializedField] private int damage;

void OnTriggerEnter2D(Collider2D collision) {
    // Find the receiver, the Health component of the player.
    Health health = collision.gameObject.GetComponent<Health>();
    if (health) {
        health.ApplyDamage(damage);
    }
}
```
This is straightforward, but is not correct.
The object which `Health` component is attached to is not the receiver.
I can certainly write
```c#
Health health = collision.gameObject.GetComponentInParent<Health>();
```
But this is too ad-hoc and its semantics is unclear: which `Health` are we exactly want?

# Solution
The current solution is to let the hurt box forward the message to the components doing the job.
The actual logic of applying damage and relocating the player are separated in their own components (`Health` and `CharacterController`).
```c#
// Class HurtBoxEvents.
// The only script attached to the hurt box.

public DamageEvent onApplyDamage;
public RelocateEvent OnRelocate;
```

## What if using ECS
If Entity-Component-System was used in the project, the asymmetricity would disappear.
It's because Systems are independent from any Entities: it just queries Components and manipulates them.
We can program one process regarding to distinct Entities in exactly one System.
For example, when implementing a pong game, in order to check for collision between a ball and a paddle, it is not that one object checks for the other, but that a system checks both at the same time.
```rust
// An example written in Rust using ECS. Bouncing system of Pong game.

#[derive(SystemDesc)]
pub struct BounceSystem;

impl<'a> System<'a> for BounceSystem {
    // Components selected with no limitation to objects (entities).
    type SystemData = (
        WriteStorage<'a, Ball>,
        ReadStorage<'a, Paddle>,
        ReadStorage<'a, Transform>,
        Read<'a, AssetStorage<Source>>,
        ReadExpect<'a, Sounds>,
        Option<Read<'a, Output>>,
    );

    fn run(
        &mut self,
        (mut balls, paddles, transforms, storage, sounds, audio_output): Self::SystemData,
    ) {
        // Query for ball entities.
        for (ball, transform) in (&mut balls, &transforms).join() {
            let ball_x = transform.translation().x;
            let ball_y = transform.translation().y;

            if (ball_y <= ball.radius && ball.velocity[1] < 0.0)
                || (ball_y >= ARENA_HEIGHT - ball.radius && ball.velocity[1] > 0.0)
            {
                // Ball touches boundary.
                ball.velocity[1] = -ball.velocity[1];
                play_bounce_sound(&*sounds, &storage, audio_output.as_ref().map(|o| o.deref()));
            }

            // Query for paddle entities.
            for (paddle, transform) in (&paddles, &transforms).join() {
                let paddle_x = transform.translation().x - paddle.width * 0.5;
                let paddle_y = transform.translation().y - paddle.height * 0.5;

                if point_in_rect(
                    ball_x,
                    ball_y,
                    paddle_x - ball.radius,
                    paddle_y - ball.radius,
                    paddle_x + paddle.width + ball.radius,
                    paddle_y + paddle.height + ball.radius,
                ) && ((paddle.side == Side::Left && ball.velocity[0] < 0.0)
                    || (paddle.side == Side::Right && ball.velocity[0] > 0.0))
                {
                    // Ball collides with paddle.
                    ball.velocity[0] = -ball.velocity[0];
                    play_bounce_sound(&*sounds, &storage, audio_output.as_ref().map(|o| o.deref()));
                }
            }
        }
    }
}
```

Another thing which is worth mentioning is that unlike in OOP, there is no data concealing in ECS.
Games are so complicated softwares that any variable concealed in a class has the potential to be get and set from someone else.
Concealing data may not lead to safety, but a mess.
I am willing to share my thought on this topic in later posts.

## What To Do Now
It is certainly impossible for us to switch the paradigm of the project and it is inevitable to make one thing happen in different places.
I think what we should do now is to set up our coding convention and to stick on it.