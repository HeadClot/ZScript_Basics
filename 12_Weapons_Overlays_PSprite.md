🟢 [<<< BACK TO START](README.md)

🔵 [<< Previous: Player, PlayerInfo and PlayerPawn](12.0_Player.md) 🔵 [>> Next: Arrays](13_Arrays.md)

------

*This chapter is a work-in-progress. To be revised/completed.*

# Weapons, overlays and PSprite

## Table of Contents

* [Overview](#overview)
* [Handling data from weapons](#handling-data-from-weapons)
  + [Accessing data from weapon states](#accessing-data-from-weapon-states)
  + [Accessing data from the weapon's virtual functions](#accessing-data-from-the-weapon-s-virtual-functions)
  + [Checking current weapon](#checking-current-weapon)
  + [When to use Tick()](#when-to-use-tick--)
* [Action functions](#action-functions)
* [PSprite and overlays](#psprite-and-overlays)
  + [Differences between PSprites and actor sprites](#differences-between-psprites-and-actor-sprites)
  + [Difference between PSprites and state sequences](#difference-between-psprites-and-state-sequences)
* [PSprite manipulation](#psprite-manipulation)
  + [Creating PSprites](#creating-psprites)
    * [Native function](#native-function)
    * [ZScript function](#zscript-function)
    * [Layer numbers](#layer-numbers)
    * [PSprite pointers](#psprite-pointers)
    * [Independent overlay animation](#independent-overlay-animation)
  + [Removing PSprites](#removing-psprites)
  + [PSprite flags](#psprite-flags)
  + [PSprite properties](#psprite-properties)
  + [Checking PSprite state](#checking-psprite-state)
  + [PSprite offsets](#psprite-offsets)
  + [Overlay scale, rotate and pivot](#overlay-scale--rotate-and-pivot)
  + [Overlay translation](#overlay-translation)

## Overview

Weapons (i.e. classes that inherit from the base `Weapon` class) feature a number of special behaviors that aren't found in other classes, and you need to be aware of those behaviors to use them effectively.

Here's a brief overview of their features:

* `Weapon` and `CustomInventory` are the only two classes that inherit from an internal `StateProvider` class, which allows them to draw sprite animations on the screen. The sprites drawn on the screen are handled by a separate special class called `PSprite` (short for "player sprite").
* PSprite is a special internal class whose main purpose is to handle on-screen sprite drawing. PSprites hold various properties of those sprite layers, such as duration in tics, position on the screen, and so on.
* On-screen sprites can be drawn in multiple layers. The main layer is layer 1, also defined as `PSP_Weapon` (PSP_Weapon is just an alias of number 1). New layers can be drawn above and below it. A separate PSprite class instance is be created to handle each layer.
* Most weapon functions, such as `A_FireBullets`, `A_WeaponReady` and other functions that are called from the States block, despite being defined in the weapon, are actually called and executed by the [player pawn](https://zdoom.org/wiki/Classes:PlayerPawn) carrying that weapon, rather than the weapon itself. For example, when you call `A_FireProjectile` from the weapon, it's the player pawn that spawns the projectile in the world.
  * For this reason monster attack functions, such as `A_SpawnProjectile`, can't be used in weapons, and vice versa.
* Functions that can be called from weapon states are always [action functions](09_Custom_functions.md#action-functions). Custom functions also have to be defined as action functions.

## Handling data from weapons

Weapons are a subclass of [Inventory](https://zdoom.org/wiki/Classes:Inventory) (the inheritance chain is Inventory > StateProvider > Weapon), so they have access to Inventory's [virtual functions](10_Virtual_functions.md), such as `DoEffect()` (called every tic while the weapon is in somebody's inventory).

However, weapons also have states, which, as described earlier, exist in a unique context, where sprites are drawn by PSprite and functions are executed by the player. Accessing data in those states has its peculiarities.

> *Note*: At this point you may want to refresh your memory about [state flow control](A1_Flow_Control.md#state-control), especially if you're not clear on what "state", "state sequence" and "state label" mean.

### Accessing data from weapon states

Excluding the Spawn sequence states (which are just regular actor states, just like the Spawn states of monsters, decorations, etc.), weapon states (e.g. those in the Ready, Fire, Select sequences, etc.) are not really actor states. They're drawn by a special PSprite class (more on that later), and the functions in those states are executed by the player pawn holding the weapon. This last part means that weapon functions need to interact both with the weapon and with the player: for example, `A_FireProjectile` needs to check various data on the weapon (such as what type of ammo to consume when firing), and then some data on the player pawn (such as its angle and pitch, to determine the direction in which to fire the projectile).

This is true not only for the weapon functions, however, but also for all weapon states. All weapon states exist in a special context, where they interact both with the weapon itself, and with the player pawn that holds that weapon. As a result, there are some rules regarding accessing data from weapon states:

* In the context of a weapon state, `self` is *not* the weapon iself but rather the *player pawn* that holds that weapon.
* The weapon itself is accessible via the `invoker` pointer.
* If you have a variable `varname` defined in your weapon, to access it from a state you need to use `invoker.varname`.
* If you want to access a variable on the player pawn, you can just use its name directly. For example, `speed` returns the value of the `speed` property of the player pawn. You can also use `self` pointer (as in, `self.speed`) but, as everywhere else in ZScript, this prefix is optional.
* Skipping `invoker` in a weapon state context is not optional: GZDoom will abort with a "Self pointer used in ambiguous context" error.

As an example, let's look at this version of the Plasma Rifle with an overheat/cooldown mechanic. Instead of showing a "cooldown" PLSGB0 sprite unconditionally, like the vanilla Plasma Rifle does, it shows it only if the `heatCounter` value reaches 40:

```csharp
class OverheatingPlasmaRifle : PlasmaRifle
{
    int heatCounter; //this holds how much heat is accumulated

    States
    {
    Ready:
        PLSG A 10
        {
            A_WeaponReady();
            // While we're not firing, the heat will decay at the rate
            // of 1 per 10 tics (since the frame duration is 10 tics):
            if (invoker.heatCounter > 0)
                invoker.heatCounter--;
        }
        loop;
    Fire:
        PLSG A 3 
        {
            // If we've gained too much heat, jump to Cooldown:
            if (invoker.heatCounter >= 40)
                return ResolveState("Cooldown");
            // Otherwise accumulate heat and fire:
            invoker.heatCounter++;
            A_FirePlasma();
            return ResolveState(null);
        }
        // note we're not using any cooldown animation here
        goto Ready;
    Cooldown:
        // Display a cooldown frame for 50 tics:
        PLSG B 50 
        {
            // This is meant to be a custom sound:
            A_StartSound("weapons/plasma/cool");
            // Reset the heat counter:
            invoker.heatCounter = 0;
        }
        goto Ready;
    }
}
```

Note that whenever `heatCounter` is referenced in a weapon state, it has to be prefixed with `invoker.` to tell the game it's a variable on the weapon. This happens because, as stated earlier, weapon states interact both with the weapon and with the player carrying it, and as a result the context has to be specified explicitly.

### Accessing data from the weapon's virtual functions

If you want to access data on the weapon or on the player pawn *outside* of a state—i.e. from a virtual function override—the rules are the same as with regular Inventory items:

* For the weapon's variables, use their names directly.

* For the player pawn's variables you will need to use the `owner` prefix.

If you take the Plasma Rifle described above and decide to let the gained heat stacks decay at *all* times instead of just the Ready state sequence, you can do it in `DoEffect()`, where you will not need `invoker` to check the variable:

```csharp
class OverheatingPlasmaRifle2 : PlasmaRifle
{
    int heatCounter;

    override void DoEffect()
    {
        super.DoEffect();
        // Using the modulo operator we can call this block
        // every 10 tics:
        if (level.time % 10 == 0)
        {
            // No 'invoker' required here:
            if (heatCounter > 0)
                heatCounter--;
        }
    }

    // 'invoker' is still required in the states:
    States
    {
    Fire:
        PLSG A 3 
        {
            if (invoker.heatCounter >= 40)
                return ResolveState("Cooldown");
            invoker.heatCounter++;
            A_FirePlasma();
            return ResolveState(null);
        }
        TNT1 A 0 A_ReFire;
        goto Ready;
    Cooldown:
        PLSG B 50 
        {
            A_StartSound("weapons/plasma/cool");
            invoker.heatCounter = 0;
        }
        goto Ready;
    }
}
```

> *Note:* `%` is a modulo operator, you can read about it in the [Arithmetic operators section of the Flow Control chapter](A1_Flow_Control.md#arithmetic-operators).

And, as mentioned, if you want to do something to the player pawn from `DoEffect()`, just use the `owner` pointer like you would in a regular item:

```csharp
// This pistol will give 1 HP to its owner every second:
class HealingPistol : Pistol
{
    override void DoEffect()
    {
        super.DoEffect();
        // Null-check the owner pointer and call the code every 35 tics:
        if (owner && level.time % 35 == 0)
            owner.GiveBody(1); //heal 1 HP
    }
}
```

> *Note*: `Health` property should not be modified directly on players. [`GiveBody()`](https://zdoom.org/wiki/GiveBody) is the only function that should be used for healing player pawns.

### Checking current weapon

It's important to remember that `DoEffect()` is called every tic when an item is in an actor's inventory. This means that if you add something to a weapon's `DoEffect()` override, that stuff will be called even when that weapon isn't selected.

The currently selected weapon can be accessed via the `player.ReadyWeapon`  pointer (in a virtual function it'll also require the `owner` prefix).

Here's a slight modification of the healing pistol mentioned earlier:

```csharp
// This pistol will give 1 HP to its owner every second
// but only while it's equipped:
class HealingPistol : Pistol
{
    override void DoEffect()
    {
        super.DoEffect();
        // Null-check the owner and make sure it's a player:
        if (!owner || !owner.player)
            return;
        // Do nothing if the player has no weapon selected (e.g. is dead)
        // or the selected weapon is different from this one:
        if (!owner.player.ReadyWeapon || owner.player.ReadyWeapon != self)
            return;
        // Heal the owner once a second:
        if (level.time % 35 == 0)
            owner.GiveBody(1);
    }
}
```

### When to use Tick()

You may wonder at this point why I haven't mentioned the `Tick()` function. Do weapons have it?

Yes, all actors have it, and weapons are not an exception. However, you probably won't use it very often, since most of the stuff the weapons need to do they do while in the player's inventory—and for that you'd be better off using `DoEffect()`.

But it's imaginable you may want to add some visual effects or something to the weapon while it's lying on the floor. Just remember one thing: `Tick()` is called *every tic*, it doesn't stop being called when the weapon is picked up. So, for effects that should be applied to weapons that haven't been picked up yet, you should only apply the effects when the weapon doesn't have an owner.

With that out of the way, let's make a simple example. This BFG will emit swirling green particles in a flame-like manner until it's picked up:

```csharp
class BurningBFG : BFG9000
{
    override void Tick()
    {
        super.Tick();
        // If it has an owner, return and do nothing:
        if (owner)
            return;
        A_SpawnParticle
        (
            "00FF00",
            SPF_FULLBRIGHT|SPF_RELVEL|SPF_RELACCEL,
            lifetime:random(50,90), 
            size:frandom(2,5),
            angle:frandom(0,359),
            xoff:frandom(-radius,radius), yoff:frandom(-radius,radius), zoff:frandom(height,height+9),
            velx:frandom(0.5,1.5), velz:frandom(1,2),
            accelx:frandom(-0.05,-0.2), accelz:-0.1,
            startalphaf:0.9,
            sizestep:-0.2
        );
    }
}
```

> *Note:* The way the function above is formatted may seem unusual, but since [`A_SpawnParticle`](https://zdoom.org/wiki/A_SpawnParticle) is a very long function, I'm splitting it into multiple lines and utilizing [named arguments](https://zdoom.org/wiki/ZScript_named_arguments) to be able to clearly see (and show) which arguments I'm defining where.

And this shotgun will try to run (or rather slide) away from you:

```csharp
class RunawayShotgun : Shotgun
{
    Default
    {
        +FRIGHTENED // Runs away from you, not towards you
        speed 5; // Can't move without speed
    }
    override void Tick()
    {
        super.Tick();
        if (owner)
            return;
        // A_Chase required a valid target,
        // so find it first, if there isn't one:
        if (!target)
            LookForPlayers(true);
        A_Chase();
    }
}
```

> *Note:* `LookForPlayers` is one of the many "look" functions available to actors in ZScript. If the first argument is true, like here, it'll look all around the actor (similarly to calling `A_Look` with `LOOKALLAROUND` flag on the actor). In contrast to A_Look* it doesn't imply any state jumps.

## Action functions

As mentioned in the [Custom Functions and Function Types](09_Custom_functions.md#action-functions) chapter, action functions are functions that can be called from a PSprite and differentiate between `invoker` and `self`. With what you've learned in this subsection about handling data from weapons, action functions should hopefully make more sense to you know. 

To reiterate: if a function is defined as an **action** one, it means it interprets the `invoker` and `self` pointers exactly in the same way, as they're interpreted in weapon states: i.e. `invoker` is the weapon, and `self` is the weapon's owner. As such, all functions that are meant to be called from weapon states normally should be defined as action to be able to interact with weapon and player pawn data correctly.

However, exceptions are possible. If a function is defined in the weapon but isn't an action function, it can still be called in a weapon state, but in that case, like all weapon-scope data, it'll require an `invoker.` pointer. One example of a native Weapon function that isn't define as an action one is `DepleteAmmo()`, which is a function that—you guessed it—depletes the weapon's ammo. (You can find it [on GZDoom Github repository](https://github.com/coelckers/gzdoom/blob/d348bad8234dffa9c8739c0a8233de166f2c9aea/wadsrc/static/zscript/actors/inventory/weapons.zs#L994).)

As an example, let's say you want your weapon to fire multiple projectiles but only consume 1 ammo. You could, of course, call `A_FireProjectile` with the `useammo` argument set to true once, and then call it several times more with that argument set to false, but it's not particularly pretty. Instead, you could do this:

```csharp
class PlasmaShotgun : Shotgun
{
    States
    {
    Fire:
        SHTG A 3;
        SHTG A 7 
        {
            // Consume 1 shell:
            invoker.DepleteAmmo(invoker.bAltFire, true);
            // Execute A_FireProjectile 8 times
            for (int i = 8; i > 0; i--)
            {
                A_FireProjectile("Plasmaball", angle: frandom(-5.6, 5.6), useammo: false);
            }
            A_GunFlash();
        }
        SHTG BC 5;
        SHTG D 4;
        SHTG CB 5;
        SHTG A 3;
        SHTG A 7 A_ReFire;
        Goto Ready;
    }
}
```

Using a [for loop](A1_Flow_Control.md#loop-control) we execute A_FireProjectile 8 times, making sure it doesn't consume any ammo. Note that `DepleteAmmo()` has to be called as `invoker.DepleteAmmo()` since it's being called on the weapon. It also requires passing `invoker.bAltFire` (an internal flag that determines whether the weapon is currently using primary or secondary fire).

As mentioned, the difference between action and non-action functions is how they interact with the pointers:

**Action function:**

* `invoker` — the caller of the function (for weapons it's the weapon itself)

* `self` — the owner, cast as Actor

* `player` (or `self.player`) — the PlayerInfo struct attached to the owner (as long as the owner is a player)

* `player.mo` (or `self.player.mo`) — the owner, cast as PlayerPawn

**Regular function:**

* `self` — the caller of the function (i.e. the weapon)

* `owner` — the owner, cast as Actor

* `owner.player` (or `self.player`) — the PlayerInfo struct attached to the owner (as long as the owner is a player)

* `owner.player.mo` — the owner, cast as PlayerPawn

* ~~`invoker`~~ — this pointer isn't recognized

## PSprite and overlays

PSprite (short for "player sprite") is a special class that handles drawing sprites on the player's screen (but below the HUD/UI). The sprites themselves are defined within Weapon/CustomInventory but drawn on the screen with the help of PSprite. If multiple sprite layers are used, a separate PSprite instance is created for every layer.

> *Note:* The terms "PSprite" and "overlay" are somewhat interchangeable. "PSprite" refers to either the PSprite class itself, or a specific PSprite instance (i.e. a sprite layer that is currently being handled by PSprite). The latter can also be called an "overlay", in reference to the A_Overlay* functions.

Like regular sprites, PSprites have duration in tics, offsets (those define their position on the screen), and as of GZDoom 4.5.0 they can also be rotated and scaled. You can even use the same images as PSprite and as an actor sprite, although normally this won't work well because different offsets are used, but in principle its doable.

### Differences between PSprites and actor sprites

The main differences between PSprites and regular actor sprites are:

* PSprites are drawn on the screen, just below the HUD.
* PSprites can be drawn in multiple layers, over and under each other, while still being part of the same weapon or CustomInventory. In contrast, actor sprites can't have layers.
* PSprites contain a lot of data that can be read and modified: duration, offsets, scale, rotation angle, the index of the layer the sprite is being drawn on.
* While you can access an actor's current sprite via its `sprite` field, accessing the weapon's PSprite requires `FindPSprite()` or `GetPSprite()` functions that return a pointer to the PSprite of a specific layer (more on that below).

### Difference between PSprites and state sequences

It's important to understand that PSprites and state sequences are two entirely different concepts. A state sequence defines a sequence of sprite frames and function calls. The *same* state sequence can be drawn multiple times, on multiple layers, even at the same time.

Any state sequence defined in a Weapon (aside from Spawn) can be drawn as PSprite. While `PSP_Weapon` (aka layer 1) is used by default and has to exist for the weapon to function, any other sequence can be drawn on any other layer. 

## PSprite manipulation

PSprites can be drawn in multiple layers. By default weapon sprites are drawn on layer 1, which also has an alias of `PSP_Weapon` (i.e. writing `PSP_Weapon` is the same as writing `1`). This layer has to exist for the weapon to function, and it can only be null under specific circumstances, e.g. when the player is dead (their weapon becomes deselected but no other weapon is selected), they have no weapons (e.g. they have been taken away by a script), or `stop` has been called in a weapon's state sequence used by layer 1 (which is almost always a mistake and shouldn't be done).

Vanilla Doom used layers in a very limited manner: most of the weapon sprites were drawn on the same layer (PSP_Weapon), and only muzzle flashes were placed on a separate layer, so that they could be made fullbright without making the whole weapon fullbright. This was achieved via `A_GunFlash`, which is a simple function that draws a predefined state sequence (either `Flash` or `AltFlash`, depending the function was called in `Fire` or `AltFire`) on the `PSP_Flash` layer (which is layer 1000). `A_GunFlash` doesn't allow choosing the layer; layers created by it are always drawn on layer index 1000, which also uses an alias `PSP_Flash`.

GZDoom, however, offers functions that allow drawing any state sequence on any layer, allowing mod authors to create weapons with however many layers they want, and control those layers in various ways:

* There's a number of `A_Overlay*` functions that can be found among the [list of weapon functions on the ZDoom wiki](https://zdoom.org/wiki/Category:Decorate_Weapon_functions). Most of them are available both in ZScript and DECORATE.

* There are also ZScript-only PSprite methods: a selection of player functions that can manipulate PSprites. For example, it's possible to get a pointer to a specific PSprite via `player.FindPSprite` or `player.GetPSprite` and manipulate its state and properties directly. 

In this subsection we'll be taking a look at both approaches in parallel.

In order to explain in detail all PSprite interactions and manipulations, among other things I'll also be using a new weapon class as an example. In contrast to most other examples in this guide, this one will use an original set of sprites, not found in Doom. Here they are:

PGUNA0
![Graph](pistolAngled/PGUNA0.png)

PGUNB0
![Graph](pistolAngled/PGUNB0.png)

PGUNC0
![Graph](pistolAngled/PGUNC0.png)

PGUND0
![Graph](pistolAngled/PGUND0.png)

PGUFA0
![Graph](pistolAngled/PGUFA0.png)

PGUFZ0
![Graph](pistolAngled/PGUFZ0.png)

The sprites can be found in the `pistolAngled` folder accompanying this guide. As you can see, there are 4 sprites here showing the movement of the slide (PGUNA–PGUND), one sprite with muzzle highlights (PGUFA) designed to be overlayed on top of PGUNA, and one sprite with the muzzle flash (PGUFZ). I will be referring to these sprites specifically in the example code for `PistolAngled` and `PistolAngledDual` below.

### Creating PSprites

##### Native function

The main overlay function is `A_Overlay(<layer>, <state label>, <override>)`. Historically an expanded version of `A_GunFlash`, this is the function that creates overlays: more specifically, it creates a new PSPrite on a specified layer, and draws the specific state sequence on that layer, independently from PSP_Weapon.

It's important to note that muzzle flashes in vanilla Doom could only be drawn above the weapon layer (since `A_GunFlash` draws on layer 1000). This approach requires cutting out the shape of a barrel from the muzzle flash sprite, which can be annoying. Also, vanilla Doom sprites highlight a specific, roughly cut-out section of the gun's barrel, rather than using realistic muzzle highlights, which is pretty noticeable in the dark environment.

Using the custom sprites provided above, we'll define a more complex muzzle flash: it'll consist if a flash sprite drawn *below* the gun (using negative layer numbers), and a muzzle highlights layer drawn *above* the gun:

```csharp
class PistolAngled : Pistol
{
    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(-2, "Flash");
            A_Overlay(2, "Highlights");
        }
        PGUN BD 1;
        PGUN CBA 2;
        PGUN A 5 A_ReFire;
        goto Ready;
    Flash:
        PGUF Z 2 bright A_Light1;
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        loop;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

> *Note:* It's important to remember that the [vanilla weapon illumination functions](https://zdoom.org/wiki/A_Light0), such as as `A_Light1` and `A_Light2`, actually illuminate *the whole level*, and if you don't call `A_Light0` after them, the map's light level will remain permanently raised. While using them is typical in vanilla weapons, it's not a requirement; you may opt for something GZDoom-specific, like a dynamic light.

The third argument of `A_Overlay` is a boolean value by default set to `false`. If it's set to `true`, the overlay will not be created if that layer is already occupied by a PSprite. If `false`, the overlay will be created unconditionally, and if any animation was already active on that layer, it'll be destroyed first. 

You can use that argument if you want to avoid recreating the layer when not necessary; more examples of that will be provided below.

##### ZScript function

A ZScript-only player function that behaves similarly to `A_Overlay` is `SetPSprite`. The syntax is:

```csharp
player.SetPSprite(<layer number>, <state pointer>);
// 'Layer' is an integer number defining the
// number of the sprite layer to be created.
// 'State' is a state pointer.
```

It has to be called with the `player` prefix if called from a weapon state. Also, instead of a state label it takes a state pointer as a 2nd argument, so if you want to feed it a state label, you need to get a state pointer. As described in [Flow Control](A1_Flow_Control.md#finding-and-returning-states-in-zscript), there are two functions that can do that—`FindState()` and `ResolveState()`—but only `ResolveState()` is context-aware and can be used in PSprite context.

Other than that it can be used as a direct analog of `A_Overlay`:

```csharp
class PistolAngled : Pistol
{
    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            player.SetPSprite(-2, ResolveState("Flash"));
            player.SetPSprite(2, ResolveState("Highlights"));
        }
        PGUN BD 1;
        PGUN CBA 2;
        PGUN A 5 A_ReFire;
        goto Ready;
    Flash:
        PGUF Z 2 bright A_Light1;
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        loop;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

The advantage of `SetPSprite` is that, in contrast to `A_Overlay`, it can be called *outside* of a weapon state, e.g. from its virtual function. This can be useful if you need to be able to conditionally create a PSprite layer independently from the state sequence the main layer is in. You'll find an example of that later.

##### Layer numbers

Every sprite layer handled by a PSprite instance has a number. Some of these numbers have aliases:

- `PSP_Weapon` — same as `1`, this is the main weapon layer. All weapons need it to function, and it can only be null if the player has no weapons at all or is dead. Do not try to destroy or recreate this layer; use regular state jumps to change it.

- `PSP_STRIFEHANDS` — same as `-1`, used by [`A_ItBurnsItBurns`](https://zdoom.org/wiki/A_ItBurnsItBurns) function from Strife, to show the player's burning hands. This function has special behavior tied to it, where it will display a specific "FireHands" state sequence defined on the *player pawn* rather than on any of the player's weapon or custom inventory objects; the activator of the function will also be set to the player rather than the weapon. In short, it's best not to touch this layer either.

- `PSP_Flash` — same as `1000`, used by the `A_GunFlash` in the vanilla Doom weapons to draw muzzle flashes separately from the weapon sprite. Normally there's no reason to use it.

- `OverlayID()` — in contrast to the others, this isn't a constant but rather a function that returns a dynamic value: the number of the layer used by the state where this function is called. Think of it like using "self" of sorts.

Aside from `OverlayID()`, the aliases above are constants, they always point to the same numbers. It's also recommend to avoid them when creating new layers, and stick to numbers -2 and below, as well as 2 and above.

It is, however, not recommended to use numbers literally. If your weapon has several layers, it's very easy to lose track of them in a range of numbers, and also it'll be very hard to change the numbers if you realize that you need to squeeze a new layer between existing ones. A much more sensible approach is to use an [enum](14_Constants.md#enums):

```csharp
class PistolAngled : Pistol
{
    enum PALayers
    {
        // I prefer to use higher values so that I can
        // easily insert more layers between them if needed:
        PSP_MFlash = -10,
        PSP_Hlights = 10
    }

    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(PSP_MFlash, "Flash");
            A_Overlay(PSP_Hlights, "Highlights");
        }
        PGUN D 2;
        PGUN CBA 3;
        PGUN A 5 A_ReFire;
        goto Ready;
    Flash:
        PGUF Z 2 bright A_Light1;
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        loop;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

As mentioned, calling `OverlayID()` will return the number of the layer where the state sequence the function is called in is present. If you want to modify the sprite layer used by the state sequence from the state sequence itself, it's preferable to use `OverlayID()` rather than defining a number directly.

Do not try to use `OverlayID()` in `A_Overlay`, since this will try to destroy the current layer and create a new one in its place. `OverlayID()` specifically designed to be used for *modifying* layers, not creating them (e.g. modifying PSprite flags—more examples on that below).

##### PSprite pointers

Once a PSprite has been created, you can modify its properties. For that you can either utilize the native A_Overlay* functions, or get pointers to them and use ZScript functions. To use the latter approach you first need to know how to get those pointers.

There are two functions that can do that:

* `FindPSprite(<layer number>)` — checks if the specified sprite layer exists; if it does, returns a pointer to it. Otherwise returns null.

* `GetPSprite(<layer number>)` — checks if the specified sprite layer exists; if it doesn't, creates it and then returns a pointer to it.

Just like `SetPSprite`, these will need a `player` prefix if used in a state. In other contexts you'll also need a pointer to the player pawn first, so, for example, from a weapon's virtual function the prefix will be `owner.player`.

Most of the time you'll be using `FindPSprite` to get access to the PSprite. This is done pretty much the same way as any other [pointer casting](08_Pointers_and_casting.md#casting-and-custom-pointers):

```csharp
// Get a pointer to the PSprite of layer 5
// (assuming this is a layer you created
// earlier, since it's not used by the
// Doom weapons):
let psp = player.FindPSprite(5);

// Get a pointer to the PSprite of layer 1,
// the main weapon layer:
let psp = player.FindPSprite(PSP_Weapon);

// Get a pointer to the PSprite of layer 1000,
// normally used by the "Flash" sequence:
let psp = player.FindPSprite(PSP_Flash);

// Get a pointer to the PSprite of the current
// layer (i.e. the layer used by the state
// where this is called from):
let psp = player.FindPSprite(OverlayID());
```

> *Note:* As with all casting, the name of the pointer can be anything. However, it's fairly common practice to make pointer names something similar to what they're pointing to, so `psp` is a commonly used name for a PSprite pointer. You can use whatever works for you.

Once you establish a pointer to a PSprite, you can use that pointer to modify all properties on it (more examples of that in the further subsections).

If you absolutely need to make sure that the specified layer actually exists, `GetPSprite` should be used instead of `FindPSprite`, because `GetPSprite` will always create the specified PSprite, so it never returns null. However, `FindPSprite` is useful when you actually need to do different things depending on whether the PSprite exists: e.g. in the example above we null-check `PSP_Weapon` because it can be null if the player is dead (since the current weapon gets deselected in that case).

PSprite pointers are used when you need to get a direct access to the layer's properties or flags. `Curstate` is one of such properties; more examples will be provided below.

##### Independent overlay animation

PSprite layers exist largely separately from each other. Once a layer has been created, it can progress through state sequences, animate and call functions independelty from other functions.

To give a more advanced example of overlay animation, let's add a quick punch animation as an additional attack to our angled pistol. We can perform it in such a way that it'll be possible to execute it completely separately from the pistol animation, both when the pistol is read or when it's firing. 

Before we do it, however, we need to remember that we can't utilize the typical attack states, such as `AltFire` sequence for this. The reason is simple: if we add `AltFire` to the weapon, that state sequence will be utilized by PSP_Weapon, breaking our pistol animation. So, we need to make sure that animation is entered by a *different* sprite layer, and as such as we can't utilize the native attack states.

Thankfully, we can utilize `player.cmd.buttons` pointer, which is a bit field that contains the buttons currently pressed by the player:

```csharp
class PistolAngledFist : Pistol
{
    enum PALayers
    {
        PSP_Fist        = -15, //we'll use this for our punch animation
        PSP_MFlash      = -10,
        PSP_Hlights     = 10
    }

    States
    {
    Ready:
        PGUN A 1 
        {
            A_WeaponReady();
            // This returns true if the player presses
            // the alt fire button:
            if (player.cmd.buttons & BT_ALTATTACK)
            {
                // Note the third argument is true, so that
                // the layer isn't continuously overwritten
                // if it's already active:
                A_Overlay(PSP_Fist, "Punch", true);
            }
        }
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(PSP_MFlash, "Flash");
            A_Overlay(PSP_Hlights, "Highlights");
        }
        PGUN D 2;
        PGUN CBA 3;
        PGUN A 5 A_ReFire;
        goto Ready;
    // The punch animation. Almost a copy of the vanilla
    // Doom fist, except we need to use A_CustomPunch
    // because, surprisingly, the vanilla A_Punch function
    // actually consumes ammo:
    Punch:
        PUNG B 4;
        PUNG C 4 A_CustomPunch(2, meleesound: "*fist");
        PUNG D 5;
        PUNG CB 4;
        // We can safely call 'stop' here to remove the
        // overlay. Note, doing that on the main layer
        // would destroy it and make the weapon unusuable.
        stop;
    Flash:
        PGUF Z 2 bright A_Light1;
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        loop;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

So, this allows us to perform a punch while the main layer is in the Ready sequence. But what if we want to let the player *always* perform a punch, regardless of what the pistol is doing? Let's say we won't to be able to punch while firing (don't do this at home).

It's simple: we just move the `player.cmd.buttons` check to `DoEffect()`, which, as we remember, is called every tick the weapon is in inventory. We also have to utilize ZScript functions and use the correct pointers, as explained above:

```csharp
class PistolAngledFist : Pistol
{
    enum PALayers
    {
        PSP_Fist        = -15,
        PSP_MFlash      = -10,
        PSP_Hlights     = 10
    }

    override void DoEffect()
    {
        super.DoEffect();
        // First, return and do nothing if the owner is null
        // or isn't a player:
        if (!owner || !owner.player)
            return;

        // Then do nothing if the player has no readyweapon
        // or the readyweapon is not this weapon (because we
        // don't want the fist to appear while using other
        // weapons the player may have):
        if (!owner.player.readyweapon || owner.player.readyweapon != self)
            return;

        // Null-check the main weapon layer. If it's null
        // (e.g. because the player is dead), we don't want
        // to do anything:
        let psp = owner.player.FindPSprite(PSP_Weapon);
        if (!psp)
            return;

        // Check if the player is pressing alt fire:
        if (owner.player.cmd.buttons & BT_ALTATTACK)
        {
            // Get a pointer to PSP_Fist:
            let pspf = owner.player.FindPSprite(PSP_Fist);
            // Only create an overlay if it IS null,
            // because we don't want to recreate it
            // if the layer is already used:
            if (!pspf)
            {
                owner.player.SetPSprite(PSP_Fist, ResolveState("Punch"));
            }
        }
    }

    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(PSP_MFlash, "Flash");
            A_Overlay(PSP_Hlights, "Highlights");
        }
        PGUN D 2;
        PGUN CBA 3;
        PGUN A 5 A_ReFire;
        goto Ready;
    Punch:
        PUNG B 4;
        PUNG C 4 A_CustomPunch(2, meleesound: "*fist");
        PUNG D 5;
        PUNG CB 4;
        stop;
    Flash:
        PGUF Z 2 bright A_Light1;
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        loop;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

### Removing PSprites

##### Native function

An existing overlay can be removed by calling `A_ClearOverlay(<start>, <end>)` where and are the first and last layer number for the range you want removed. Note, the second argument is not optional; if you want only one layer removed, you need to provide the same value for both.

##### ZScript method

If you have a pointer to a PSprite instance, you can remove it in two ways: by moving it to the "Null" state sequence via `SetPSprite` (which all actors have by default, it doesn't need to be defined) or by calling `Destroy()` on it:

```csharp
// Method #1:
let psp = player.FindPSprite(PSP_Flash);
if (psp)
{
    player.SetPSprite(PSP_Flash, "Null");
}

// Method #2:
let psp = player.FindPSprite(PSP_Flash);
if (psp)
{
    player.SetPSprite(PSP_Flash, "Null");
}

```

### PSprite flags

`A_OverlayFlags(<layer>, <flag>, <true/false>)` can be used to set or unset PSprite flags. 

Alternatively, you can get a pointer to a PSprite layer via something like `let psp = player.FindPSprite(<layer>)` (where <layer> is the actual number/alias), and after that you'll be able to access and modify the flags via `psp.bFLAGNAME`. This works exactly the same way as modifying flags on actors via [actor pointers](08_Pointers_and_casting.md) does.

In short, the two ways to modify a PSprite flag look as follows:

```csharp
// Set the "AddWeapon" flag on the current layer
// to false using a native function:
SPRT A 1 A_OverlayFlags(OverlayID(), PSPF_AddWeapon, false);


// Set the "AddWeapon" flag on the current layer
// to false using a PSprite pointer:
SPRT A 1
{
    // Get a pointer:
    let psp = player.FindPSprite(OverlayID());
    // Null-check it:
    if (psp)
    {
        // Modify the value:
        psp.bAddWeapon = false;
    }
}
```

Talking about the flags is not particularly meaningful on its own, since flags are mostly used to allow other PSprite-related features to work. I'll briefly cover the most common flags and then I'll be referencing those flags further in this chapter.

Note that `A_OverlayFlags` doesn't use the real flag names, rather it uses their aliases that start with "PSPF_". The internal names of the flags are different, so I'll be providing them in parallel:

* `PSPF_AddWeapon` (internally `bAddWeapon`) — this flag is used by *overlays* (i.e. all PSprite layers that are *not* PSP_WEAPON) and it makes the overlay follow the main layer's offset ([PSprite offsets](#psprite-offsets) below will cover that in detail). This flag is **true** by default. If you want to move an overlay around separately from the main layer, you need to set it to false.

* `PSPF_RenderStyle` — allows changing the sprite layer's renderstyle, just like it can be done on actors. After setting it to true, you can use [`A_OverlayRenderstyle`](https://zdoom.org/wiki/A_OverlayRenderstyle) to change the layer's style. **This flag can't be modified directly** through a pointer, and neither can the renderstyle itself; renderstyle manipulation can only be done from the C++ part of the engine via functions. This is true for actors as well, which require `SetRenderstyle()` to change their renderstyle dynamically.

* `PSPF_Alpha` and `PSPF_ForceAlpha` — these two flags enable modifying the alpha (i.e. translucency) of the sprite layer. The difference between the two is that some renderstyles (set via `A_OverlayRenderstyle`) force alpha to a specific value and block changes to it, which `PSPF_Alpha` can't override while `PSPF_ForceAlpha` can. In practice it means that `PSPF_Alpha` can basically be ignored, and you can always use `PSPF_ForceAlpha` instead if you need to make sure the overlay's alpha will be set to the value you want. Similarly to `PSPF_RenderStyle`, these two **can't be modified directly**, only  via `A_OverlayFlags`.

* `PSPF_Flip` (internally `bFlip`) — flips the sprite horizontally. Note, this won't affect its placement on the screen, just its visuals.

* `PSPF_Mirror`  (internally `bMirror`) — mirrors the sprite's horizontal offsets (leftward becomes positive numbers, rightward becomes negative). This affects both the offsets you set to the sprite in Slade, and the offsets set via functions such as `A_WeaponOffset`/`A_OverlayOffset`. Note, this doesn't affect the sprite's visuals—for that you need to combine this with `PSPF_Flip`. Combining both flags is necessary if you need to completely mirror the sprite; it can also be useful when creating akimbo weapons (will be described later in the chapter).

The full list of flags that can be modified with `A_OverlayFlags` can be found [on the wiki](https://zdoom.org/wiki/A_OverlayFlags). The internal flag names are defined as boolean values in the PSprite class in GZDoom, you can find them [on the GZDoom GitHub repository](https://github.com/coelckers/gzdoom/blob/254da4b7699cc4d3abd964c9f4f0e2bf31f8bb20/wadsrc/static/zscript/actors/player/player.zs#L2622).

As mentioned, talking about flags separately from other features isn't particularly meaningful, but just to give you an example of the syntax:

```csharp
class RightHandFist : Fist
{
    States
    {
    Select:
        TNT1 A 0 A_OverlayFlags(PSP_Weapon, PSPF_Flip|PSPF_Mirror, true);
        goto super::Select;
    }
}
```

> *Note:* Don't let the word "overlay" in the names of the functions confuse you: they can be used on any layer, including the main weapon layer.

This is an extremely simple modification of the Doom's Fist weapon to make it right-handed. Note, we use both `PSPF_Flip` and `PSPF_Mirror`, so that both the sprites visuals *and* position are flipped. We call the function in the first Select state, so that it's applied immediately.

As for modifying the flags directly, in general it's not =really more useful than using `A_OverlayFlags`, since the code will be longer. However, it's still useful to know how it works. Here's the same right-hand fist done via ZScript pointers:

```csharp
class RightHandFist : Fist
{
    States
    {
    Select:
        TNT1 A 0 
        {
            let psp = player.FindPSprite(PSP_Weapon);
            if (psp)
            {
                psp.bFlip = true;
                psp.bMirror = true;
            }
        }
        goto super::Select;
    }
}
```

### PSprite properties

PSprite properties can be accessed and modified similarly to actor pointers. This can be done via native A_Overlay* functions, where each function is designed to modify a specific property. 

Alternatively, you can get a pointer to a PSprite layer via something like `let psp = player.FindPSprite(<layer>)` (where is the actual number/alias), and after that you'll be able to access and modify the properties via `psp.propertyname`. This works exactly the same way as modifying flags on actors via [actor pointers](08_Pointers_and_casting.md) does.

However, in contrast to the previous section on flags, here I will split the native functions and direct property manipulation into separate subsections, because the correlations aren't always as obvious, and the list of internal properties is actually larger than the list of the native property-related functions.

##### Native property-altering functions

There are a few functions that can modify PSprite properties. Most of them will be covered in separate subchapters, but here's a brief overview:

* `A_OverlayRendestyle(<layer>, <style>)` — modifies the renderstyle of the specified layer. The styles are mostly the same as actor render styles: normal, translucent, additive and so on. The full list and their names is provided [on the wiki](https://zdoom.org/wiki/A_OverlayRenderstyle). Note, this is the only way to modify the renderstyle of a PSprite—similarly to actors, its renderstyle can't be modified directly (actors require the `SetRenderstyle` function).

* `A_OverlayAlpha(<layer>, <value>)` sets the specified layer's alpha (translucency) to the specified double value. For this to work, the renderstyle has to support alpha modification. The fastest way to allow this is to set the `PSPF_ForceAlpha` flag to true, since this forcibly allows alpha manipulation with any renderstyle.

* `A_OverlayTranslation(<layer> , <translation name>)` — modifies the translation of the specified layer, setting it to a translation as defined in the [TRNSLATE](https://zdoom.org/wiki/TRNSLATE) lump.

* `A_WeaponOffset` and `A_OverlayOffset` allow moving the main sprite layer and the overlay layers around; covered in detail below.

* `A_OverlayScale (<layer>, <x>, <y>, <flags>)` allows scaling the sprite layer; covered in detail below.

* `A_OverlayRotate (<layer>, <angle>, <flags>)` allows rolling the sprite layer; covered in detail below.

* `A_OverlayPivot` and `A_OverlayPivotAlign` allow moving the point of orientation for scaling and rolling.

Here's a simple example of changing the renderstyle:

```csharp
class FuzzyRightHandedFist : Fist
{
    States
    {
    Select:
        TNT1 A 0
        {
            // Flip the sprite and enable renderstyle change:
            A_OverlayFlags(PSP_Weapon, PSPF_Flip|PSPF_Mirror|PSPF_RenderStyle, true);
            // Apply fuzzy renderstyle:
            A_OverlayRenderstyle(PSP_Weapon, Style_Fuzzy);
        }
        goto super::Select;
    }
}
```

This Fist will appear fuzzy, like the Spectre enemy, and will also be right-handed as in the example used earlier.

##### Internal properties

Once you get a pointer to a PSprite, you can modify its properties. Properties are defined the same way they are defined in actors (except PSprite doesn't have any sort of Default block). You can find them in GZDoom, [in the PSprite class](https://github.com/coelckers/gzdoom/blob/254da4b7699cc4d3abd964c9f4f0e2bf31f8bb20/wadsrc/static/zscript/actors/player/player.zs#L2583).

Some of the PSprite properties:

* `curstate` — a pointer to the state the layer is currently is. Using [`InStateSequence`](A1_Flow_Control.md#checking-current-state-sequence), you can check in which sequence it is (see example with an overheating plasma rifle above). This can not be modified with the native A_Overlay* functions.

* `sprite` — same as with actors, this contains the ID of the sprite currently displayed by the PSprite. Note, it's not a *name* of the image, but rather a `SpriteID`. It can be modified directly with GetSpriteIndex("SPRT")` where "SPRT" is the actual name of the graphic. This can not be modified with the native A_Overlay* functions.

* `frame` — same as with actors, this contains the current sprite frame used by the PSprite. It's an integer value, where 0 equals A, 1 equals B, and so on. Can be modified directly. This can not be modified with native A_Overlay* functions.

* `x` and `y` are the double values that contain the horizontal and vertical position of the sprite layer, i.e. its offsets. Can be modified directly, but normally using `A_OverlayOffset` and `A_WeaponOffset` is more convenient.

* `tics` — the duration of the current frame in tics. Can be modified directly or by calling [`A_SetTics`](https://zdoom.org/wiki/A_SetTics) from the corresponding state.

* `alpha` — the current alpha value of the sprite layer. Can be modified direclty or with `A_OverlayAlpha`.

* `scale` — a vector2 value that contains the scale of the sprite layer. Can be modified directly or with `A_OverlayScale`.

* `rotation` — a double value that contains the current roll of the sprite layer. Can be modified directly or with `A_OverlayRotate`.

* `translation` — a uint value that contains the current translation of the sprite layer. It's similar to the actor `tranlsation` field. Can be modified via `A_OverlayTranslation` or by copying another translation into it.

Most of these properties require additional explanations, so I'll cover them in detail and provide examples in the subsections below.

### Checking PSprite state

Once you get a pointer to a Psprite, you can use [`InStateSequence`](A1_Flow_Control.md#checking-current-state-sequence) to check the PSprite's `curstate` and find which state the overlay is in.

`InStateSequence(<current state>,  <state sequence>)` is a static boolean function that returns true if the specified state is currently inside the specified state sequence. In short:

```csharp
// Called from a weapon state:
let psp = player.FindPSprite(PSP_Weapon);
if (psp && InStateSequence(psp.curstate, ResolveState("Ready"))
{
    // This block is executed if PSP_Weapon is non-null
    // and is currently in the Ready sequence.
}
```

>  *Note:* Finding state sequences is done the same way as it is done when jumping from anonymous functions—with the help of `ResolveState`. This is covered in more detail [in the Flow Control appendix](A1_Flow_Control.md#finding-and-returning-states-in-ZScript). Note, you need to use `ResolveState`, not `FindState` in the weapon context, since `FindState` is not aware of the invoker/self difference.

For a practical example let's forget our angled pistol for a second and look back to our overheating plasma rifle. In the example we originally used the heat counter would decay in the weapon's `DoEffect()`, which meant it would decay at all times—even while the weapon was firing. This may not be optimal, since usually in systems like this if the counter grows, it shouldn't decay at the same time.

We can utilize a `curstate` pointer with`InStateSequence` to make sure the heat can't dissipate while the weapon is firing:

```csharp
class OverheatingPlasmaRifle3 : PlasmaRifle
{
    int heatCounter;

    override void DoEffect()
    {
        super.DoEffect();
        // Double-check the owner exists and is a player:
        if (!owner || !owner.player)
            return;
        // In our earlier example we let heat stacks decay only
        // once 10 tics. We do the same here, but we invert the
        // check and move it up for performance reasons: this
        // is the simplest check in the chain, so it makes sense
        // to start with it. Of course, we invert it, so that we
        // can plug a return after it:
        if (level.time % 10 != 0)
            return;
        // Get a pointer to the PSprite that is handling layer 1:
        let psp = owner.player.FindPSprite(PSP_Weapon);
        // Always null-check pointers. For example, if the player
        // is dead, its PSP_Weapon layer will be null, so we need
        // to do nothing in this case:
        if (!psp)
            return;
        // Now check that psp.curstate (the current state of the
        // PSprite we found) is in the "Fire" sequence. If so,
        // return and do nothing (i.e. heat doesn't dissipate).
        if (InStateSequence(psp.curstate, ResolveState("Fire"))
            return;
        // And only after all of this we let the heat stacks
        // decay:
        if (heatCounter > 0)
            heatCounter--;
    }

    // The states block remains unchanged:
    States
    {
    Fire:
        PLSG A 3 
        {
            if (invoker.heatCounter >= 40)
                return ResolveState("Cooldown");
            invoker.heatCounter++;
            A_FirePlasma();
            return ResolveState(null);
        }
        TNT1 A 0 A_ReFire;
        goto Ready;
    Cooldown:
        PLSG B 50 
        {
            A_StartSound("weapons/plasma/cool");
            invoker.heatCounter = 0;
        }
        goto Ready;
    }
}
```

The way this is coded now, the heat will decay at all times, including while the weapon is deselected, but not while it's firing. There are various changes you can add to it, of course. For example, you can lock the heatcounter decay so that it happens only while the weapon *is* inside a specific state, or you can add a check for the `owner.player.readyweapon` field to make sure the heat can't decay while the weapon is not selected—any of those options may work, depending on how you'd like to design this feature.

### PSprite offsets

##### Native offset functions

All weapon sprites can be moved around and offset. These offsets are added on top of the offsets embedded into the sprite itself, which can be edited with SLADE.

The basic way to do that is by using the `offset` keyword, as it is used, for example, in the [Fighter's Fist from Hexen](https://zdoom.org/wiki/Classes:FWeapFist):

```csharp
// This sequence is entered after every 3rd punch,
// gradually moving the fist sprite down and to the right:
    Fire2:
        FPCH DE 4 Offset (5, 40);
        FPCH E 1 Offset (15, 50);
        FPCH E 1 Offset (25, 60);
        FPCH E 1 Offset (35, 70);
        FPCH E 1 Offset (45, 80);
        FPCH E 1 Offset (55, 90);
        FPCH E 1 Offset (65, 100);
        FPCH E 10 Offset (0, 150);
        Goto Ready;
```

However, a much more flexible method to do that is [`A_WeaponOffset`](https://zdoom.org/wiki/A_WeaponOffset): this function allows modifying offsets more smoothly, and also allows doing it additively.

`A_OverlayOffset` basically works almost the same way as [`A_WeaponOffset`](https://zdoom.org/wiki/A_WeaponOffset), except it allows offsetting any sprite layer (whereas `A_WeaponOffset` only interacts with PSP_Weapon).

There are some basic rules to take into account when using it:

* By default, overlays automatically follow the offets of the main weapon layer, meaning `A_OverlayOffset`'s offsets will be added on top of PSP_Weapon's offsets. This also means that by default `A_WeaponOffset` will not only move the main layer, but also all overlays. This behavior can be changed by setting `PSPF_AddWeapon` flag to false by calling `A_OverlayFlags` on the relevant layer. 

* Note that the base offsets of the PSP_Weapon layer of a ready-to-fire weapon is not (0, 0) but (0, 32). The first number is horizontal offset (from left to right), while the second one is vertical offset (from top to bottom—i.e. positive numbers will move it further *down*). Don't ask why the numbers work like that, it's some kind of a legacy decision. What's important is that PSP_Weapon's base offsets are (0, 32), but since overlays are attached on top of the main layer by default, their offsets are (0, 0). If you set `PSPF_AddWeapon` flag of an overlay to `false`, the sprite will be immediately moved up 32 pixels, since it'll become detached from the weapon. If you want to line it up with PSP_Weapon at that point, you'll have to call `A_OverlayOffset(<layer>, 0, 32)` first.

* Another thing to keep in mind is that PSP_Weapon layer will also bob while the weapon is calling `A_WeaponReady` and the player is moving. Overlays will follow the same bob by default (this can be changed by setting `PSPF_AddBob` flag to false). Bob and offsets are separate and unrelated values.

`A_OverlayOffset` and `A_WeaponOffset` are pretty powerful functions that can be used to move sprites around for some nice effects. Taking into account all of the above, let's look at our angled pistol again and add some function-based recoil. There are several ways to do it. First, we could specifcy all offsets for every frame manually:

```csharp
class PistolAngled : Pistol
{
    enum PALayers
    {
        PSP_MFlash      = -10,
        PSP_Hlights     = 10
    }

    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(PSP_MFlash, "Flash");
            A_Overlay(PSP_Hlights, "Highlights");
            A_WeaponOffset(2, 34);
        }
        PGUN D 2 A_WeaponOffset(10,42);
        PGUN C 3 A_WeaponOffset(7,39);
        PGUN B 3 A_WeaponOffset(5,36);
        PGUN A 3 A_WeaponOffset(2,34);
        PGUN A 5
        {
            A_WeaponOffset(0,32);
            A_ReFire();
        }
        goto Ready;
    Flash:
        PGUF Z 2 bright A_Light1;
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        loop;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

But this isn't particularly efficient or good-looking. To make it smoother and simpler we need to utilize the flags available to `A_WeaponOffset`:

* `WOF_INTERPOLATE` — adds interpolation to the layer movements, so that even if the frames are several tics long, the offset interpolation will be applied more or less every tic.

* `WOF_ADD` — makes the offsets relative, which means we don't have to provide specific values. It also implies interpolation, so you don't have to add `WOF_INTERPOLATE` if you're already using this flag.

With these two, let's optimize:

```csharp
class PistolAngled : Pistol
{
    enum PALayers
    {
        PSP_MFlash    = -10,
        PSP_Hlights   = 10
    }

    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(PSP_MFlash, "Flash");
            A_Overlay(PSP_Hlights, "Highlights");
            A_WeaponOffset(2, 2, WOF_ADD); //light offset on the first frame of shooting
        }
        // Immediate strong recoil as the slide moves back:
        PGUN D 2 A_WeaponOffset(10, 10, WOF_ADD);
        // Gradually return to the original position:
        PGUN CBA 3 A_WeaponOffset(-3, -3, WOF_ADD);
        PGUN A 5
        {
            // Restore the base offsets before making the weapon
            // ready for refiring. Apply interpolation so that
            // the offsets are restored smoothly:
            A_WeaponOffset(0, 32, WOF_INTERPOLATE);
            A_ReFire();
        }
        goto Ready;
    Flash:
        PGUF Z 2 bright A_Light1;
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        loop;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

You can use `A_OverlayOffset` to modify the offsets of both the current layer and another layer. For example, let's say you want to slightly randomize the position of your pistol's muzzle flash to make it look a little more varied. You can do this:

```csharp
class PistolAngled : Pistol
{
    enum PALayers
    {
        PSP_MFlash    = -10,
        PSP_Hlights   = 10
    }

    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            // Create the flash:
            A_Overlay(PSP_MFlash, "Flash");
            // Randomize the position of the flash within
            // a 3x3 square area:
            A_OverlayOffset(PSP_MFlash, frandom(-1.5, 1.5), frandom(-1.5, 1.5));
            A_Overlay(PSP_Hlights, "Highlights");
            A_WeaponOffset(2, 2, WOF_ADD);
        }
        PGUN D 2 A_WeaponOffset(10, 10, WOF_ADD);
        PGUN CBA 3 A_WeaponOffset(-3, -3, WOF_ADD);
        PGUN A 5
        {
            A_WeaponOffset(0, 32, WOF_INTERPOLATE);
            A_ReFire();
        }
        goto Ready;
    Flash:
        PGUF Z 2 bright A_Light1;
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        loop;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

> *Note:* You don't need to add 32 to the vertical offset of the muzzle flash, since overlay offsets are automatically added on top of the main layer offsets.

You can, of couse, do the same from the Flash sequence itself instead:

```csharp
    Flash:
        PGUF Z 2 bright 
        {
            // Modify the offsets of the current layer:
            A_OverlayOffset(OverlayID(), frandom(-1.5, 1.5), frandom(-1.5, 1.5));
            A_Light1();
        }
        TNT1 A 0 A_Light0;
        stop;
```

> *Note:* Remember that functions are called at the start of a state, *before* the animation is displayed. So, you can safely set the offsets within the first frame, you don't need to add an extra empty state before it.

You can use whichever method works best for you.

In an earlier subsection we added an overlay-based punch attack to the pistol, which can be used at any point during the pistol's animation. Also, we added some `A_WeaponOffset`-based recoil to the pistol's Fire animation. As a result, if you use the punch while firing the pistol, the fist will be moved down and to the right, along with the main weapon layer, which looks a bit strange. If we unset `PSPF_AddWeapon`, we can get rid of this nuicance:

```csharp
class PistolAngledFist : Pistol
{
    enum PALayers
    {
        PSP_Fist        = -15,
        PSP_MFlash      = -10,
        PSP_Hlights     = 10
    }

    // Allow using the punch sequence at any point
    // during the pistol's animation:
    override void DoEffect()
    {
        super.DoEffect();
        if (!owner || !owner.player)
            return;
        if (!owner.player.readyweapon || owner.player.readyweapon != self)
            return;
        let psp = owner.player.FindPSprite(PSP_Weapon);
        if (!psp)
            return;
        if (owner.player.cmd.buttons & BT_ALTATTACK)
        {
            let pspf = owner.player.FindPSprite(PSP_Fist);
            if (!pspf)
            {
                owner.player.SetPSprite(PSP_Fist, ResolveState("Punch"));
            }
        }
    }

    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(PSP_MFlash, "Flash");
            A_Overlay(PSP_Hlights, "Highlights");
            A_WeaponOffset(2, 2, WOF_ADD);
        }
        PGUN D 2 A_WeaponOffset(10, 10, WOF_ADD);
        PGUN CBA 3 A_WeaponOffset(-3, -3, WOF_ADD);
        PGUN A 5
        {
            A_WeaponOffset(0,32, WOF_INTERPOLATE);
            A_ReFire();
        }
        goto Ready;
    Punch:
        PUNG B 4
        {
            // Disable the flag, so that this layer
            // no longer follows the main layer:
            A_OverlayFlags(OverlayID(), PSPF_AddWeapon, false);
            // Remember to set the offsets to (0, 32),
            // otherwise the layer will be positioned
            // too high:
            A_OverlayOffset(OverlayID(), 0, 32);
        }
        PUNG C 4 A_CustomPunch(2, meleesound: "*fist");
        PUNG D 5;
        PUNG CB 4;
        stop;
    Flash:
        PGUF Z 2 bright A_Light1;
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        loop;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

Nice!

Now, since we're at it, let's use this opportunity to animate the fist separately. First, since we're using the vanilla Fist sprites, the left hand is moved too close to the pistol, as if we're shooting our own hand. And he animation itself isn't particularly smooth, which could be alleviated by creating more frames with differnet offsets and generally changing the pace of the animation. 

Let's use `A_OverlayOffset` to do all of that, similarly to how we animated the pistol recoil earlier:

```csharp
class PistolAngledFist : Pistol
{
    enum PALayers
    {
        PSP_Fist        = -15,
        PSP_MFlash      = -10,
        PSP_Hlights     = 10
    }

    override void DoEffect()
    {
        super.DoEffect();
        if (!owner || !owner.player)
            return;
        if (!owner.player.readyweapon || owner.player.readyweapon != self)
            return;
        let psp = owner.player.FindPSprite(PSP_Weapon);
        if (!psp)
            return;
        if (owner.player.cmd.buttons & BT_ALTATTACK)
        {
            let pspf = owner.player.FindPSprite(PSP_Fist);
            if (!pspf)
            {
                owner.player.SetPSprite(PSP_Fist, ResolveState("Punch"));
            }
        }
    }

    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(PSP_MFlash, "Flash");
            A_Overlay(PSP_Hlights, "Highlights");
            A_WeaponOffset(2, 2, WOF_ADD);
        }
        PGUN D 2 A_WeaponOffset(10, 10, WOF_ADD);
        PGUN CBA 3 A_WeaponOffset(-3, -3, WOF_ADD);
        PGUN A 5
        {
            A_WeaponOffset(0,32, WOF_INTERPOLATE);
            A_ReFire();
        }
        goto Ready;
    Punch:
        TNT1 A 0
        {
            A_OverlayFlags(OverlayID(), PSPF_AddWeapon, false);
            // We start much further to the left and much 
            // lower (below the screen) than the vanilla fist:
            A_OverlayOffset(OverlayID(), -72, 56);
        }
        // 4 frames of rapid movement to the right and up:
        PUNG BBCC 1 A_OverlayOffset(OverlayID(), 10, -3, WOF_ADD);
        // Do the punch:
        TNT1 A 0 A_CustomPunch(2, meleesound: "*fist");
        // 4 frames of "overshoot" animation, showing 
        // minor movement to the right and upward:
        PUNG DDDD 1 A_OverlayOffset(OverlayID(), 4, -3, WOF_ADD);
        // Go back to the right and down:
        PUNG CCCC 1 A_OverlayOffset(OverlayID(), -4, 2, WOF_ADD);
        // Faster rightward and downward movement:
        PUNG BBBBB 1 A_OverlayOffset(OverlayID(), -6, 4, WOF_ADD);
        stop;
    Flash:
        PGUF Z 2 bright A_Light1;
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        loop;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

You may notice that I've added some TNT1 frames, which I hadn't done before. This is because I wanted to apply offsets to every visual frame of the animation, so it was simply easier to call the animation-related functions on the non-zero frames, while calling the attack and initial offset function on separate frames. In short, whenever I heavily utilize offset and other overlay functions for animation, it's usually easier to split the animation functions and the mechanical functions.

##### Internal offset values

As mentioned in the [PSprite properties](#psprite-properties) subsection, PSprite offsets are just its properties. If you get a pointer to PSprite via something like `let psp = player.FindPSprite(<layer>)`, you'll get access to `psp.x` and `psp.y` which are the current layer coordinates. However, use cases for this are fairly niche; normally using offset functions is simpler and shorter. Here's a comparison using our Flash state sequence example from above:

```csharp
    Flash:
        PGUF Z 2 bright 
        {
            // Get a pointer to the current layer:
            let psp = player.FindPSprite(OverlayID());
            // Null-check the pointer:
            if (psp)
            {
                // Set the offsets:
                psp.x = frandom(-1.5, 1.5);
                psp.y = frandom(-1.5, 1.5);
            }
            A_Light1();
        }
        TNT1 A 0 A_Light0;
        stop;
```

As you can see, it's a lot longer and not particularly better in terms of readability. It also doesn't offer interpolation in contrast to `A_WeaponOffset`/`A_OverlayOffset`.

So, as I said, use cases for this are fairly niche. One example that comes to mind is when you want to offset two layers relative to each other but still independently, but it's hard to come up with a generic use for that.

### Overlay scale, rotate and pivot

As of GZDoom 4.5.0, sprite layers can be rotated and scaled. This can be done vial the following methods:

* [`A_OverlayScale(<layer>, <x scale>, <y scale>, <flags>)`](https://zdoom.org/wiki/A_OverlayScale) — modifies the vector2 `scale` field of the corresponding PSprite layer, allowing to change its visual scale, similarly to how actor scale works. It's important to note that it it supports the `WOF_ADD` flag among others, which allows changing the scale additively.

* [`A_OverlayRotate(<layer>, <angle>, <flags>)`](https://zdoom.org/wiki/A_OverlayRotate) — modifies the double `rotation` field of the corresponding PSprite layer, changing its visual angle (similarly to how modifying the `roll` field works on actors that have the +ROLLSPRITE flag). Negative numbers rotate the layer clockwise, positive — counter-clockwise. It also supports the `WOF_ADD` flag, which allows changing the angle additively.

Note that rotation and scale is normally done relative to the sprite's top left corner. But this pivot point can be changed using the following functions:

* [`A_OverlayPivotAlign(<layer>, <x alignment>, <y alignment>)`](https://zdoom.org/wiki/A_OverlayPivotAlign) — modifies the int `HAlign` and `VAlign` fields of the corresponding PSprite layer, moving the pivot point. The arguments are integers, but constant aliases are normally used:
  
  * Horizontal: `PSPA_LEFT` (default), `PSPA_CENTER`, `PSPA_RIGHT`
  
  * Vertical: `PSPA_TOP` (default), `PSPA_CENTER`, `PSPA_BOTTOM`.

* [`A_OverlayPivot(<layer>, <x offset>, <y offset>, <flags>)`](https://zdoom.org/wiki/A_OverlayPivot) — modifies the vector2 `pivot` field of the corresponding PSprite layer, also moving the pivot point using the double offset values. If combined with `A_OverlayPivotAlign`, these values will be added on top of it. The values specified here depend on the value of the `PSPF_PivotPercent` (internally `bPivotPercent`) flag of the specified PSprite layer:
  
  * If `PSPF_PivotPercent` is true (which is the default value): `A_OverlayPivot` arguemnts are used as scalar coordinates, going from 0.0 to 1.0, which is left to right for the horizontal offset, and top to bottom for the vertical offset. (I.e. `0, 0` is top left, `0.5, 0.5` is center, `1.0, 1.0` is bottom right, etc.)
  
  * If `PSPF_PivotPercent` is false: the arguments are used as coordinates in pixels, similarly to how offsets work in `A_WeaponOffset`/`A_OverlayOffset`. For example, for a 16x16 pixels graphic, `8, 8` will be its center, `16, 16` will be bottom right.

To elaborate on these two functions: both `A_OverlayPivotAlign` and `A_OverlayPivot` can be used to move around the pivot point. However, `A_OverlayPivot`'s values are added *on top* of `A_OverlayPivotAlign`'s values, and they allow for higher precision (e.g. you can move the pivot point 60% to the right by using 0.6, which you can't do with `A_OverlayPivotAlign`). `A_OverlayPivot` also allows putting the pivot point *outside* of the graphic (by using values that are outside of the 0.0–1.0 range). 

Most of the time using both functions is overkill. Personally I find it more intuitive to rely on `A_OverlayPivot` only and its scalar coordinates.

To illustrate all of the above, let's utilize these functions to add some variety to our pistol's muzzle flash. Since the muzzle flash is visually round and is placed under the pistol, we can freely randomize its angle and slightly randomize its scale to give it a different appearance:

```csharp
class PistolAngled : Pistol
{
    enum PALayers
    {
        PSP_MFlash        = -10,
        PSP_Hlights     = 10
    }

    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(PSP_MFlash, "Flash");
            A_Overlay(PSP_Hlights, "Highlights");
            A_WeaponOffset(2, 2, WOF_ADD);
        }
        PGUN D 2 A_WeaponOffset(10, 10, WOF_ADD);
        PGUN CBA 3 A_WeaponOffset(-3, -3, WOF_ADD);
        PGUN A 5
        {
            A_WeaponOffset(0,32, WOF_INTERPOLATE);
            A_ReFire();
        }
        goto Ready;
    Flash:
        PGUF Z 2 bright 
        {
            A_Light1();
            // Set the pivot point to the center of the sprite:
            A_OverlayPivot(OverlayID(), 0.5, 0.5);
            // Scale it randomly between 85% and 100% of its
            // size (using 0 for the second argument makes it
            // use the same value as the first one):
            A_OverlayScale(OverlayID(), frandom(0.85, 1), 0);
            // Rotate the sprite randomly:
            A_OverlayRotate(OverlayID(), frandom(0,360));
        }
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        wait;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

> *Note:* Instead of `A_OverlayPivot(OverlayID(), 0.5, 0.5);` you can also use `A_OverlayPivotAlign(OverlayID(), PSPA_CENTER, PSPA_CENTER)`, but I'm used to relying to only the first function.

Now our muzzle flash will rotate and change size, which will give it some nice variety. It's especially useful on automatic weapons, since appearing identical muzzle flash can look pretty boring. 

Let's do the same thing using ZScript pointers, and also throw in the randomization of the `bFlip` flag, so that it occasionally gets flipped on the X axis:

```csharp
class PistolAngled : Pistol
{
    enum PALayers
    {
        PSP_MFlash        = -10,
        PSP_Hlights     = 10
    }

    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(PSP_MFlash, "Flash");
            A_Overlay(PSP_Hlights, "Highlights");
            A_WeaponOffset(2, 2, WOF_ADD);
        }
        PGUN D 2 A_WeaponOffset(10, 10, WOF_ADD);
        PGUN CBA 3 A_WeaponOffset(-3, -3, WOF_ADD);
        PGUN A 5
        {
            A_WeaponOffset(0,32, WOF_INTERPOLATE);
            A_ReFire();
        }
        goto Ready;
    Flash:
        PGUF Z 2 bright 
        {
            A_Light1();
            // Get a pointer to the current PSprite:
            let psp = player.FindPSprite(OverlayID());
            if (psp)
            {
                psp.pivot = (0.5, 0.5); //set pivot
                // Get a random value between 0.85 and 1:
                double s = frandom(0.85, 1);
                psp.scale = (s, s); //set scale
                psp.rotation = frandom(0, 360); //set angle
                // We can't literally randomize 'true'
                // and 'false' values in ZScript, but
                // since they are 0 and 1 internally,
                // we can use random(0,1) to set or unset
                // the flag's value:
                psp.bFlip = random(0, 1);
            }
        }
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        wait;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

Very good.

Let's take this to the next level. Right now our pistol's recoil is very linear, but we can add some angle and scale change within the code.

In practice doing this requires some trial and error, since sometimes it's difficult to figure out which numbers will produce the desired effect. But I already have an example for you that, I think, looks pretty good:

```csharp
class PistolAngled : Pistol
{
    enum PALayers
    {
        PSP_MFlash        = -10,
        PSP_Hlights     = 10
    }

    States
    {
    Ready:
        PGUN A 1 A_WeaponReady;
        loop;
    Fire:
        PGUN A 2
        {
            A_FireBullets(5.6, 0, 1, 5);
            A_PlaySound("weapons/pistol", CHAN_WEAPON);
            A_Overlay(PSP_MFlash, "Flash");
            A_Overlay(PSP_Hlights, "Highlights");
            A_WeaponOffset(2, 2, WOF_ADD);
            // Start by setting the pivot point. For this one
            // I moved it half-way to the right and all the way
            // down, since the handle would move much less
            // than the barrel:
            A_OverlayPivot(OverlayID(), 0.5, 1);
        }
        // This frame is the furthest point of the gun's
        // recoil, so here the gun will get its highest
        // angle and scale values:
        PGUN D 2 
        {
            A_WeaponOffset(10, 10, WOF_ADD);
            // Rotate the gun 8 degrees to the right
            // (we're using additive rotation):
            A_OverlayRotate(OverlayID(), -8, WOF_ADD);
            // Slightly scale up the gun (also additively).
            // It's better to avoid large values here, since
            // the pixelization will become too obvious:
            A_OverlayScale(OverlayID(), 0.2, 0.2, WOF_ADD);
        }
        // These 3 frames are the return movement:
        // it's much smoother and slower than the initial
        // 1-frame recoil was:
        PGUN CBA 3 
        {
            A_WeaponOffset(-3, -3, WOF_ADD);
            // Slowdly reduce the angle:
            A_OverlayRotate(OverlayID(), 2.2, WOF_ADD);
            // Slowly reduce the scale:
            A_OverlayScale(OverlayID(), -0.06, -0.06, WOF_ADD);
        }
        PGUN A 5
        {
            // Now explicitly (not additively) reset the angle:
            A_OverlayRotate(OverlayID(), 0, WOF_INTERPOLATE);
            // Reset the scale the same way:        
            A_OverlayScale(OverlayID(), 1, 1, WOF_INTERPOLATE);
            A_WeaponOffset(0,32, WOF_INTERPOLATE);
            A_ReFire();
        }
        goto Ready;
    Flash:
        PGUF Z 2 bright 
        {
            A_Light1();
            let psp = player.FindPSprite(OverlayID());
            if (psp)
            {
                psp.pivot = (0.5, 0.5);
                double s = frandom(0.85, 1);
                psp.scale = (s, s);
                psp.rotation = frandom(0, 360);
                psp.bFlip = random(0, 1);
            }
        }
        TNT1 A 0 A_Light0;
        stop;
    Highlights:
        PGUF A 2 bright;
        stop;
    Select:
        PGUN A 1 A_Raise;
        wait;
    Deselect:
        PGUN A 1 A_Lower;
        loop;
    }
}
```

This gives us a nice, pretty natural-looking recoil movement—most of it is done via code!

There are a few points you need to keep in mind here:

* Always remember to reset scale and rotation *before* the gun calls `A_Refire` or `A_WeaponReady`, since those functions allow the gun to be refired or switched, and the PSprite's angle and scale won't reset when they're called unless you've done it manually.

* This is more of an animation principle but it's important to remember it: recoil is fast, while the return movement is fairly slow. The ratio is generally around 1:2 or even 1:3. Here we have 2 tics of the shooting frame (which you could bring down to 1, for fast firing speed), then 2 tics of recoil, followed by 6 tics of return movement.

* Be careful with the values, so that you don't over- or underdo the scale or rotation.

### Overlay translation

PSprites support translation, similarly to actors. It can be modified in 3 ways:

- `A_OverlayTranslation(<layer>, "<translation name>")` — this native function will set the sprite layer's translation to a specific named translation from the TRNSLATE lump.

- `Translate.GetID("<translation name>")` is an internal function that retrieves a specified translation from the TRNSLATE lump. If you have pointer `psp` to a PSprite instance, you can call `psp.translation = Translate.GetID("NAME")` where "NAME" is the name from TRNSLATE.

- By copying translation from another pointer—which can be either another PSprite or even an actor (actor translations are handled the same way, since in both cases it's just a matter of remapping colors of an image). I.e. `psp.translation = anotherPointer.translation` where "anotherPointer" is a pointer to another PSprite instance or an actor.

For a very simple example, let's take our PlasmaShotgun we used earlier and make sure its muzzle flash is correctly colorized:

**TRNSLATE:**

```css
PlasmaFlash = "0:255=%[0.00,0.07,0.94]:[0.49,1.32,2.00]"
```

**ZScript:**

```csharp
class PlasmaShotgun : Shotgun
{
    States
    {
    Fire:
        SHTG A 3;
        SHTG A 7 
        {
            // Consume 1 shell:
            invoker.DepleteAmmo(invoker.bAltFire, true);
            // Execute A_FireProjectile 8 times
            for (int i = 8; i > 0; i--)
            {
                A_FireProjectile("Plasmaball", angle: frandom(-5.6, 5.6), useammo: false);
            }
            A_GunFlash();
        }
        SHTG BC 5;
        SHTG D 4;
        SHTG CB 5;
        SHTG A 3;
        SHTG A 7 A_ReFire;
        Goto Ready;
    Flash:
        // Apply the translation and go to the parent Flash:
        TNT1 A 0 A_OverlayTranslation(OverlayID(), "PlasmaFlash");
        goto super::Flash;
    }
}
```

And here's an example of how the PSprite translation can be modified from outside:

**TRNSLATE:**

```css
PlasmaOrb_Green = "192:207=112:127"
PlasmaOrb_Orange = "192:207=208:223"
PlasmaOrb_Red = "192:207=169:191"
PlasmaOrb_Purple = "192:207=250:254"
PlasmaOrb_Pink = "192:207=16:31"
```

**ZScript:**

```csharp
// Note, this specifically replaces the vanilla Plasmaball:
class ColorfulPlasmaball : Plasmaball replaces Plasmaball
{
    // A static array of translation names:
    static const name PlasmaColors[] =
    {
        'PlasmaOrb_Green',
        'PlasmaOrb_Orange',
        'PlasmaOrb_Red',
        'PlasmaOrb_Purple',
        'PlasmaOrb_Pink'
    };

    // Called right after spawning:
    override void PostBeginPlay()
    {
        super.PostBeginPlay();
        // Do nothing if there's no shooter
        // or the shooter isn't a player:
        if (!target || !target.player)
            return;
        // Change the projectile's translation to
        // a random one from the static array above:
        A_SetTranslation(PlasmaColors[random(0,4)]);
        // Get a pointer to the PSprite at the
        // muzzle flash layer:
        let psp = target.player.FindPSprite(PSP_Flash);
        // Null-check it:
        if (!psp)
            return;
        // Copy the translation from this projectile
        // to the PSprite:
        psp.translation = translation;
    }
}
```

This orb will replace the vanilla Plasmaball (do remember that you can handle replacements not only with a `replaces` keyword but also [with an event handler](11_Event_Handlers.md#actor-replacement-via-event-handlers)). It uses a static array (these will be covered in detail in the [Arrays](13_Arrays.md) chapter) to hold a list of translation names and picks one randomly when the orb is spawned. After that it finds the Flash PSprite on the shooter's weapon and sets its translation to the same value.

Basically this is the inverse of the earlier method.

------

🟢 [<<< BACK TO START](README.md)

🔵 [<< Previous: Player, PlayerInfo and PlayerPawn](12.0_Player.md)        🔵 [>> Next: Arrays](13_Arrays.md)
