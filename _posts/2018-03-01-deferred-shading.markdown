---
layout: post
title:  "We've Got Deferred Shading"
subtitle: <div style="margin-top:-15px;"><img src="https://i.imgur.com/Fc1T4zq.jpg" style="width:32%;margin-right:0.5%;margin-bottom:5px;" border="1"><img src="https://i.imgur.com/NZiLxez.jpg" style="width:32%;margin-right:0.5%;margin-bottom:5px;" border="1"><img src="https://i.imgur.com/LuV2odd.jpg" style="width:32%;margin-bottom:5px;" border="1">After some experimenting with OpenGL's framebuffers previously, the next step was to implement deferred shading on the framework.
date:   2018-03-01 08:33:00
language: en
image: https://i.imgur.com/Fc1T4zq.jpg
description: Implementing deferred shading.
---

<div style="text-align:center;padding-bottom:10px;">
<a href="https://i.imgur.com/Fc1T4zq.jpg" style="color: black;"><img src="https://i.imgur.com/Fc1T4zq.jpg" style="float: left; width: 32%; margin-right: 0.5%" border="1"></a>
<a href="https://i.imgur.com/NZiLxez.jpg" style="color: black;"><img src="https://i.imgur.com/NZiLxez.jpg" style="float: left; width: 32%; margin-right: 0.5%" border="1"></a>
<a href="https://i.imgur.com/LuV2odd.jpg" style="color: black;"><img src="https://i.imgur.com/LuV2odd.jpg" style="float: left; width: 32%" border="1"></a>
<p><div style="text-align:center;"><font color="gray" size="2px"><p>From left to right, texture with positions, texture with normals and final rendered scene.</p></font></div></p>
</div>

Last time I worked on getting shadows working on my engine through the depth mapping. That helped me better understand how to work with different framebuffers, how to render to textures and how to then read data from those textures.

This time, with that knowledge, the next thing up was deferred rendering. It's a technique, from what I understand, extensively used in games because it greatly speeds up rendering. This happens because the rendering engine does not need to iterate through every fragment of every object on the scene, even the ones that will not be rendered (fragments that are behind other fragments, etc).

To implement deferred rendering I have created a basic gBuffer to store all the necessary data for rendering.

## The gBuffer

<div align="center">
<a href="https://i.imgur.com/Fc1T4zq.jpg" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/Fc1T4zq.jpg" style="width:35%;" border="1"/></a>
<a href="https://i.imgur.com/NZiLxez.jpg" style="color:black;"><img src="https://i.imgur.com/NZiLxez.jpg" style="width:35%" border="1"></a>
</div>
<div align="center" style="padding-top:10px">
<a href="https://i.imgur.com/nQUHVwi.jpg" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/nQUHVwi.jpg" style="width:35%;" border="1"/></a>
<a href="https://i.imgur.com/5r96ats.jpg" style="color:black;"><img src="https://i.imgur.com/5r96ats.jpg" style="width:35%" border="1"></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>Here we have the gBuffer textures, position, normal, diffuse material and specular material.</p></font></div>
</div>

The idea behind the gBuffer, which I suppose is the same as with most compter graphics techniques, is optimization. If you render a scene normally, what will happen is that every fragment will go through lighting calculations, which are very expensive. Using the gBuffer we render the scene in two passes and we are able to only do lighting calculations on fragments that are definately going to be on screen.

What we do is essentially convert the object information to textures in the first pass. Take an object's position for instance. What we want to do is, represent the position on a texture. For that we set up the scene normally, but we change our shaders to instead output the position as colors (the RGB values in the texture are in fact the XYZ positions of the fragment).

In the end it's not really that complicated, here we have part of what the vertex shader looks like:

{% highlight c++ %}
layout (location = 0) in vec3 vPosition; // model position

out vec3 FragPos; // output to fragment shader

void main () {
    // Position to the frag shader
    // We want all the data (position, normals, etc on that same coord system)
    vec4 worldPos = mTransform * vec4(vPosition, 1.0);
    FragPos = worldPos.xyz;

    // Now the position of the vertex should be the actual final object position
    gl_Position = mProjMatrix * mViewMatrix * worldPos;
}
{% endhighlight %}

Then on the fragment shader all we need to do is throw that data on a texture. For that you need to set up a framebuffer with the number of color attachements being the same as the number of textures you are going to use. When you bind that framebuffer, on the fragment shader the `layout (location = 0) out` variable will correspond to the `GL_COLOR_ATTACHMENT0` on the framebuffer. So, something like this:

{% highlight c++ %}
// Out gbuffer values
layout (location = 0) out vec3 gPosition;

// In values from the fragment shader
in vec3 FragPos;

void main () {
    // Position on the g buffer
    gPosition = FragPos;
}
{% endhighlight %}

Then, what we have on that top left picture above is the texture showing positions. You are actually able to somewhat see the shapes of the objects, because their positions were calculated using the modelview and projection matrices. However the color of the texture is actually the fragment position in world coordinates.

What this helps with is to minimiza the data we need to process only to the fragments that will be seen in the final render. The first pass creates a texture like that (and uses the same idea for the other values, normals, diff values and spec values).

## The Final Render

Now in the first pass we generated all these four textures, and we are going to use them to render the complete scene. For that the second pass will render the whole scene to a screen quad that fills the entire screen. And here is the trick: when we are rendering this screen quad we will sample from the texture the necessary data we know is there!

<div align="center">
<a href="https://i.imgur.com/LuV2odd.jpg" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/LuV2odd.jpg" style="width:50%;" border="1"/></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>The final render.</p></font></div>
</div>

So, think about the screen coordinates. We will begin at coordinates (0,0) and go pixel by pixel to position (512,512), or whatever is the resolution of your render. Then for every (x,y) position during that loop what we do is sample the textures we have generated on the gbuffer. Position (x,y) on the position texture results in a vec3 value that, even though it is a rgb texture, we now represents the world position of that pixel. We do the same for all the other textures and bam, we have positon, normal, diffuse and specular colors. Now we are able to use that information and the lighting info to calculate the final color of that pixel.

On the fragment shader for the second pass there will be something like this:

{% highlight c++ %}
void main () {
    vec3 FragPos = texture(gPosition, uvTexCoord).rgb;
    vec3 Normal = texture(gNormal, uvTexCoord).rgb;
    vec3 Albedo = texture(gColorAlb, uvTexCoord).rgb;
    vec3 Specular = texture(gColorSpec, uvTexCoord).rgb;

    vec3 viewDir = normalize(CameraWorldPos - FragPos); // everything is in world pos!
    vec3 lightDir = normalize(light.position - FragPos); // everything is in world pos!

    // Calculate final color with phong shading normally...

    fragColor = vec4(ambientColor + diffuseColor + specularColor, 1.0);
}
{% endhighlight %}

And that is the trick, since we are looping and sampling the texture we are for sure only looking at pixels that are going to be visible. It's easy to see on the textures above that we have a handful of objects behind other objects. With a simple forward rendering technique, every single fragment from those objects would go through the lighting calculations, inclusing the ones that will not be shown because of their depths. This greatly reduces this expensive calculation and renders the final scene much faster!

# References

* [Metal Gear Solid V - Graphics Study](http://www.adriancourreges.com/blog/2017/12/15/mgs-v-graphics-study/) - Adrian Courrèges' blog has some great articles on computer graphics, this in particular talks about how scnes are rendered on MGSV.
* [Deferred shading](https://en.wikipedia.org/wiki/Deferred_shading) - Wikipedia's article on deferred shading has some good info and history about the technique.
* [Ivysaur Pokemon Free Sample](https://www.turbosquid.com/3d-models/free-obj-model-ivysaur-pokemon-sample/1136333) - Here is a link to the object I have used to render the scenes on this post, though in the end I had to slightly modify the object to work with my object loader (needed to change it to triangular faces and invert the normals).