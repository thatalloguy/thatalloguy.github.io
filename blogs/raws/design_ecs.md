




### Why choose ECS over EC?
Originally i wanted to use ECS from the start for catos. However after finishing Atomus and the Reflection system I choose EC.

```cpp
/// sudo code of how an component would function in an EC env.
class TransformComponent : Component {

		public:
			void init() override {};
			void update() override{};
			void destroy() override{};
		private:
			Vec3 position;
}

```

Now I really liked this syntax (and still do), but when starting on my scripting implementation I realized that I really disliked how to whole flow of: cashing a different component to use later, in the update function actually do logic.

#### EC is error prone (IMO)
Lets say you made an component for your player, in the `init` function you would need to cash **all** the components which you would need to do logic for the player. Now first off all you would need to check if the entity even has the components needed for the player component.
Now you could just do this check in the `init` function, but what if a component gets removed from the entity? Then you would be asking for data from a component that doesn't even exist!
Now you could go and give **every** entity the same base data, such as transforms, rigid bodies, etc (as seen in [cave engine](https://unidaystudio.github.io/CaveEngine-Docs/PythonAPI/Components/transform/) ). But what if you want an entity that does not need all of that data, that would just be a waste of memory! 


###  So what does ECS do different?
To put it really simple: `ECS splits the data and logic of a component from the EC design.`

So instead of doing what you would do with EC you would do:
```cpp
// I use a struct here because I prefer using structs for structures that only hold fields.
struct TransformComponent {
	Vec3 position;
}

class System {
	public:
		void update(TransformCompont& tr, OtherComponent& o);
}
```

Now after defining the `TransformComponent` you can create different systems that loop over different components.

Lets go back to that example about the player component. Here you could just create a new system, then tell the scene that you would like to iterate over components like the rigid-body and transform component. Then when the scene is updated your system gets every entity that has a rigid-body and transform component. So now if a component gets removed that your system needs you would not get any issues since the scene does not pass the entity to your system since it does not meet your given requirements. 

now this does **not** mean that ECS is better then EC. In fact I would argue that in allot of cases EC is the "*better*" choice, because lets be realistic: if you're just making a simple game you would probably be able to handle checking and managing if components exist or not. And then having to write a system for everything would really be a pain. Now that I've said that I would still like to note that I prefer ECS (both because of the problems it solves and of the workflow it brings).
I love the games that are extremely flexible and I would like my engine to support that!


## Lets design an ECS!

#### My concerns for my ECS implementation.
- What about flow?
So lets say you have a render component and a physics component. How would you tell the ECS that you would want the physics to be first calculated and then the render gets ran.
Now the "*solution*" I came up is relatively simple: that wont happen!
What I mean is that in an update cycle before the ECS gets updated I could just update the Physics engine (which isn't part of the ECS) after the ECS update **then** I call the renderer (which is also not part of the ECS). Now when it comes to components and their order: the dev has to figure it out :) (This also counts for EC, since there you don't have control over it either)
- Performance 
I know that when handling lots of components and entities ECS is faster then EC. But in EC the structure is really simple and easy to program, while with ECS they're many different ways of handling components and how to loop through them. And if I were to write a bad implementation it could end up being slower then EC!


### What Catos needs from ECS.

- Not bound to C++
 Catos has C# scripting (at the time of writing is the C# scripting engine still highly under development so things-might be out dated later on. ). I don't want the ECS the be not compatible with my scripting workflow, nor do I want it to make C# embedding more difficult. 
- Fast (as possible)
 As I said in the performance section I want this to be fast, or better said: " I don't want my ECS to be slow". Now this is very logical as a requirement, but I sometimes when i write big systems I get so focused on making it work that I forgot on making it fast. So here I want to give performance allot of attention.
 - Fit into the family
 Unlike Atomus (the renderer which Catos uses) I don't want to make my ECS a different lib/ecosystem. I want this to work flawless with the other system (which isn't allot right now.). 
 For example: I should fully integrate and use the reflection system.


#### The Workflow!

- The setup phase:
In this phase all of our components and systems should be defined. I would like to support adding new components and system at (mainloop) runtime, but its not a big priority for now.
It would somewhat look like:

```cpp
//Sudo Setup phase code

struct TransformComponent {
	vec3 Position;
	vec4 Rotation;
	vec3 Scale;
}

struct LightComponent {
	vec3 position;
	vec4 color;
	int lightType;
}

class systemExample {
	public:
		static void update(TransformCompont& t, LightComponent& l) {
				l.position = t.Position;
				// other stuff
		}
}
/// World = Scene.
World world;

///Note to self: this could be semi replaced with the help of the reflection system
world.register_component<TransformComponent>();
world.register_component<LightComponent >();

/// Give it the components you want to modify.
world.register_system<TransformComponent, LightComponent>(&systemExample::update);

// entity is just an id
Entity entity = world.new_entity();
```

- The Update phase
Since we already did most of the work in the setup phase, is the update phase is simple.

```cpp
// In the update loop of the application.
world.update(); // all we need ;)
```


### Closing Thoughts
First off all: thank you so much for reading! I don't except anyone to sit through this entire essay which is just an basic explanation of ECS. I mostly wrote this to get what I want and my reasons straight in my head. I would also like to say that I have small to none experience with actually writing an ECS, so lots of choices here are definitely subject to change! I also reckon I will write a new article when I actually finish the ECS with a more technical explanation on how it works behind the scenes.





