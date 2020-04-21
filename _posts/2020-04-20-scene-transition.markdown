---
layout: post
title:  "Scene Transition"
categories: blue transition
---

# Scene Transition
My major task of the last sprint was to complete the scene transition between the Crystal Cave and the Windmill Fort.
By design, Io jumps off a deep pit at the end of the level and enters the next scene.
Instead of having a blacked loading scene, I decided to use async load, which loads the scene while not blocking the game loop.

If both scenes share the same transition part, player will merely notice the transition, at least in principle.
Another point is that, all dynamic game objects (including player and boxes with physics) need also to be brought into the new scene, maintaining their relative position and velocity.

In order to pass the essential game objects to the next scene, I used *additive* mode, which does not destroy the old scene after the new one is loaded.
There was a transition room which is the overlap part of both scenes.
Every game object in this room will be preserved.
In the video below, see how the broken wall is passed to the new scene.

<div style="width:100%;height:0px;position:relative;padding-bottom:60%;"><iframe src="https://streamable.com/e/gcsyfa?loop=0" frameborder="0" width="100%" height="100%" allowfullscreen style="width:100%;height:100%;position:absolute;left:0px;top:0px;overflow:hidden;"></iframe></div>

Note that in the video, the transition is still obvious because of Io's scarf.
In the test scene, the transition rooms of both scenes are not in the same position.
A sudden reposition of Io's body stretches her scarf.

<div style="width:100%;height:0px;position:relative;padding-bottom:60%;"><iframe src="https://streamable.com/e/axgeol?loop=0" frameborder="0" width="100%" height="100%" allowfullscreen style="width:100%;height:100%;position:absolute;left:0px;top:0px;overflow:hidden;"></iframe></div>

This is how it looks in real game.
There is one frame when Io slides down the wall where the character view disappears, and that's when transition happens.
Despite that, the system works well - it sends Io to the right place in the new scene.

The solution to this one-frame glitch related to a bug I solved up next.

# Parallax Glitch
My development of this game starts from parallax, and ends from parallax, which is really my fortune.

Play testers report that the parallax backgrounds glitches a lot when the camera is moving.
George and I looked into it and found out that it is caused by async update of the camera and the background.
We changed the update method of both the camera and the background to fixed update and this solved the problem.

But that's how the one-frame glitch is introduced: since the camera updates at fixed rate, it renders just before everything moves to the right place.
Time limited, I had to make an ad-hoc fix: initialize camera's update method to smart update at the first frame, and switch it back immediately.

# Play Testing
In the last week I spend one hour everyday doing nothing but play testing the game.
Most bugs I found were solved in a week.

<div style="width:100%;height:0px;position:relative;padding-bottom:60%;"><iframe src="https://streamable.com/e/ez99br?loop=0" frameborder="0" width="100%" height="100%" allowfullscreen style="width:100%;height:100%;position:absolute;left:0px;top:0px;overflow:hidden;"></iframe></div>

The stalactite does not fall.

<div style="width:100%;height:0px;position:relative;padding-bottom:60%;"><iframe src="https://streamable.com/e/1szds5?loop=0" frameborder="0" width="100%" height="100%" allowfullscreen style="width:100%;height:100%;position:absolute;left:0px;top:0px;overflow:hidden;"></iframe></div>

A controller bug.
I am the coauthor of the player controller in this game.
The player controller actually contains lots of spaghetti methods and coroutines.
If there are something I can do in summer, I think is to refactor the controller into a state machine and do some research in animation generation or adaptive animation.

# Time Breakup

| Title            | Hours | Description |
| :--------------- | ----: | :---------- |
| Scene Transition |     3 |             |
| Parallax Glitch  |     1 |             |
| Play Testing     |     6 |             |
| Discussion       |     4 |             |