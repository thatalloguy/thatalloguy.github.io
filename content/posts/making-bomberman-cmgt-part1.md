---
title: "Making Bomberman - CMGT part 1"
date: 2026-06-27
draft: false
tags: ["CMGT", "c++", "Buas"]
categories: ["general"]
---


Now that my first year of CMGT (Or ICAD) here at Buas is finished, I would like to go over each block and talk about what I've done.

You can also find all of the code here: https://github.com/thatalloguy/Bomberman


# Overview (of the block)
I would describe block A as a block where you find your footing, see how you handle the study.
In my opinion it's best to focus on actually being productive and passing the block, rather than doing something crazy.

Now the basic premise of the block is: Create a simple arcade game **with the provided template**.
Besides that, we also got some "homework" every friday, which consisted of:
- Simple number guessing game (with the use of std*) *i think.
- That same guessing game but with stars that move down when guessing wrong or explode when the number is guessed.
- Simple breakout game.
- Bombjack (spread out over multiple fridays)

Important to note is that these were required for us to do.

<br>

# The Template
The template we used was [Jacco's Tmpl8](https://github.com/jbikker/tmpl8), which has all of that Jacco's charm.
Some people adore it saying that its "misunderstood", others dislike it.
Personally I think it's fine for what it tries to do. It also has a very specific code style, which I think teaches you on how to adopt to more non-conventional code styles.

It has also has some deliberate limitations and flaws, made for you to overcome.
The template basically provides you with a simple math library (which I liked :) ) and the ability to render sprites.
The template also does its rendering on the CPU, writing the pixels directly into a texture, all stored on the CPU.


---

# Starting
Starting off, I first analyzed the template, drawing the conclusions I wrote above.
My intake assignment had a pretty solid, simple 2D opengl renderer. So I asked wether or not I was allowed to modify the template.
My teacher told me that, while I can modify it, I cannot replace it entirely.

My first couple of weeks were spent on getting the foundation up and running. Stuff like player(s), map rendering and physics. (With physics being quite a pain).

![beginning](/images/bomberman/beginning.png)

# Architecture

![codebase](/images/bomberman/codebase.png)

*My final codebase*

## Asset Manager
Since I'd be needing to load different assets and having multiple references to them, I decided to write a simple form of an AssetManager.
```c++ {title="assetManager.h", open=false} 
#pragma once

#ifndef TEXTURE_AMOUNT
#define TEXTURE_AMOUNT 4
#endif


#ifndef SHADER_AMOUNT
#define SHADER_AMOUNT 6
#endif

namespace bomberman
{
	struct ShaderConfig
	{
		const char* vertPath;
		const char* fragPath;
	};

	enum class TextureType
	{
		TILES	 = 0,
		ARROW	 = 1,
		MENU	 = 2,
		TEXT	 = 3,
	};

	enum class ShaderType
	{
		DEFAULT		= 0,
		ANIMATED	= 1,
		DEBUG		= 2,
		DEPTH		= 3,
		SCREEN		= 4,
		TILE		= 5
	};


	static constexpr const char* texturePaths[4] = {
		"assets/tiles.png",
		"assets/arrow.png",
		"assets/mainscreen.png",
		"assets/text.png"
	};

	static constexpr ShaderConfig shaderPaths[6] = {
		ShaderConfig{"assets/shaders/default.vert", "assets/shaders/default.frag"},
		ShaderConfig{"assets/shaders/animated.vert", "assets/shaders/animated.frag"},
		ShaderConfig{"assets/shaders/debug.vert", "assets/shaders/debug.frag"},
		ShaderConfig{"assets/shaders/default.vert", "assets/shaders/depth.frag"},
		ShaderConfig{"assets/shaders/screen.vert", "assets/shaders/screen.frag"},
		ShaderConfig{"assets/shaders/tile.vert", "assets/shaders/tile.frag"},
	};


	namespace Assets
	{
		bool LoadAssets();
		void UnloadAssets();

		GLTexture* GetTexture(TextureType type);
		Shader* GetShader(ShaderType type);
	}
;
}
```

<br>

First thing I like to point out is: All of the asset path and types are hardcoded.
Now of course you could define these externally, but lets be honest, thats just overengineering for a project of this scope.
Many programmers that come to the study who are already quite familiar with C++ fall into the overengineering trap.
Including myself. 

You will realize that at this project scale that if you have a super cool engineered system, nobody gives a shit and 9/10 times you don't need it.
I will play devils advocate and say that IF you're very passionate about a specific system or aspect and you want too dive deeper into it, go ahead! 
As long as it doesn't come at the cost of the project its all fair game!

Last thing I want to point out is my usage of namespaces.
I use a namespace instead of a class, because all of the functions would be static anyways.
You could say to implement a proper singleton pattern, but I find this implementation cleaner and it works fine for this scenario.


## Audio
Nothing worthy to note, just an incredibly simple layer above the MiniAudio library.

## Entities
For this project, an entity is anything you see on screen.
Previously I've discussed my dislike at the time for many different Entity models, including the "OOP way". 
For this project, the teachers pushed OOP to the students. It even was required in the ILO's to display understanding and great usage of OOP.
Honestly even without the push from the teachers, I would still argue that OOP is best for this project.

```c++ 
class Entity
{
public:
	//Note We the entity do NOT manage the memory of the texture.
	Entity() = default;
	virtual ~Entity() = default;

	virtual void Initialize() = 0;
	virtual void PrepareRender() = 0;
	virtual void Tick(float deltaTime) = 0;
	virtual void Destroy() = 0;

	float2 GetPosition() const { return position;  }
	float2 GetScale() const { return scale; }

	void SetPosition(const float2& newPosition) { position = newPosition;  }
	void SetScale(const float2& newScale) { scale = newScale;  }

	GLTexture* GetTexture() const { return texture; }
	Shader* GetShader() const { return shader;  }

protected:
	float2 position{ 0, 0 };
	float2 scale{ 0, 0 };

	uint id{ 0 }; // NOTE: Unused
	// When the entity gets initialized it itself is responsible for retrieving its desired shader and texture from the Renderer.
	GLTexture* texture{nullptr};
	Shader* shader{nullptr};
	
};
```
This is the base entity class on which every other entity is based on, with a couple of things to note.
First, there is a *PrepareRender* function, here the entity can upload anything it needs to its shader.
Secondly notice my usage of raw pointers, a bad habit that stuck until Block C.

Only a couple of entities inherit directly from the Entity class.
Many inheriting from the AnimatedEntity class, which can (you guessed it!) be animated!
![codebase](/images/bomberman/entityoverview.png)

# Rendering
One of the more special things about my project is the way I render my scene, since im not using the built in template renderer.

The first thing I do when rendering the scene (after clearing the screen) is rendering all of the **static tiles**.
Static tiles are the ones that never change.
I can do this all in 1 drawcall because I have all of the static tiles in 1 batch.

![rendering1](/images/bomberman/rendering1.png)

Afterwards I draw all of the dynamic objects in the scene, starting with every breakable tile.
Important to note: I never did batching for dynamic entities so every entity (including the breakable tiles) is 1 drawcall. I never did this because time :( and other things took priority, also my game was fast enough but still.

![rendering2](/images/bomberman/rendering2.png)

Then I draw the rest of the entities, which are the player(s), enemies, power-ups and (sometimes) bonuses.

![rendering3](/images/bomberman/rendering3.png)

All of the rendering previously isn't actually done to actual screen, instead its done to a framebuffer (texture you can render to).
This is because if the game is played with 2 players, we need to render the scene multiple times.

So finally I render all of the player perspectives to the actual screen and draw the UI on-top of that.

![rendering4](/images/bomberman/rendering4.png)

# Gameplay (programming)
One of my biggest weaknesses: writing clean gameplay code.
Honestly I think I did okay for this project (except the collision code, that was horrible).
I think the reason why it wasn't a complete mess is because I stook to OOP closely.
For example I have a base power-up class, which handles everything from collision to rendering, etc.
And then every different type of power-up inherits from that class, same goes for the enemies and bonuses.

It was also quite nice that the grading was made in a way that you could easily scale up.
In the beginning I chose intermediate, not wanting to bite more off than I could chew.
However I actually ended up going for hard, implementing every enemy, bonus and power-up, because it was so easy to do.

<br>


# Reflection (the boring kind)
At the end of every block, we reflect on what we did, how we felt and what we can do to improve in the future.


Looking back at it, I write about being happy on actually finishing the project, making a finished game.
Also being happy it being actually stable and running.

For the difficult parts of the block I write about some bugs I struggled with.
Also mentioning my dislike towards the LevelManager and Game class, noting them as messy.

For improvement I write about breaking my tasks into smaller tasks (Something I never ended up doing).
Also noting on trying to avoid techdebt because of laziness.

<br>

---

<br>
Anyways, tune in next time when I cover the hell that was block B in part 2!