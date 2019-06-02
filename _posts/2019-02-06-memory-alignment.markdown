---
layout: post
title:  "Dealing with memory alignment in Vulkan and C++"
subtitle: Or in other words, how the mapping of data in your struct can impact what your shader ends up receiving.
subtitleImage: https://i.imgur.com/77LHk83.png
date:   2019-02-06 08:11:00
language: en
image: https://i.imgur.com/77LHk83.png
description: Memory Alignment.
---

Some time ago I decided to learn a bit of Vulkan, and I also took this opportunity to learn a bit better how to work with compute shaders. So my idea was: let's write Vulkan program that sets up a compute shadder, and then let's write a simple Ray Tracer on this one shader and see where that takes us!

My first go at it was to have everything extremely simple, the Vulkan project would only setup all the necessary pumbling to start a compute shader and read back its result (if you ever tried to render something using Vulkan you probably already know it's incredibly verbose and you will need a lot of code even before getting a black window to appear). So the idea was, start a compute shader, have all the code related to the Ray Tracer on the compute shader (so I wasn't sending any sort of data using buffers or anything like that) and read the result back. The only thing that would communicate between the shader and the rest of the code was my description of an image (a matrix of RGBA values, all floats) that used a Storage Buffer.

So nothing incredbly fancy, only one ray per pixel, no anti-aliasign, no model loading, just a really simple and minimal ray tracer (maybe much more verbose than it needed to be, but that's because the point was to describe and explain everything that I am doing). And that worked really well! The code is [available at my GitHub repo](https://github.com/fvcaputo/vulkan-projects/tree/master/Ray%20Tracer%20(hard-coded%20values)), it includes a lot of commentaries so that anyone can follow what I'm doing. With it you can render the following image:

<div align="center">
<a href="https://i.imgur.com/77LHk83.png" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/77LHk83.png" style="width:40%;border-style:solid;border-width:1px"/></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>The ray tracing result.</p></font></div>
</div>

Next step: I wanted to actually send some data to the shader. The plan was to send the spheres data (position, radius, lighting related data) using Uniform Buffers. That's where I ran into problems related to memory alignment.

## Setting up the data

Spheres were already described on my previous implementation of the ray tracer, they were defined as follows:

{% highlight c++ %}
struct Sphere {
    vec3 position;
    float radius;
    vec3 albedo;
    vec3 specular;
};
layout(binding = 1) uniform SphereBuffer {
    Sphere spheres[3];
};
{% endhighlight %}

The object is really simple, just a position, a radius, albedo and specular colors. So one float and three vec3. The only actual change on the shader for this next step was adding the layout binding to receive three of theses structs, and then use these values during the ray tracing.

Seems simple enough. Now, on the other side (the C++ program) I just mirrored this data. I added all the other necessary plumbing to submit the info to the shader, so I had:

{% highlight c++ %}
struct Sphere {
    glm::vec3 position;
    float radius;
    glm::vec3 albedo;
    glm::vec3 specular;
};

// Bunch of code related to layout binding and etc...

void createAndMapObjectsUniformBufferData() {
    Sphere sphere1 = {};
    Sphere sphere2 = {};
    Sphere sphere3 = {};

    // setting up the values for all the three spheres ...

    Sphere spheres[3] = { sphere1, sphere2, sphere3 };
    void* data;
    if (vkMapMemory(device, spheresUniformBufferMemory, 0, sizeof(spheres), 0, &data) != VK_SUCCESS) {
        throw std::runtime_error("Error mapping memory!");
    }
    memcpy(data, spheres, sizeof(spheres));
    vkUnmapMemory(device, spheresUniformBufferMemory);
}
{% endhighlight %}

I'm using [GLM](https://glm.g-truc.net/0.9.9/index.html) to take advantage of objects such as `glm::vec3` to make my life a little easier. After the three spheres are created, I simply map this data to the uniform buffer memory and send it on its way. I'm expecting to see the same image my Ray Tracer rendered before when the program ends, because the three spheres use the same info as they used on the shader, so let's run it!

<div align="center">
<a href="https://i.imgur.com/2dBu1jg.png" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/2dBu1jg.png" style="width:40%;border-style:solid;border-width:1px"/></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>The final result using the uniform buffer.</p></font></div>
</div>

## Wait, what?

My thoughts exactly! What is going on? Why am I seeing only one sphere, and why does it have some weird artifacts on its surface?

After this initial result my first reaction was to double check all the code to see if I was doing something wrong along the lines of "only reserving enough memory for one sphere". Ended up looking at a handful of stackoverflow questions, tutorials and whatnot to find how to correctly setup uniform buffers.

But I was actually looking at the wrong problem. I was in fact correctly setting up the uniform buffer, and it had the necessary memory to hold 3 Sphere structs. What was wrong here was the memory alignment of this struct.

## The struct definition matters

Because of how compilers deal with C++ structs, and how Vulkan expects this data to be received on the shader, all sorts of weird effects can happen. For instance, what if I changed the order of the variables inside the struct (both on the shader and on the program)?

{% highlight c++ %}
struct Sphere {
    float radius;
    glm::vec3 position;
    glm::vec3 albedo;
    glm::vec3 specular;
};
{% endhighlight %}

What you get is another new image:

<div align="center">
<a href="https://i.imgur.com/UniSaE8.png" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/UniSaE8.png" style="width:40%;border-style:solid;border-width:1px"/></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>No spheres!</p></font></div>
</div>

Now there are no spheres, but at the bottom you can also sort of see the shadow of a sphere (is the sphere actually with a completely different position?). The explanation to this weird behaviour is actually at [Vulkan's specification itself](https://www.khronos.org/registry/vulkan/specs/1.1-extensions/html/chap14.html#interfaces-resources-layout). Vulkan expects the data in the struct to be defined in a specific way. For this particular struct we are interested in two points:

- A scalar of size N has a base alignment of N.
- A three- or four-component vector, with components of size N, has a base alignment of 4 N.

In the Sphere struct we have two types of variables: float (4 bytes) and vec3 (4x3 = 12 bytes). This struct messes up all the requirements. For instance, by having the float first we get `radius` using 4 bytes, so `position` has an offset of 4, `albedo` has an offset of 16 and `specular` has an offset of 28 (for a total size of 40 bytes).

Vulkan can't really deal with the data aligned like this, but it doesn't just break (and apparently there aren't any validation layers that will let you know about this issue), the program still runs and is able to finish, but operating on the wrong address. This can corrupt your memory, and return incorrect results!

What we need is to align all these variables, and to help us we can use [C++11's alignas](https://en.cppreference.com/w/cpp/language/alignas). On the shader we leave things as-is, but on the program:

{% highlight c++ %}
struct Sphere {
    alignas(4) float radius;
    alignas(16) glm::vec3 position;
    alignas(16) glm::vec3 albedo;
    alignas(16) glm::vec3 specular;
};
{% endhighlight %}

Now we are specifing exactly [what alignment we want](https://en.cppreference.com/w/cpp/language/object#Alignment) for each variable (in this case you don't really need to specify it for the float value, since the other alignments would still have the struct defined correctly, but it doesn't hurt to be clear with everything here).

With this you have `radius` using 4 bytes plus 12 bytes of padding, so `position` has an offset of 16 bytes, uses 12 bytes (12+16 = 28) and have some more 4 bytes of padding, making `albedo` with an offset of 32 and, following the same logic, making `specular` with an offset of 48. The total size of the struct now is 64 bytes (because objects of type Sphere now must be allocated at 16-byte bondaries).

Here's a quick check you can do if you are curious:

{% highlight c++ %}
struct SphereA {
    float radius;
    glm::vec3 position;
    glm::vec3 albedo;
    glm::vec3 specular;
};

struct SphereB {
    alignas(4) float radius;
    alignas(16) glm::vec3 position;
    alignas(16) glm::vec3 albedo;
    alignas(16) glm::vec3 specular;
};

printf("%d\n", sizeof(float));     // Should show 4
printf("%d\n", sizeof(glm::vec3)); // Should show 12

printf("%d\n", sizeof(SphereA));   // Should show 40
printf("%d\n", alignof(SphereA));  // Should show 4

printf("%d\n", sizeof(SphereB));   // Should show 64
printf("%d\n", alignof(SphereB));  // Should show 16
{% endhighlight %}

Also, important to keep in mind that in C++ data types are plataform specific. On my machine I'm getting `sizeof(float) == 4` but your milage may vary. Still, this alignment will be required regardless.

After dealing with this issue, the Ray Tracer is finally rendering the image correctly. You can [check this implementation on my GitHhub repo](https://github.com/fvcaputo/vulkan-projects/tree/master/Ray%20Tracer%20(using%20UBOs)).

## That's cool but, can we do better?

The answer is: it depends. You might be thinking that without all the `alignas` we had a struct with a total size of 40 bytes, but because of the alignment requirements we ended up with a struct with total size of 64 bytes. Seems like there is a lot of padding (i.e., useless bytes), could we do something about that?

One approach is to describe your struct in a different way:

{% highlight c++ %}
struct Sphere {
    alignas(16) glm::vec3 position;
    alignas(16) glm::vec3 albedo;
    alignas(16) glm::vec3 specular;
    alignas(4) float radius;
};

printf("%d\n", sizeof(Sphere));   // Should show 48
printf("%d\n", alignof(Sphere));  // Should show 16
{% endhighlight %}

We changed where our float value is, and we ended up with a smaller struct! And indeed, if you run this code (and make the same change on the shader), you will endup with the correct result!

What is happening here is, all the three `glm::vec3` are 16 byte aligned, and they take 12 bytes of storage so all of them have 4 bytes of padding. And, well, isn't 4 bytes exactly the same size as `float` here? Indeed! In the resulting alignment of this struct, the compiler can take advantage of these memory requirements and instead of using 4 empty bytes as padding it can just add the `float` value instead.

The result would look like: `position` uses 12 bytes plus 4 of padding, `albedo` uses 12 bytes plus 4 of padding, `specular` uses 12 bytes and, instead of empty 4 bytes of padding, we have the `radius` here. All the fields stil match the necessary alignments of Vulkan's specification, with less storage space nedded. In this example we are dealing with 3 spheres that went from 64 to 48 bytes each, so instead of sending a total of 192 bytes we are sending 144 bytes to the GPU!

## Bottom line

Data alignment can be confusing, but with APIs like Vulkan you need to be sure to align your data so that the shader is able to access it, so it's important to learning how all of this works (specially if you are interested in doing little optimizations like the one described above).

There are some good resources out there to learn more. I'll give a shotout to IBM's Developer Works article [Data alignment: Straighten up and fly right](https://www.ibm.com/developerworks/library/pa-dalign/), which was really helpful to me.

Hopefully this post will also help you out :)
