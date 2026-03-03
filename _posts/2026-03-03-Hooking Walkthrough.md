---
title: "Hooking Walkthrough"
date: 2026-03-03 18:00:00 +0000
categories:
  - Mobile Security
  - Android
  - Reverse Engineering
tags: [android, frida, hooking, game-hacking, reverse-engineering, dynamic-analysis]
image:
  path: /assets/Hooking/4.png
---

# 🎮  Frida Hooking Deep Dive

we explore **dynamic instrumentation** on an Android game using **Frida**.

We will:

- Understand what Hooking is
- Learn how Frida works
- Modify runtime values (Lives, High Score)
- Remove shooting cooldown
- Force coin spawning
- Analyze Activity lifecycle interception




---

# 🧠 What is Hooking?

Android applications are written in Java or Kotlin and compiled into DEX bytecode, which runs inside the Android Runtime (ART) on the device. The runtime is responsible for executing the app’s code, managing memory, and interacting with system APIs. Frida does not intercept a connection between the app and the virtual machine; instead, it injects into the running application process itself. By hooking methods inside ART at runtime, Frida allows us to monitor, modify, and manipulate application behavior dynamically.

Hooking is a technique that allows us to:

1. Intercept a function call
2. Access or modify its arguments
3. Call the original function (or not)
4. Modify its return value

### Normal Flow
EX : 
SUPPOSE U HAVE App do 2 FUNCTION  

1- GET KEY 
2- ENCRYPT

THE REAL  FLOW   CALLER CALL THE FUNCTION AND FUNCTION RETURN A VALUE 

![Desktop View](/assets/Hooking/1.png){: width="700" height="400" .normal }

WITH HOOKING FLOW  WE CAN 

1- HAVE ACCESS TO THE FUNCTION ARGUMENTS 

2- CALL THE REAL FUNCTION WITHOUT THE CALLER NEED 

3- RETUNR  A REAL OR  MODIFIED VALUES



```shell
Caller → Function → Return Value
```

### Hooked Flow

```shell
Caller → Hook → (Optional) Real Function → Modified Return Value
```

![Desktop View](/assets/Hooking/2.png){: width="700" height="400" .normal }



With hooking we can:

- Read sensitive arguments
- Modify internal variables
- Bypass security checks
- Change application behavior at runtime

---

# 🔥 What is Frida?

**Frida is a dynamic instrumentation toolkit** used for:

- Reverse engineering
- Runtime modification
- Security testing
- Mobile penetration testing

Frida allows us to inject JavaScript into a running process and:

- Intercept Java methods
- Modify object fields
- Call internal functions
- Enumerate memory instances

In Android:

Frida bridges communication between the Android application and the Java Virtual Machine (ART/Dalvik).

---

# 📱 Android Activity Lifecycle Hooking


A foreground activity is:

> The screen currently visible and interacting with the user.

We can hook lifecycle methods like `onResume()`.

![Desktop View](/assets/Hooking/3.png){: width="700" height="400" .normal }

### Example: Hook All Activities

```javascript
Java.perform(() => {
    let ActivityClass = Java.use("android.app.Activity");
    ActivityClass.onResume.implementation = function() {
        console.log("Activity resumed:", this.getClass().getName());
        return this.onResume();
    }
});
```

This logs every activity when resumed.

# 🎮 Target Application: SpacePeng
![Desktop View](/assets/Hooking/4.png){: width="700" height="400" .normal }

SpacePeng! is a small, free, and open-source (GPLv3) space shooter game for Android, heavily inspired by classic arcade titles like Space Invaders and Phoenix. It is designed for quick sessions, challenging players to see how many levels they can beat. 


THE PLAYER INSTENCE SOURCE CODE IS :

```java 
package de.fgerbig.spacepeng.components;

import com.artemis.Component;
import com.artemis.annotations.PooledWeaver;

@PooledWeaver
/* loaded from: classes.dex */
public class Player extends Component {
    public static final int DEFAULT_LIVES = 5;
    public static final String SPRITE_NAME = "player";
    public static final String SPRITE_NAME_SHIELD = "playershield";
    public int score;
    private State state = State.ALIVE;
    
    public int lives = 5;

    public enum State {
        ALIVE,
        RESPAWNING,
        DEAD
    }

    public void setState(State state) {
        this.state = state;
    }

    public boolean isState(State state) {
        return this.state.equals(state);
    }
}
```
AS WE SEE IT WE HAVE 2 ROWS OF LIVES 

```java
    public static final int DEFAULT_LIVES = 5;
```

```java
    public int lives = 5;
```


WITH THIS SCRIPT

Package observed during analysis:

de.fgerbig.spacepeng

We decompiled the APK and began analyzing components.

## ❤️ Modifying Player Lives
Player Class Analysis

```java
public class Player extends Component {
    public static final int DEFAULT_LIVES = 5;
    public int lives = 5;
}
```

We notice:

Static default lives

Instance variable lives

Changing static constants at runtime often fails.

So instead, we enumerate existing instances and modify them directly.

## 💉 Frida Script: Modify Player Lives

```java
Java.perform(function() {

    Java.choose("de.fgerbig.spacepeng.components.Player", {

        onMatch: function(instance) {

            console.log("Player instance found: " + instance);

            instance.lives.value = 9005;

            console.log("Lives changed to: " + instance.lives.value);
        },

        onComplete: function() {
            console.log("Done scanning for Player instances");
        }

    });

});
```
![Desktop View](/assets/Hooking/5.png){: width="700" height="400" .normal }


Result

✅ Player now has 9005 lives.

![Desktop View](/assets/Hooking/6.png){: width="700" height="400" .normal }


## 🏆 Modifying High Score

We found another interesting class:

```java
de.fgerbig.spacepeng.services.Profile

Profile Class
private int highScore = 0;

public void setHighScore(int highScore) {
    this.highScore = highScore;
}
```

We can modify:

The private variable directly

Or hook the setter method

## 💉 Frida Script: Force High Score
```java

Java.perform(function() {

    Java.choose("de.fgerbig.spacepeng.services.Profile", {

        onMatch: function(instance) {

            console.log("Profile instance found: " + instance);

            instance.highScore.value = 999999;

            console.log("HighScore changed to: " + instance.highScore.value);
        },

        onComplete: function() {
            console.log("Done scanning for Profile instances");
        }

    });

});
```


Result

✅ High score updated in memory instantly.

![Desktop View](/assets/Hooking/7.png){: width="700" height="400" .normal }

## 🔫 Removing Shooting Cooldown

We analyzed the shooting logic inside:
```java
de.fgerbig.spacepeng.systems.PlayerInputSystem
Fire States

The system depends on:

fireState

timeToShoot

timeToContinue

shoot

world.delta

States:

BLOCKED

CONTINUE

ALLOW

Cooldown logic:

this.timeToShoot = 0.25f;

```

Meaning:

 4 shots per second maximum.

## 💉 Frida Script: Rapid Fire

```java
Java.perform(function() {

    Java.choose("de.fgerbig.spacepeng.systems.PlayerInputSystem", {

        onMatch: function(instance) {

            console.log("Hooking instance: " + instance);

            setInterval(function() {

                instance.setFireAllowed();
                instance.timeToShoot.value = 0;
                instance.shoot.value = true;

            }, 50);

        },

        onComplete: function() {
            console.log("Persistent rapid fire active");
        }

    });

});
```

Result

🔥 Unlimited rapid fire. No cooldown. Always shooting.

![Desktop View](/assets/Hooking/8.png){: width="700" height="400" .normal }

## 🪙 Coin Spawn Manipulation

We found coin spawning logic inside:

```java
de.fgerbig.spacepeng.systems.CoinSpawningSystem
Original Logic
public void dispenseRandomCoin() {
    float r = MathUtils.random();

    if (r < 0.1f) {
        createCoin(EXTRALIFE);
    } else if (r < 0.5f) {
        createCoin(DOUBLESHOT);
    } else {
        createCoin(SHIELD);
    }
}
```
Coins depend on randomness + delay.

## 💉 Frida Script: Coin Rain

```java
Java.perform(function() {

    Java.choose("de.fgerbig.spacepeng.systems.CoinSpawningSystem", {

        onMatch: function(instance) {

            console.log("Hooking CoinSpawningSystem: " + instance);

            setInterval(function() {

                instance.enabled.value = true;
                instance.delay.value = 0.0;

            }, 50);

        },

        onComplete: function() {
            console.log("Persistent coin spam active");
        }

    });

});
```
Result

🪙 Continuous coin spawning. Power-ups everywhere.

![Desktop View](/assets/Hooking/9.png){: width="700" height="400" .normal }




## 🧠 Final Thoughts

Frida allows complete runtime control over Android applications.

With it, we:

Modified lives

Overwrote high scores

Removed shooting cooldown

Forced coin spawning

All without modifying the APK.

Dynamic instrumentation is extremely powerful in mobile security research and reverse engineering.

## Happy Hacking 👾