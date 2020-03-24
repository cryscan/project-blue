---
layout: post
title:  "Game Manager and Save/Load System"
categories: blue parallax
---

<div style="width:100%;height:0px;position:relative;padding-bottom:60%;"><iframe src="https://streamable.com/s/nasq7/hotkax" frameborder="0" width="100%" height="100%" allowfullscreen style="width:100%;height:100%;position:absolute;left:0px;top:0px;overflow:hidden;"></iframe></div>

# Motivation
In order to save the data required for relocating the player after touching hazards, I implement a Game Manager to serve as a container for those game-wise data.
```c#
// class GameManager

public struct LevelInfo 
{
    GameObject currentLevel;
    Transform currentRespawnPoint;
    Transform currentRelocationPoint;
}
```
When relocating, the handler method references these data and places the player to the right position.

In order to be accessed from anywhere in the project, the Game Manager is made into a singleton and a Don't Destroy on Load (DDOL) object.
```c#
// class GameManager

private static GameManager instance { get; private set; }

private void Awake() 
{
    // Make sure that there will only be one copy at the same time
    if (instance != null && instance != this)
        Destroy(gameObject);
    else
    {
        instance = this;
        DontDestroyOnLoad(gameObject);
    }
}
```

In previous versions, `LevelInfo` was a member of the Game Manager, but later on we found that it is scene specific so it is extracted out and became a component of level zones.
From then on the Game Manager only keeps data that passes through levels, such as player health.
It also serves saving mechanism in the next section.

# Support Save/Load
Saving the player's progress is to write the current game state into a file (serialization) and to load it back next time (deserialization).

At first I had no idea of what kind of information should I store for the progress and planed to make the system as generic as possible.
I have even tried to write a custom attribute annotates individual fields to be saved, which wasted a quite amount of time.
After discussing with pod leaders, we choose only to keep track of the last level and check point the player reached to.

```c#
// class SaveLoad

[System.Serializable]
class SaveData
{
    public string sceneName;

    public string levelPath;
    public string respawnPointPath;
}

void Save() {
    // data to be serialized
    var data = new SaveData
    {
        sceneName = SceneManager.GetActiveScene().name,
        levelPath = levelInfo.currentLevel.transform.GetPath(),
        respawnPointPath = levelInfo.currentRespawnPoint.GetPath()
    };

    // ...
}
```
I set up a structure recording the full paths of corresponding objects because of the following two reason:
1. The progress (defined in `LevelInfo`) are references to `GameObject` and `Transform` (which are determined at runtime)
2. To make it easier for finding those objects back after loading

The method `GetPath()` is an extension to `Transform`:
```c#
public static string GetPath(this Transform current)
{
    if (current.parent == null) 
        return "/" + current.name;
    return current.parent.GetPath() + "/" + current.name;
}
```
After loading the scene, the object references can be fetched by `GameObject.Find()` method using the path.

# Resetting Scene After Loading
The code for loading the scene is inherited from Dream Willow.
The function starts a coroutine which pauses the game, load the scene and resume.
The first implement of scene resetting is intuitive: checking the flag `inTransition` and executing the resetting code immediately after the transition has done.
But this doesn't work: the player location is not set.

Step-by-step investigating shows that the references and the player's location are correctly set, but the player is still generated at the origin.
After checking the [manuel of scene loading](https://docs.unity3d.com/ScriptReference/SceneManagement.SceneManager.LoadScene.html), I found that the scene is loaded in the next frame.

So the solution is:
1. Pass a callback function to the scene loading coroutine to be executed after the `LoadScene()` call
2. Use `yield return null;` to defer the executing of the callback for one frame after reloading the scene
3. Wrap level transition and player relocation code into a closure and pass it to the callback

# Next Step
As is shown in the video, through the player is placed correctly, the camera is not set.
This is because the code for level transition is in somewhere else (oh this problem again).
Later I will refactor level transition code so that the whole process happens together.

# Time Breakdown
| Title        | Hours | Description                                                 |
| :----------- | ----: | :---------------------------------------------------------- |
| Game Manager |     1 | Set up the `GameManager` singleton class                    |
|              |     1 | Separate `LevelInfo` from Game Manager to be a component    |
|              |     2 | Encapsulate parts of level transition code into `LevelInfo` |
| Save         |     1 | Learn how to serialize data and write into file             |
|              |     4 | Implement save system (tried several implementations)       |
| Load         |     2 | Implement scene loading                                     |
|              |     2 | Fix scene resetting bug                                     |