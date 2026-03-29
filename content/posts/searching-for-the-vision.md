---
title: "Searching for The Vision & Entity models"
date: 2026-03-29
draft: false
tags: ["entity models", "engines", "the vision"]
categories: ["general", "engines"]
---


During the time that I have spent writing game engines (or at least the attempts that I've made at writing engines) I have come across a variation of ways to handle entities. Now I am aware that I simply don't know everything about every entity model (even if I would love to know).


Some of the models I have encountered are the following:
- **E**ntity & **C**omponents.
- **E**ntity **C**omponent **S**ystem
- Nodes (From both [Catos](https://github.com/thatalloguy/Catos) & [Godot](https://godotengine.org/))
- Object oriented.

## What even is an Entity?
But first I would like to give my definition for an "Entity"(*or "Game Object" for all of you poor Unity developers out there*). For me an Entity is an object that "lives" inside of your Scene or Game world. Now I know that might not make it any clearer so here is an example, Let's take for example the game [Bomberman](https://wikipedia.org/wiki/Bomberman). You have the player, some enemies and the map. I would define all of these things as entities (there can be some discussion about if the map is one entity or each tile is an entity but I digress).
However something like the system that spawns enemies or a system that loads the map into memory would not be considered as entity, since I could not see any benefit from it.
However one could argue that the map should be split up into dynamic tiles and static tiles (which is something I did for my [Bomberman Remake](https://github.com/thatalloguy/bomberman)), but let's not get stuck on specifics.


---
Now I would like to talk about each "Model" And how I used them and my likes/dislikes of them. 


## Entity & Components
A Model famous from game engines such as [Unity](https://unity.com/), [Unreal](https://www.unrealengine.com/en-US) and (as of writing) [Skore](skoreengine.com). The main idea is that you have a base Entity object/class which holds some data (and logic) that every Entity would need. Then the user can add "Components" to each entity. These components hold extra data (and logic) that not every entity might need.
This way you can customize your entities to only hold/store the logic and data each entity needs and you can even customize their behaviour based on their components. 
Here is an example of what I am talking about:

![Entity & Components](/images/ec_entity_components.svg)

```c++
//Note: this is pseudo code.

class Entity {
public:
    template<typename T>
    void AddComponent(/* You could even take some arguments in here to construct said component*/);

    template<typename T>
    void RemoveComponent();

    void Update();
private:
    //ComponentType be a size_t aslong as it can identify Components via their class.
    HashMap<ComponentType, Component*> _components{};
}

template<typename T>
void AddComponent() {
    Component* component = static_cast<Component*>(new T{});
    _components.insert({GetComponentType<T>(), component});
}

void Entity::Update() {
    // Do update logic
    for (auto& pair : _components) {
        pair.second->Update();
    }
}
```
Then you can have a base Component class which just holds the functions you want to call from inside of the Entity class.
```c++
class Component {
public:
    virtual void Initialize() = 0;
    virtual void Update() = 0;
    virtual void Destroy() = 0;
}
```
And Finally you have your actual component that holds logic and data:
```c++
class TransformComponent: public Component {
public:
    void Initialize() override;
    void Update() override;
    void Destroy() override;

    void SetPosition(const Vector3& vec);
    Vector3 GetPosition();

    void SetScale(const Vector3& vec);
    Vector3 GetScale();
    //etc.
private:
    Vector3 _position;
    Vector3 _scale;
}
```
So now the user would do the following:
```c++
TransformComponent* transform = entity.GetComponent<TransformComponent>();
Vector3 position = transform->GetPosition();

position.y += 20;

transform->SetPosition(position);
```
Now this of course has some positives and negatives. To start with the positives, this provides a very easy way to design gameplay around the idea of writing gameplay for an entity as an individual. 
So let's say you are writing the gameplay for your player, this way you can just have to worry about getting the components you need and modifying them accordingly.
And now the negatives. 
First of all with the usage of virtual functions you lose the speed you would get with a data oriented approach,
since the cache and virtual functions aren't the best of friends (but this isn't something this blog is about so I digress).
Another big downside is that you "treat" everything as an individual. Let's say you are writing the logic for grass (I don't know why you would), you can probably imagine that writing logic for each individual blade of grass would be both a horrible experience for you and your performance. Another big issue is that you can't (most of the time) 100% guarantee that an entity has a component, which means you have to validate if you have the components before being able to use them.

-------

## Entity, Components and Systems (ECS)

A Model certainly not as commonly seen in more general purpose engines such as EC, but a very interesting model nonetheless. It also fixes most issues found in the EC model. The way by which ECS works is by separating Data and Logic.
So this means that an entity can be simply reduced to a single ID and you store their components separately.
For creating logic you create "Systems" which just iterates over entities which hold specific components.


![Entity & Components & Systems](/images/ecs_entity_component_system.svg)


An example from the [ENTT repository](https://github.com/skypjack/entt):
```c++
#include <entt/entt.hpp>

struct position {
    float x;
    float y;
};

struct velocity {
    float dx;
    float dy;
};

void update(entt::registry &registry) {
    auto view = registry.view<const position, velocity>();

    // use a callback
    view.each([](const auto &pos, auto &vel) { /* ... */ });

    // use an extended callback
    view.each([](const auto entity, const auto &pos, auto &vel) { /* ... */ });

    // use a range-for
    for(auto [entity, pos, vel]: view.each()) {
        // ...
    }

    // use forward iterators and get only the components of interest
    for(auto entity: view) {
        auto &vel = view.get<velocity>(entity);
        // ...
    }
}

int main() {
    entt::registry registry;

    for(auto i = 0u; i < 10u; ++i) {
        const auto entity = registry.create();
        registry.emplace<position>(entity, i * 1.f, i * 1.f);
        if(i % 2 == 0) { registry.emplace<velocity>(entity, i * .1f, i * .1f); }
    }

    update(registry);
}
```
You might notice the lack of inheritance which is done on purpose. This way of storing entities and their data is very efficient for the cache which is very nice for performance. It also solves the problem of having to validate if the entity holds the desired component, because you ask for every entity that holds the desired components.
However this approach also brings some issues with itself.
With the main one being the amount of extra baggage it brings.
Let's say you want to create a player, how many components should the player have? That depends of course on the game you are making but I digress. The main issue I have with ECS is that not every "entity" fits nicely into that mold of separating everything into components. This leads to having some gameplay code that is more difficult to write than it should and it also ends up more complicated/over-engineered than it would've been with something like EC.

## Nodes (Composition)
When I am talking about Nodes I am mainly referring to how they are represented in [Godot](https://godotengine.org/).
Each Node holds some data and logic, with each Node being able to hold children nodes.
This is very similar to the way EC does it, but unlike EC where you will always have a base entity, in Nodes any node type can be the "base entity". 
This approach means that you can compose an object/entity consisting of only the data you need (within reason).
So some Entities do not need a physical position inside of the game world, this means that you can have a simple base node that does not store any positional data.
Also unlike EC, (in Godot) each Node can hold their own script/logic, which means that instead of executing logic from a Object/Entity level you are executing it from the Component/Node level.
Another great advantage of that is iterability, you can add some Nodes with predefined logic to another node and make it work simultaneously. 
Sadly the issue of being unable to guarantee the presence of required nodes at runtime is not always possible, which is the same issue that EC suffers from.


![Nodes](/images/nodes_composition_tree.svg)
```c++

class Node {
public:
    virtual void Update(float deltaTime) {
        for (auto& child : _children) {
            child->Update(deltaTime);
        }
    }

    void AddChild(Node* child) {
        child->_parent = this;
        _children.push_back(child);
    }

    Node* GetParent() { return _parent; }

protected:
    Node* _parent{nullptr};
    std::vector<Node*> _children{};
};

//Unlike EC, any node type can be the "base" of your entity.
class TransformNode : public Node {
public:
    void Update(float deltaTime) override {
        // Update world position based on parent, etc.
        Node::Update(deltaTime); // Propagate to children
    }

    Vector3 position{};
    Vector3 scale{1, 1, 1};
};

```


## Object Oriented Programming
While not being exclusively for Entities, object oriented is a commonly used way to represent objects (specifically for more older game titles). 
You can easily have a base Entity class and instead of adding components to an existing entity instance, 
you create a new class, inherit from the base class and then just add the data you need.
This means that you don't suffer (for the most part) from the having to validate required data issue like EC & Nodes.
![OOP](/images/oop_inheritance_hierarchy.svg)
```c++
//Example from my bomberman remake

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

        uint id{ 0 };
        // When the entity gets initialized it itself is responsible for retrieving its desired shader and texture from the Renderer.
        GLTexture* texture{nullptr};
        Shader* shader{nullptr};
        
    };

    //An animated entity is an entity that has multiple keyframes stored in their texture.
    // It also uses a different shader.
    class AnimatedEntity : public Entity {
    public:


        ~AnimatedEntity() override = default;

        void Initialize() override;
        void PrepareRender() override;
        void Destroy() override;
        void Tick(float) override {}

        void SetFrame(const int2& frame);
        float2 GetFrameSize() const { return frameSize; }
        float2 GetCurrentFrame() const { return currentFrame; }
        float4 GetFilterColor() const { return filterColor;  }
    protected:
        float4 filterColor{ 0, 1, 0, 0 };
        float4 secondaryColor{ 0, 0, 0, 1 };
        float4 replaceColor{ 0,0, 1, 1};
        float2 frameSize{ 16, 384 };
        float2 currentFrame{ 1, 1 };
    };

```
Now the disadvantages of Object oriented programming are widely known (I hope), with such great issues as the diamond shape problem (Solved by the Composition pattern) and Virtual functions being bad for memory cache efficiency. Also just the sheer amount of inheritance and a big hierarchy can create large difficult to maintain and work with code base.


## So what is the Vision?
Good question! I don't know :). 
Alright I will be more specific, "The Vision" is how I call the entity model that I am working on.

With the goals being:
- Being fast! (While I am sure that I will not achieve speeds such as ENTT, but that is not the goal, just not be slow).
- Flexibility (Like nodes, using composition to your advantage).
- Solving the validation problem (When writing gameplay code, I just want to execute gameplay logic not having to validate if I have everything).

I will also be targeting a more general purpose model, with ease of adapting to your game idea being chosen over speed (unlike ECS). 
I also want the gameplay code to be able to do what it needs to do and not having to fight the entity model.

-------

Now I am aware that I am still being incredibly vague, but that's because I simply do not yet *see the vision*.
But be sure to tune back for when I release a new blog in which I describe the Vision.