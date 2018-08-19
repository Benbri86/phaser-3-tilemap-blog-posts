# Modular Game Worlds in Phaser 3 (Tilemaps #5) - Matter.js & Collisions

Author: [Mike Hadley](https://www.mikewesthad.com/)

Reading this on GitHub? Check out the [medium post](coming soon).

This is the fifth post (and finale!) in a series of blog posts about creating modular worlds with tilemaps in the [Phaser 3](http://phaser.io/) game engine. In this edition, we'll step up our Matter.js knowledge and create a little puzzle-y platformer:

![](./images/final-demo-smaller-optimized.gif)

_↳ Pushing crates around to avoid spikes and hopping over seesaw platforms_

If you haven't checked out the previous posts in the series, here are the links:

1.  [Static tilemaps & a Pokémon-style world](https://medium.com/@michaelwesthadley/modular-game-worlds-in-phaser-3-tilemaps-1-958fc7e6bbd6)
2.  [Dynamic tilemaps & puzzle-y platformer](https://medium.com/@michaelwesthadley/modular-game-worlds-in-phaser-3-tilemaps-2-dynamic-platformer-3d68e73d494a)
3.  [Dynamic tilemaps & Procedural Dungeons](https://medium.com/@michaelwesthadley/modular-game-worlds-in-phaser-3-tilemaps-3-procedural-dungeon-3bc19b841cd)
4.  [Meet Matter.js](https://medium.com/@michaelwesthadley/modular-game-worlds-in-phaser-3-tilemaps-4-meet-matter-js-abf4dfa65ca1)

Before we dive in, all the source code and assets that go along with this post can be found in [this repository](https://github.com/mikewesthad/phaser-3-tilemap-blog-posts/tree/master/examples/post-5).

## Intended Audience

This post will make the most sense if you have some experience with JavaScript (classes, arrow functions & modules), Phaser and the [Tiled](https://www.mapeditor.org/) map editor. If you don't, you might want to start at the beginning of the [series](https://medium.com/@michaelwesthadley/modular-game-worlds-in-phaser-3-tilemaps-1-958fc7e6bbd6), or continue reading and keep Google, the Phaser tutorial and the Phaser [examples](https://labs.phaser.io/) & [documentation](https://photonstorm.github.io/phaser3-docs/index.html) handy.

Alright, Let's get into it!

## Overview

In the [last post](https://medium.com/@michaelwesthadley/modular-game-worlds-in-phaser-3-tilemaps-4-meet-matter-js-abf4dfa65ca1), we got acquainted with the Matter.js physics engine and played with dropping bouncy emoji around a scene. Now we're going to build on that Matter knowledge and step through building a 2D platformer. We'll learn how collisions work in Matter, get familiar with a plugin that will allow us to nicely watch for collisions in Phaser and then get into the core of the platformer.

## Collisions in Matter

The crux of what we are going to do revolves around handling Matter collisions. If we want to use physics in a game, we need to be able to respond when certain objects collide with one another, e.g. like a player character stepping on a trap door. Since Phaser's implementation of Matter is a thin wrapper around the underlying library, it's worth revisiting our vanilla Matter example from last time to learn about collision detection in Matter. If you are impatient, you _could_ jump ahead two sections to get straight to the platformer. But if you like to understand how something really works - which I think will pay off in the long run - then stick with me here.

Here's what we're aiming for in this section:

![](./images/matter-collisions-optimized.gif)

_↳ The shapes are flickering when they collide with each other and they are turning purple when they hit the floor._

Here's a [CodeSandbox starter project](https://codesandbox.io/s/kw73yy6375) that matches what we did last time. I'd recommend opening that up and coding along there (there's a comment towards the bottom of the file that shows you where to start coding along). The setup is almost exactly the same:

1. We create a renderer and engine.
2. We create some different shaped bodies that will bounce around the world.
3. We add some static bodies - bodies that are unable to move or rotate - to act as the world.
4. We add everything to the world and kick off the renderer & engine loops.

The difference is that we added a new module alias at the top of the file, `Events`:

```js
import { Engine, Render, World, Bodies, Body, Events } from "matter-js";

// Or when using Matter globally as a script:
// const { Engine, Render, World, Bodies, Body, Events } = Matter;
```

`Events` allows us to subscribe to event emitters in Matter. The two events we will play with in this demo are [`collisionStart`](http://brm.io/matter-js/docs/classes/Engine.html#event_collisionStart) and [`collisionEnd`](http://brm.io/matter-js/docs/classes/Engine.html#event_collisionEnd). (See the docs for other [engine events](http://brm.io/matter-js/docs/classes/Engine.html#events).)

```js
Events.on(engine, "collisionStart", event => {
  event.pairs.forEach(pair => {
    const { bodyA, bodyB } = pair;
  });
});

Events.on(engine, "collisionEnd", event => {
  event.pairs.forEach(pair => {
    const { bodyA, bodyB } = pair;
  });
});
```

On each tick of the engine's loop, Matter keeps track of all pairs of objects that just started colliding (`collisionStart`), have continued colliding for multiple ticks (`collisionActive`) or just finished colliding (`collisionEnd`). The events have the same structure. Each provides a single argument - an object - with a `pairs` property that is an array of all pairs of Matter bodies that were colliding. Each `pair` has `bodyA` and `bodyB` properties that give us access to which two bodies collided. Inside of our event listener, we can loop over all the pairs, look for collisions we care about and do something. Let's start by making anything that collides slightly transparent (using the body's [render property](http://brm.io/matter-js/docs/classes/Body.html#property_render)):

```js
Events.on(engine, "collisionStart", event => {
  event.pairs.forEach(pair => {
    const { bodyA, bodyB } = pair;

    // Make translucent until collisionEnd
    bodyA.render.opacity = 0.75;
    bodyB.render.opacity = 0.75;
  });
});

Events.on(engine, "collisionEnd", event => {
  event.pairs.forEach(pair => {
    const { bodyA, bodyB } = pair;

    // Return to opaque
    bodyA.render.opacity = 1;
    bodyB.render.opacity = 1;
  });
});
```

Now we can extend our "collisionStart" to have some conditional logic based on which bodies are colliding.

```js
Events.on(engine, "collisionStart", event => {
  event.pairs.forEach(pair => {
    const { bodyA, bodyB } = pair;

    // Make translucent until collisionEnd
    bodyA.render.opacity = 0.75;
    bodyB.render.opacity = 0.75;

    // #1 Detecting collisions between the floor and anything else
    if (bodyA === floor || bodyB === floor) {
      console.log("Something hit the floor!");

      // The conditional ternary operator is a shorthand for an if-else conditional. Here, we use it
      // to access whichever body is not the floor.
      const otherBody = bodyB === floor ? bodyA : bodyB;
      otherBody.render.fillStyle = "#2E2B44";
    }

    // #2 Detecting collisions between the floor and the circle
    if ((bodyA === floor && bodyB === circle) || (bodyA === circle && bodyB === floor)) {
      console.log("Circle hit floor");

      const circleBody = bodyA === circle ? bodyA : bodyB;
      World.remove(engine.world, circleBody);
    }
  });
});
```

In the first conditional, we check if one of the bodies is the floor, then we adjust the color of the other body to match the floor color. In the second conditional, we check if the circle hit the floor, and if so, kill it. With those basics, we can do a lot in a game world - like checking if the player hit a button, or if any object fell into lava.

https://codesandbox.io/s/yqv0qqjoj9

This approach isn't terribly friendly or modular though. We have to worry about the order of `bodyA` and `bodyB` - was the floor A or B? We also have to have a big centralized function that knows about all the colliding pairs. Matter takes the approach of keeping the engine itself as lean as possible and leaving it up to the user to add in their specific way of handling collisions. If you want to go further with Matter without Phaser, then check out this Matter plugin to that makes collision handling easier by giving us a way to listening to collisions on specific bodies: [dxu/matter-collision-events](https://github.com/dxu/matter-collision-events#readme). When we get to Phaser, we'll similarly solve this with a plugin.

## Simple Collisions in Phaser

Now that we understand how collisions work in Matter, let's use them in Phaser. Before getting into creating a platformer, let's quickly revisit our emoji dropping example from last time:

![](./images/collisions-angry-emoji.gif)

_↳ Dropping emoji like last time, except now they get angry when they collide._

When an emoji collides with something, we'll make it play a short angry face animation. Here is another [starter template](https://codesandbox.io/s/l5ko8wo917) for this section where you can code along. It has a tilemap set up with Matter bodies on the tiles. Note: Phaser versions 3.11 and lower had a bug with Matter's "collisionEnd", but it's patched now in 3.12 and up. The starter project uses 3.12.

When the player clicks on the screen, we drop a Matter-enabled emoji. Last time we used a [`Phaser.Physics.Matter.Image`](https://photonstorm.github.io/phaser3-docs/Phaser.Physics.Matter.Image.html) for the emoji, but this time we'll use a [`Phaser.Physics.Matter.Sprite`](https://photonstorm.github.io/phaser3-docs/Phaser.Physics.Matter.Sprite.html) so that we can use an animation. This goes into our Scene's `create` method:

```js
// Drop some 1x grimacing emoji sprite when the mouse is pressed
this.input.on("pointerdown", () => {
  const worldPoint = this.input.activePointer.positionToCamera(this.cameras.main);
  const x = worldPoint.x + Phaser.Math.RND.integerInRange(-10, 10);
  const y = worldPoint.y + Phaser.Math.RND.integerInRange(-10, 10);

  // We're creating sprites this time, so that we can animate them
  this.matter.add
    .sprite(x, y, "emoji", "1f62c", { restitution: 1, friction: 0.25, shape: "circle" })
    .setScale(0.5);
});

// Create an angry emoji => grimace emoji animation
this.anims.create({
  key: "angry",
  frames: [{ key: "emoji", frame: "1f92c" }, { key: "emoji", frame: "1f62c" }],
  frameRate: 8,
  repeat: 0
});
```

Now we just need to handle the collisions (also in `create`):

```js
this.matter.world.on("collisionstart", event => {
  event.pairs.forEach(pair => {
    const { bodyA, bodyB } = pair;
  });
});

this.matter.world.on("collisionend", event => {
  event.pairs.forEach(pair => {
    const { bodyA, bodyB } = pair;
  });
});
```

The structure is pretty much the same as with native Matter, except that Phaser lowercases the event name to match its own conventions. `bodyA` and `bodyB` are Matter bodies, but with an added property. If the bodies are owned by a Phaser game object (like a Sprite, Image, Tile, etc.), they'll have a `gameObject` property. We can then use that property to identify what collided:

```js
this.matter.world.on("collisionstart", event => {
  event.pairs.forEach(pair => {
    const { bodyA, bodyB } = pair;

    const gameObjectA = bodyA.gameObject;
    const gameObjectB = bodyB.gameObject;

    const aIsEmoji = gameObjectA instanceof Phaser.Physics.Matter.Sprite;
    const bIsEmoji = gameObjectB instanceof Phaser.Physics.Matter.Sprite;

    if (aIsEmoji) {
      gameObjectA.setAlpha(0.5);
      gameObjectA.play("angry", false); // false = don't restart animation if it's already playing
    }
    if (bIsEmoji) {
      gameObjectB.setAlpha(0.5);
      gameObjectB.play("angry", false);
    }
  });
});
```

We've got two types of colliding objects in our scene - sprites and tiles. We're using the [`instanceof`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/instanceof) to figure out which bodies are the emoji sprites. We play an angry animation and make the sprite translucent. We can also use the "collisionend" event to make the sprite opaque again:

```js
this.matter.world.on("collisionend", event => {
  event.pairs.forEach(pair => {
    const { bodyA, bodyB } = pair;
    const gameObjectA = bodyA.gameObject;
    const gameObjectB = bodyB.gameObject;

    const aIsEmoji = gameObjectA instanceof Phaser.Physics.Matter.Sprite;
    const bIsEmoji = gameObjectB instanceof Phaser.Physics.Matter.Sprite;

    if (aIsEmoji) gameObjectA.setAlpha(1);
    if (bIsEmoji) gameObjectB.setAlpha(1);
  });
});
```

https://codesandbox.io/s/810yw745v9

Now we've seen native Matter events and Phaser's wrapper around those Matter events. Both are a bit messy to use without a better structure, but they are important to cover before we start using a plugin to help us manage the collisions. It's especially important if you decide you don't want to rely on my plugin 😉.

The approach in this section still isn't very modular. One function handles all our collisions. If we added more types of objects to the world, we'd need more conditionals in this function. We also haven't looked at how compound bodies would work here - spoiler, they add another layer of complexity.

## Collision Plugin

I created a Phaser plugin to make our lives a bit easier when it comes to Matter collisions in Phaser: [phaser-matter-collision-plugin](https://github.com/mikewesthad/phaser-matter-collision-plugin). We'll use it to build this (last stop before we step up the complexity with a platformer):

![](./images/emoji-love-hate.gif)

_↳ Love-hate collisions_

With the plugin, we can detect collisions between specific game objects, for example:

```js
const player = this.matter.add.sprite(0, 0, "player");
const trapDoor = this.matter.add.sprite(200, 0, "door");

this.matterCollision.addOnCollideStart({
  objectA: player,
  objectB: trapDoor,
  callback: () => console.log("Player touched door!")
});
```

Or between groups of game objects:

```js
const player = this.matter.add.sprite(0, 0, "player");
const enemy1 = this.matter.add.sprite(100, 0, "enemy");
const enemy2 = this.matter.add.sprite(200, 0, "enemy");
const enemy3 = this.matter.add.sprite(300, 0, "enemy");

this.matterCollision.addOnCollideStart({
  objectA: player,
  objectB: [enemy1, enemy2, enemy3],
  callback: eventData => {
    console.log("Player hit an enemy");
    // eventData.gameObjectB will be the specific enemy that was hit
  }
});
```

Or between a game object and any other body:

```js
const player = this.matter.add.sprite(0, 0, "player");

this.matterCollision.addOnCollideStart({
  objectA: player,
  callback: eventData => {
    const { bodyB, gameObjectB } = eventData;
    console.log("Player touched something.");
    // bodyB will be the matter body that the player touched
    // gameObjectB will be the game object that owns bodyB, or undefined if there's no game object
  }
});
```

There are some other useful features - check out [the docs](https://www.mikewesthad.com/phaser-matter-collision-plugin/docs/manual/README.html) if you want to learn more. We'll be using it, and unpacking how it works, as we go.

Phaser's plugin system allows us to hook into the game engine in a structured way and add additional features. The collision plugin is a scene plugin (vs a global plugin, see [docs](https://photonstorm.github.io/phaser3-docs/Phaser.Plugins.PluginManager.html)), so an instance will be accessible on each scene after we've installed it via `this.matterCollision`.

Here's a [CodeSandbox starter project](https://codesandbox.io/s/316pq9j541) for coding along. It has the dependencies - Phaser and PhaserMatterCollisionPlugin - already installed as dependencies. (There are additional instructions [here](https://www.mikewesthad.com/phaser-matter-collision-plugin/docs/manual/README.html#installation) on how to load the plugin from a CDN or install it locally.)

Inside of index.js, we can load up the game with the plugin installed:

```js
import MainScene from "./main-scene.js";

const config = {
  type: Phaser.AUTO,
  width: 800,
  height: 600,
  backgroundColor: "#000c1f",
  parent: "game-container",
  scene: MainScene,
  physics: { default: "matter" },
  plugins: {
    scene: [
      {
        plugin: PhaserMatterCollisionPlugin, // The plugin class
        key: "matterCollision", // Where to store in Scene.Systems, e.g. scene.sys.matterCollision
        mapping: "matterCollision" // Where to store in the Scene, e.g. scene.matterCollision
      }
    ]
  }
};

const game = new Phaser.Game(config);
```

Then inside of main-scene.js, inside of `create`:

```js
// Create two simple animations - one angry => grimace emoji and one heart eyes => grimace
this.anims.create({
  key: "angry",
  frames: [{ key: "emoji", frame: "1f92c" }, { key: "emoji", frame: "1f62c" }],
  frameRate: 3,
  repeat: 0
});
this.anims.create({
  key: "love",
  frames: [{ key: "emoji", frame: "1f60d" }, { key: "emoji", frame: "1f62c" }],
  frameRate: 3,
  repeat: 0
});

const bodyOptions = { restitution: 1, friction: 0, shape: "circle" };
const emoji1 = this.matter.add.sprite(150, 100, "emoji", "1f62c", bodyOptions);
const emoji2 = this.matter.add.sprite(250, 275, "emoji", "1f62c", bodyOptions);

// Use the plugin to only listen for collisions between emoji 1 & 2
this.matterCollision.addOnCollideStart({
  objectA: emoji1,
  objectB: emoji2,
  callback: ({ gameObjectA, gameObjectB }) => {
    gameObjectA.play("angry", false); // gameObjectA will always match the given "objectA"
    gameObjectB.play("love", false); // gameObjectB will always match the given "objectB"
  }
});
```

Now we don't have to worry about which order the colliding pair is in, or finding the attached game object (or dealing with compound bodies). We can organize pieces of our collision logic within classes/modules as we see fit - like having the player listen for collisions that it cares about within player.js.

Here's the final code, with a little extra added in to make the emojis draggable:

https://codesandbox.io/s/v829vxpp8l

## Ghost Collisions

At some point in exploring Matter, you might run into the common problem of ghost collisions. If you notice a player seemingly tripping over nothing as it walks along a platform of tiles, you are likely running into ghost collisions. Here's what they look like:

![](./images/ghost-collision-demo.gif)

You can check out the corresponding live [demo](http://labs.phaser.io/view.html?src=src/game%20objects/tilemap/collision/matter%20ghost%20collisions.js) that I created on Phaser Labs. As the mushrooms move over the top platform, you can see that they catch on the vertical edge of the tiles. That's due to how physics engines resolve collisions. The engine sees the tiles as separate bodies when the mushroom collides against them. It doesn't know that they form a straight line and that the mushrooms shouldn't hit any of the vertical edges. Check out [this article](http://www.iforce2d.net/b2dtut/ghost-vertices) for more information and to see how Box2D solves for this.

There are a couple ways to mitigate this:

- Add chamfer to bodies, i.e. round the edges, or use circular bodies to reduce the impact of the ghost collisions.
- Map out your level's hitboxes as as a few convex hulls instead of giving each tile a separate body. You can still use Tiled for this. Create an object layer, and fill it with shapes, convert those shapes to Matter bodies in Phaser. The demo code does that.
- You might be able to use Matter's ability to handle polygons to join all your tiles into one body, depending on the complexity of your map.

Or, just live with the little glitches for a bit. [@hexus](https://github.com/hexus) is working on a Phaser plugin for solving for ghost collisions against tilemaps, so keep an eye on his GitHub feed. It should be dropping pretty soon.

## Up Next

Thanks for reading, and if there's something you'd like to see in future posts, let me know!

## About Me

I’m a creative developer & educator. I wrote the Tilemap API for Phaser 3 and created a ton of guided examples, but I wanted to collect all of that information into a more guided and digestible format so that people can more easily jump into Phaser 3. You can see more of my work and get in touch [here](https://www.mikewesthad.com/).

```

```