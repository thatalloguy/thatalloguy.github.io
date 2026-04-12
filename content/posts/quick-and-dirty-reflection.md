---
title: "Quick and simple reflection in c++"
date: 2026-04-12
draft: false
tags: ["reflection", "c++"]
categories: ["general", "engines"]
---

So I have been hard at work on our new entity system (The Vision) and while working hard on it I ran into the classic problem: Not having reflection in c++.

Now I have already written a reflection system in the past, one for my (now abandoned game engine) [Catos](github.com/thatalloguy/Catos). However that system was written quite some time ago and I would say that while impressive on a personal level (at the time), it is quite flawed.

---


 # The old
In catos registering a class for the reflection system would go as the following.
```c++

catos::Registry registry;
registry.register_class<Foo>("Foo")
    .property("a", &Foo::a, "variable description")
    .property("b", &Foo::b, "variable description");

```
Keen readers might recognize that the api is incredibly similar to the [RTTR](https://www.rttr.org/) api. This was done on purpose and while being fine, I think having to do as little as possible outside of just defining the member is best.

Another big thing I dislike about the old system is the way I stored Properties. You see the properties were stored by name in a std::unordered_map . But the problem was that you can't store a templated class inside of an std::unordered_map (unless you know the template at compile time).

So what I did at the time was actually having 2 classes: 
- Property class that isn't dependent on any templates.
- PropertyImpl class that can have as many templates as it wants.

Then with the use of virtual functions we can call functions of the PropertyImpl while only having access to a Property. Now this does have the drawback of virtual functions not being compatible with templates, but with a couple of void pointers here and there you can get around it.

---

# The goal

Now let's say that we have the following struct

```c++
struct Foo {
    float a;
    unsigned char b;
}
```

So let's say for this scenario we would like to access the variable b. Without actually having direct access to the struct at runtime. So we would like to just get b and give a void* which is the instance of Foo.

First let's take a look at how Foo would be structured in memory.

![Struct Foo](/images/struct_foo_float_uchar_layout.svg)

We can see that it first takes 4 bytes for the float a. And finally it takes 1 byte for unsigned char b.
Now because we are working here with c++ we have access to pointers, which are an incredibly powerful tool that we will use. 

You see as long as we know the offset and size of a member, all we really need is the base pointer.
That base pointer will give us an address in memory, to which we can simply add our offset and voilà we have data.
```c++
template<typename T>
T* Get(void* instance, size_t offset) {
    return static_cast<T*>(((char*) instance) + offset);
}
```

---

# My simple approach

So another one of my goals is to have the user have to type as little as possible while still having their structs and classes reflected.

So first I need something to store an offset and size of a struct member in. For this I chose the following simple struct:
```c++
struct PropInfo {
    long long offset = 0;
    size_t size = 0;
};
```
##### *A little fun fact is that I right now don't actually make use of the size variable, but I can imagine that it will come in handy in the future.*


So now I need something to both store and access them later. For this I decided a simple struct will do. This does mean that every reflected object **has** to inherit from this struct, but that isn't a problem for my requirements.

```c++
struct Object {
    long long offset = 0;
    std::vector<PropInfo> _infos;

    template<typename T>
    T* Get(int index) {
        return static_cast<T*>(((char*) this) + _infos[index].offset);
    }
}
```

Alright I now have a place to store my properties' information, but now comes the actual tricky part: **registering properties**.


So I needed a way to create a PropInfo and push them into the _infos vector. Which means I have to run logic at definition. And while I don't really know how to do that, I do know a way to get close to it.

You see, C++ introduced this very niche functionality called "Constructors". Which will allow me to execute logic as soon as the struct is constructed.

So we create a simple struct that only houses a constructor with the sole purpose of creating the PropInfo and pushing it to the vector.
It turned out like this:
```c++
template<typename R>
struct Prop {
    Prop(Object* parent) {
        parent->_infos.push_back({
            parent->offset,
            sizeof(R)
        });
        parent->offset += sizeof(R);
    }
};
```

So let's say we would like to register the property: float a. The only thing we have to do is:
```c++
Prop<float> a_info{this};
```

Now having to type this every time we create a variable is pretty boring. So to hide this we use my beloved macros :).
```c++
#define PROPERTY(type, name, value) \
    Prop<type> __##name##_info{this}; \
    type name{value};
```
You see this handy little macro does both the registration and declaration of variable a for us.

So if we would like to access the variable a or b, all we have to do is pass along their index and voilà! Data!
```c++
struct Foo: Object {
    PROPERTY(float, a, 2.0f);           //index - 0
    PROPERTY(unsigned char, b, 3);      //index - 1
}

Foo foo{};
float* foo_a = foo.Get<float>(0);
//foo_a = 2.0f;
```
---

Now I would like to note that in this scenario we still know that foo_a should be a float, but I digress.

---

# The Adder under the grass.

Now the keen reader that is familiar with memory and layout will probably have recognized one of the great pains in my life: **padding**

Now do not worry, the little things we just wrote can actually be modified pretty easily to deal with padding. 
And I would actually argue that these changes are better all around even if padding wasn't an issue.

So the way we calculate the offset right now is simply by just adding the size of each registered property.
This also means that if I have a struct with 3 variables with the middle one not being reflected, my system breaks.

So first we can actually define the variable before defining its PropInfo, this means that we have access to the variable which will greatly help us!

```c++
#define PROPERTY(type, name, value) \
    type name{value}; \
    Prop<type> __##name##_info{this};
```
So now we can actually just calculate the offset inside of the call of the constructor.
We can simply do this by taking the address of our variable and the pointer of our current instance and voilà! Offset!
This means that our macro comes to look like the following:
```c++
#define PROPERTY(type, name, val) \
    type name{val}; \
    Prop<type> __##name##_info{this, \
     (reinterpret_cast<char*>(&name) - reinterpret_cast<char*>(this))};
```

This also means that we can get rid of that nasty offset variable (this also means using less memory!).
Which means our previously made structs come to look like this:
```c++
struct Object {
    std::vector<PropInfo> _infos;
    template<typename T>
    T* Get(int index) {
        return static_cast<T*>(((char*) this) + _infos[index].offset);
    }
};

template<typename R>
struct Prop {
    Prop(Object* parent, long long offset) {
        parent->_infos.push_back({
            offset,
            sizeof(R)
        });
    }
};
```

---

# The end of the beginning

While missing some very much needed features such as accessing variables by name, reflected functions and reflected constructors. This system is a (in my very biased opinion) nice starting point for reflection!

And for what I am trying to achieve here, which is accessing (reflected) variables at runtime with minimal overhead. I think this is just fine. But there are still issues such as the object being reflected everytime its constructed, instead of once. And that the Prop variables also take up memory (not a lot but still). 

I also would like to note that this will be nowhere close to what the reflection system for Nave will actually be.
But for The Vision and its needs, this fits just fine.

At the end of the day, it's all just 0's and 1's