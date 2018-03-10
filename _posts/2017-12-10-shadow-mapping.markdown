---
layout: post
title:  "Framebuffers and Shadow Mapping"
subtitle: <div style="margin-top:-15px;"><img src="https://i.imgur.com/77EkHMj.jpg" style="width:32%;margin-right:0.5%;margin-bottom:5px;" border="1"><img src="https://i.imgur.com/vgbukjA.jpg" style="width:32%;margin-right:0.5%;margin-bottom:5px;" border="1"><img src="https://i.imgur.com/glXogdt.jpg" style="width:32%;margin-bottom:5px;" border="1">Time to add shadow mapping to the engine, a feature that requires a better understanding of OpenGL's framebuffers.
date:   2017-12-10 10:33:00
language: en
image: https://i.imgur.com/glXogdt.jpg
description: Implementing shadow mapping using framebuffers.
---

<div style="text-align:center;padding-bottom:10px;">
<a href="https://i.imgur.com/77EkHMj.jpg" style="color: black;"><img src="https://i.imgur.com/77EkHMj.jpg" style="float: left; width: 32%; margin-right: 0.5%" border="1"></a>
<a href="https://i.imgur.com/vgbukjA.jpg" style="color: black;"><img src="https://i.imgur.com/vgbukjA.jpg" style="float: left; width: 32%; margin-right: 0.5%" border="1"></a>
<a href="https://i.imgur.com/glXogdt.jpg" style="color: black;"><img src="https://i.imgur.com/glXogdt.jpg" style="float: left; width: 32%" border="1"></a>
<p><div style="text-align:center;"><font color="gray" size="2px"><p>From left to right, scene with no shadows, depth map of the scene, scene using the depth map to calculate shadows.</p></font></div></p>
</div>

Up until now I feel like almost every "basic" technique was implemented un my engine. Mesh tessellation, illumination algorithms, light mapping, texture mapping, model loading, etc. The only thing that I felt was missing really was shadow mapping. You could make a lot of different scenes with different effects and textures already, but you could not actually project shadows of any objects yet.

So that is the plan this time, get shadows to work. And to do that we need to better understand framebuffers and how to render to textures, since the whole idea behind the usual shadow mapping technique involves a depth map.

## The Depth Map

Internally OpenGL has a depth buffer (also called z-buffer) that can be utilized by simply using `glEnable(GL_DEPTH_TEST)`. Most of the sample programs I wrote simply use the depth buffer like that, which OpenGL uses to decide which fragment is actually going to be displayed on screen. If you imagine two polygons, one behind another, there will probably be certain fragments from both that will map to the same (x,y) coordinate, and the way openGL knows which one is "in front" and should be displayed is through the calculated depth buffer (which you can see if you just render `gl_FragCoord.z` on the frag shader).

To create a shadow map though, what we want is not exactly to use this depth map OpenGL gives you by enabling depth test, but rather make our own. The most simple way to do it seems intuitive enough: we need to render a depth map using the light source as the camera and then use this to decide which fragment will be displayed fully, and which will be displayed "in shadow".

### Enter: The Framebuffer

<div align="center">
<a href="https://i.imgur.com/vgbukjA.png" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/vgbukjA.png" style="width:35%;" border="1"/></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>The depth map.</p></font></div>
</div>

OpenGL works with a default framebuffer, essentially a combination of different types of buffers (color, depth, stencil). But it also gives us the option to use our own framebuffers, essentially giving us the option of running more than one "render pass". Shadow mapping requires at least two passes: one to render the scene from the point of view of the light to generate the shadow map, and another to render our scene using that depth map.

Rendering the depth is easy enough when have the idea down. Simply create the view matrix considering the light as the point of view, transform the objects using the model, view and projection and in the end render the `gl_FragCoord.z`. You can actually write a program that does just that. But what we need is a single program that does that, and uses that result on another rendering process.

For that we use a framebuffer. We set it up as a framebuffer with only one component, a `GL_DEPTH_COMPONENT`, and attach a texture to it. This is called "render to a texture". The code below shows this framebuffer initialization.

{% highlight c++ %}
// Our ids
GLuint depthMapFBO;
GLuint texDepthBuffer;

// bind framebuffer
glGenFramebuffers(1, &depthMapFBO);
glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO);

// create the texture
glGenTextures(1, &texDepthBuffer);
glBindTexture(GL_TEXTURE_2D, texDepthBuffer);

// set it up as simply a depth component
glTexImage2D(GL_TEXTURE_2D, 0, GL_DEPTH_COMPONENT, SHADOW_WIDTH, SHADOW_HEIGHT, 0, GL_DEPTH_COMPONENT, GL_FLOAT, NULL);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MIN_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_MAG_FILTER, GL_NEAREST);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_S, GL_REPEAT);
glTexParameteri(GL_TEXTURE_2D, GL_TEXTURE_WRAP_T, GL_REPEAT);
glFramebufferTexture2D(GL_FRAMEBUFFER, GL_DEPTH_ATTACHMENT, GL_TEXTURE_2D, texDepthBuffer, 0);

// framebuffer by default needs at least a color component, but we wont use it, so we need to state that
glDrawBuffer(GL_NONE);
glReadBuffer(GL_NONE);

// check if it is correct
if(glCheckFramebufferStatus(GL_FRAMEBUFFER) != GL_FRAMEBUFFER_COMPLETE) {
    std::printf("Error building Framebuffer!\n");
}
{% endhighlight %}

Now you can just render to this framebuffer by using `glBindFramebuffer(GL_FRAMEBUFFER, depthMapFBO)` and you are set, you will render to the "texture" `texDepthBuffer` only the depth component.

## Shadow Mapping

<div style="text-align:center;padding-bottom:10px;">
<iframe width="560" height="315" src="https://www.youtube.com/embed/3Iax7v3LzwM" frameborder="0" allowfullscreen></iframe>
<div style="text-align:center;margin-top:-5px;"><font color="gray" size="2px">Real time shadow mapping with a moving light source.</font></div>
</div>

Ok, now we have the depth map using the light source as the point of view and we will use that information to render shadows. Besides needing to render the scene using out camera position, we also need the coordinates using the light source. This is actually the data we use to decide what is in shadow: the depth value of the current fragment is smaller than than the fragment with that same coordinate on our depth map? If the answer is true then the fragment is displayed normally, if not then the the fragment should be in a shadow.

All these checks occur in the fragment shader. It looks something like this:

{% highlight c++ %}
posLightSpace <- the current fragment position using light source as point of view
shadowMap <- the depth map calculated on the previous pass as a texture,
             also using the light source as point of view

currentFragmentDepth = posLightSpace.z;
closestDepth = texture(shadowMap, posLightSpace.xy)

inShadow = currentFragmentDepth < closestDepth ? false : true
{% endhighlight %}

This is why shadow mapping consists of two passes. First, render the depth map to a texture, then actually render the scene to the screen using the depth map to help the render. Above I wrote a really simplified algorithm of how it works, you can check out this [link](https://github.com/fvcaputo/openglframework/blob/master/shaders/phongLightingShadowFrag.glsl) to my shader to have a better idea of how it works.

## Final Thoughts

All of this requires some setting up, but in the end you are really just comparing two values to decide what should be in shadow. The idea is simple enough and OpenGL is really helpful by allowing us to use framebuffers, which can actually be used for all sorts of different techniques. Next up I am planning on using framebuffers to tackle deferred lighting.

# References

* [Learn OpenGL](https://learnopengl.com/#!Advanced-Lighting/Shadows/Shadow-Mapping) - Shadow mapping tutorial.
* [OpenGL Archive](https://www.opengl.org/archives/resources/faq/technical/depthbuffer.htm) - OpenGL Archived Depth Buffer FAQ.