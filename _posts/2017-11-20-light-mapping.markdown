---
layout: post
title:  "Lightmaps and Normal Mapping"
subtitle: With reliable texture loading added previously, now I further improved the engine to support light and normal mapping. I have changed the object loading to get all the necessary files, the textures, and the calculation of the TBN matrix to get everything to work.
subtitleImage: https://i.imgur.com/mAUNY3h.jpg
date:   2017-11-20 11:23:00
language: en
image: https://i.imgur.com/mAUNY3h.jpg
description: Some thoughts about dealing with lightmaps in OpenGL.
---

<div style="text-align:center;padding-bottom:10px;">
<a href="https://i.imgur.com/92hqDAV.jpg" style="color: black;"><img src="https://i.imgur.com/92hqDAV.jpg" style="float: left; width: 32%; margin-right: 0.5%" border="1"></a>
<a href="https://i.imgur.com/p8ZcGGO.jpg" style="color: black;"><img src="https://i.imgur.com/p8ZcGGO.jpg" style="float: left; width: 32%; margin-right: 0.5%" border="1"></a>
<a href="https://i.imgur.com/mAUNY3h.jpg" style="color: black;"><img src="https://i.imgur.com/mAUNY3h.jpg" style="float: left; width: 32%" border="1"></a>
<p><div style="text-align:center;"><font color="gray" size="2px"><p>From left to right, a textured plane without lightmaps, with lightmaps, and with lightmaps and normal map.</p></font></div></p>
</div>

Alright, so after playing around with textures the last time, and get everything to work correctly there without dealing with unreliable external libs, the next things I wanted to tackle were light mapping and normal mapping (and while I was at it, decided to throw some shadow mapping in there too).

These techniques are sort of related at least in the sense that they require dealing with textures, the difference is that in light mapping and normal mapping we read and process the texture, and in shadow mapping we create the texture from our geometry. Here I'll write about the light and normal mapping, and I'll make another post about the shadow mapping at a later date.

## Lightmaps

Ok, so really the last update to my engine (the previous post on this blog) was about updating how textures are loaded from files and mapped to objects. I had to refactor part of the code to make it clearer and get it working using a more reliable lib, but after that dealing with textures became much simpler.

However, that only allowed me to create scenes with textured objects that did not react to lighting. The texture was simply mapped to the geometry and displayed on screen. My next objective then was: get lightmaps to work.

### Diffuse and Specular Maps

So there are actually a number of different texture maps that can be used to improve the look of some object. Now, the first couple I implemented were diffuse and specular maps. The idea is not that complicated, instead of just mapping the texture to the geometry, we actually and those textures to the phong illumination model. This required some changes on how I stored and submitted data to the shaders, but most of the changes to actually get the effect only had to be done on the shaders (the fragment shader).

<div align="center">
<a href="https://i.imgur.com/NpOweqq.png" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/NpOweqq.png" style="width:35%;" border="1"/></a>
<a href="https://i.imgur.com/NylY4j6.png" style="color:black;"><img src="https://i.imgur.com/NylY4j6.png" style="width:35%;" border="1"></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>Diffuse texture and Specular texture.</p></font></div>
</div>

Below are the relevant pieces to the fragment shader:

{% highlight c++ %}
// Light values
struct Light {
    vec3 position;
    vec3 ambient;
    vec3 lightIntensity;
};

// Phong Illumination values
struct Material {
    sampler2D diffuse;
    sampler2D specular;
    float     specExp;
};

// ...

void main () {
    // ...

    vec3 diffText = vec3(texture(material.diffuse, uvTexCoord));
    vec3 specText = vec3(texture(material.specular, uvTexCoord));

    vec3 ambientColor = light.ambient * diffText;
    vec3 diffuseColor = light.lightIntensity * max(dot(nN, nL), 0.0) * diffText;
    vec3 specularColor = light.lightIntensity * pow(max(dot(R, nV),0.0), material.specExp) * specText;

    // ...
}
{% endhighlight %}

So, a few changes there. First, I changed some parts of the shader to use structs so that we would have wrappers for the relevant information. This was not really necessary, but it just made things much clearer. Then, after loading the textures and sending them to the shader, together with the texture coordinates, simply use the new values in the phong illumination to get the final result.

<div align="center">
<a href="https://i.imgur.com/92hqDAV.jpg" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/92hqDAV.jpg" style="width:35%;" border="1"/></a>
<a href="https://i.imgur.com/p8ZcGGO.jpg" style="color:black;"><img src="https://i.imgur.com/p8ZcGGO.jpg" style="width:35%;" border="1"></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>Results, on the left no light effect, on the right using lightmaps we see how the object is supposed to react to the light.</p></font></div>
</div>

The specular texture is only used to calculate the specular color, but the diffuse one is used both on the diffuse and ambient color to simplify things. Below two pictures comparing the object with no lightmaps and one with them.

### Normal Map

With the general idea of using lightmaps down we can now start adding new effects, and the next one I tackled was normal mapping. Here, although the idea is also simple, we have the normals of each fragment encoded into an image called the normal map of the object, for this to work it required a deeper change on the code if compared to the changes made to get the diff and spec mapping to work.

#### Reading the Normal Map

With diffuse and specular maps we just need to deal with them like they are simple textures, read the texture files and the texture coordinates, submit them to the shaders and let the shader to the work. With the normal map things are different, we need to take into account what is commonly called the "Tangent Space".

<div align="center">
<a href="https://i.imgur.com/PsIJTsx.png" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/PsIJTsx.png" style="width:35%;" border="1"/></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>The normal map for our brick wall.</p></font></div>
</div>

Here's the thing, in a texture used for normal mapping  what we usually see is a blueish image like the above picture. This is because the normals of the object are encoded into the image, and since we have the image describe with the RGB values we get a blue image because normally the normals are encoded pointing outwards the screen (positive Z considering the a right haded coordinate system), so if RGB are used as XYZ, blue is going to be the primary value.

Now lets say we just get those values from the texture and use as is as a normal. You can imagine that it would work for, let's say, a plane with its surface normal pointing to Z positive. Now, if we start rotating that object and still have the normals pointing to Z, things start getting weird. And not only that, let's say we want to use that normal map in any other object that is not a plane, like a cylinder for instance. We can't have all the normals pointing to positive Z, they should wrap around the cylinder instead.

To deal with that you take into account the "tangent space", we need to transform the normals on the normal map considering the position of the fragment on the object. For that we calculate what is called the "tangent" and the "bitangent" using the edges and the uv coordinates of our object. This data is calculated while we are reading the normal map and sent to the shaders to calculate the so-called TBN matrix, used to transform the normals.

#### Transforming and Using the Normals

Now, with the tangent and bitangent values calculated, and with the object's surface normal we calculate the TBN matrix on the vertex shader.

{% highlight c++ %}
// Shape values
in vec3 vNormal;
in vec3 vTangent;
in vec3 vBitangent;

out mat3 TBN;

// ...

void main () {
    // ...

    // TBN matrix for normal mapping
    vec3 T = normalize(vec3(modelView * vec4(vTangent, 0.0)));
    vec3 B = normalize(vec3(modelView * vec4(vBitangent, 0.0)));
    vec3 N = normalize(vec3(modelView * vec4(vNormal, 0.0)));
    TBN = mat3(T, B, N);

    // ...
}
{% endhighlight %}

This is the matrix used to transform the normal from the normal map to the tangent space, taking into account the object geometry and the current modelView matrix. The matrix is sent to the fragment shader where it transforms the normals:

{% highlight c++ %}
// Phong Illumination values
struct Material {
    sampler2D diffuse;
    sampler2D specular;
    sampler2D normal;
    float     specExp;
};

// in for TBN
in mat3 TBN;

// ...

void main () {
    // ...

    // Use normalMap values and TBN matrix to calculate the normal
    vec3 norm = vec3(texture(material.normal, uvTexCoord));
    norm = normalize(norm * 2.0 - 1.0);
    norm = normalize(TBN * norm);

    // ...
}
{% endhighlight %}

Then, for the following calculations of the phong illumination, instead of using the surface normal we use this new normal. The end result is a much more realistic object, with a "bumpy" look, without actually changing the geometry to add those bumps.

<div align="center">
<a href="https://i.imgur.com/p8ZcGGO.jpg" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/p8ZcGGO.jpg" style="width:35%;" border="1"/></a>
<a href="https://i.imgur.com/mAUNY3h.jpg" style="color:black;"><img src="https://i.imgur.com/mAUNY3h.jpg" style="width:35%;" border="1"></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>On the left the wall only with the diffuse and specular textures, on the right the wall using the normal map.</p></font></div>
</div>

### Final Thoughts

Adding support to lightmaps makes it so that textured objects look a lot more real. So does normal mapping. Not a lot of changes were necessary to get all of this working, since it's mostly new features. The TBN matrix is something important to take into account, otherwise you would get really weird results without knowing exactly why. And I really think you should first try it with a simple plane texture, it's much simpler.

# References

* [Learn OpenGL](https://learnopengl.com/#!Advanced-Lighting/Normal-Mapping) - Normal mapping tutorial.
* [Opengl-Tutorial](https://www.opengl-tutorial.org/intermediate-tutorials/tutorial-13-normal-mapping/) - Tutorial 13 : Normal Mapping.
* [CrazyBump](https://www.crazybump.com/) - Helpful tool to create/edit normal maps.
