---
layout: post
title:  "OpenGL Debugging"
subtitle: Working on some OpenGL projects for a while now, I wanted to talk a little bit about debugging my own code.
subtitleImage: https://i.imgur.com/3YcSnqr.png?1
date:   2018-04-16 10:23:00
language: en
image: https://i.imgur.com/Fc1T4zq.jpg
description: OpenGL Debugging.
---

I have been meaning to talk a little bit about the subject of debugging graphics applications, specifically about my experience with OpenGL. This is going to be a post without really a lot of rendering pictures (which is what I normally use this little blog for, haha), mostly about my own experiences while programming my engine.

I believe I might have mentioned this before, but when I mention my "OpenGL Engine/Framework", it's also not something that is anywhere near production-level code. This engine is really just something that I have been working on for a while to learn and study techniques and the OpenGL API. There really isn't any unit testing going on, or really any robust error recovery or anything like that. But that is by design.

Every now and then, when I'm working on some new project or new technique using the engine, I come across some problems where I have to debug my code because something is definitely not working right. I think most people that worked with a graphics API eventually had to deal with some issue where your render was simply not working. And then comes the usual ideas, "my final rendering is basically completely black, let me check my fragment render and simply render a solid color to see what happens".

This kind of approach is actually really helpful and I have used it a handful of times. If you were getting just a black screen before, and now you see your objects in this solid color you have chosen, it means a few things. First, if your objects are where they should be, all the code "before" the fragment shader is most likely correct. You are sending your data to the buffer correctly, the transformations are correct, and all that is fine. In this case, your problem is maybe somewhere related to the fragment shader data or calculations. Maybe you are sending some data the wrong way, maybe you are calculating the final color of your fragment wrong. This approach at least helps you figure out where is your issue.

Now, doing something like that sometimes help, but sometimes doesn't. I ran into some problems when implementing deferred shading (my previous post) where this did not help me much.

## The Problem

For deferred rendering you normally render in two passes, first you render texture to the gbuffer and then you use that data to render your scene. While I was implementing it, I had a problem with the second pass. The first pass did work, I checked that by having the fragment shader of the first pass render to the default framebuffer, thus actually seeing the textures that would be sent to the gbuffer.

However it seemed that those textures were not successfully being saved on the gbuffer or not being sent to the second render pass. On the second pass, no matter what I did to those shaders (like trying to render solid colors and all that), I only got a black screen as a result. So ok, at least I know the problem seems to be between the two passes, the textures were either not being saved to the gbuffer or not sent to the next pass correctly.

But at this point I was sort of stuck. Yes, there was a problem (and a really small one, that I'll show at the end here), but exactly because it was so small it took me a good long while to find it.

## Finding The Issue

Because I was stuck here, I did some research on better ways to debug OpenGL code. There has to be a better way, right? And what I found the incredibly helpful function [glGetError](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glGetError.xhtml). Great, now I have another way to debug the code and check for errors, this function that will show me the error flag so I can better understand what is going on.

What I did was simply add the following code to my rendering function:

{% highlight c++ %}
// check OpenGL error
GLenum err;
while ((err = glGetError()) != GL_NO_ERROR) {
    cerr << "OpenGL error: " << err << " - " << gluErrorString(err) << endl;
}
{% endhighlight %}

The function `gluErrorString` is just a helper function that translates `glGetError`'s error enum to a string. With this code I was able to track down the error in the deferred shading code, and also another error on my shader compiler that turns out was always wrong!

The error that kept being printed was `GL_INVALID_OPERATION`. It means some operation that was being performed was, well, invalid. But at the same time instead of having everything blow up, OpenGL simply ignored this operation (for better or worse).

## The Shader Compiler Error

By using this function I ended up finding [this small issue](https://github.com/fvcaputo/openglframework/commit/af15732fb5f417c71277896e5d30e6f2de7f46fd#diff-923ec7c7592ed43db3a3134e06f322b6R165) that was in my shader compiler code. In this file I had some checks and some log printing if there was any error. This was the code that produced and error flag:

{% highlight c++ %}
// Link program id
glLinkProgram(programID);

// Check the flag
glGetShaderiv(programID, GL_LINK_STATUS, &flag);

// Print error log, if any
printShaderLog(programID);
{% endhighlight %}

The problem here is obvious now that I look at it. Both functions `glGetShaderiv` and `printShaderLog` should acually be used for the shader objects and *not* the "program" object! For that we use `glGetProgramiv`! Turns out this thing has been wrong since forever, and OpenGL simply ignored this invalid operation. So cool, now that is one thing I was able to fix already.

## The Deferred Shading Error

The error in my deferred shading code was actually quite simple. I was doing:

{% highlight c++ %}
GLuint mViewMatrixID = glGetUniformLocation(programGeometryPass, "mViewMatrix");
...
GLuint mProjMatrixID = glGetUniformLocation(programGeometryPass, "mProjMatrix");
...
glUseProgram( programGeometryPass );
{% endhighlight %}

Waht is happening here is, I am setting up my matrices on the `programGeometryPass` before actually calling `glUseProgram(programGeometryPass)`. This is actually a really obvious error that I was never even considering because, for some reason, it still sort of worked! Like I said before, it seemed like the first pass of the rendering was working because I could see the textures created for the gbuffer, I was seeing the following pictures:

<div align="center">
<a href="https://i.imgur.com/Fc1T4zq.jpg" style="color:black;margin-right:0.5%"><img src="https://i.imgur.com/Fc1T4zq.jpg" style="width:35%;" border="1"/></a>
<a href="https://i.imgur.com/NZiLxez.jpg" style="color:black;"><img src="https://i.imgur.com/NZiLxez.jpg" style="width:35%" border="1"></a>
<div style="text-align:center;"><font color="gray" size="2px"><p>A couple gBuffer textures.</p></font></div>
</div>

They were in the expected position with the expected data. So I never thought twice about looking for errors on the transformation, view and projection matrices, because it looked like they were working. Now, honestly I don't really know how to explain why they were working. These matrices should not have been successfully sent to the shaders on the "programGeometryPass", but they were. And the collateral effect was that at the end of the rendering, in the fragment shader `out` variables, the textures were not sent to the gbuffer framebuffer at all.

But I was seeing the invalid operation error around the first pass render, which eventually led me to find this error, that turns out was really simple to correct (move the glProgram before setting up the uniform variables there).

## Some Thoughts

I wanted to write a little bit about this learning experience that I had while looking for errors in my code. When you are coding on your free time to learn new things, it's easy to just write code that "won't break" because you know how you are using everything. You might create a function that, if it receives some certain parameter, will simply break. However you are only using it for that one particular script, and you only call it with correct values, so you don't bother with any kind of check on the parameters.

Now, I'm not saying that you should strive for production level code on the stuff you are doing on the side to study and learn. But keep in mind that some error checking can help you a long way! Obviously this small change that I did (checking the OpenGL error flag), is very far from actually making my whole framework robust against errors, but it's another helpful layer against problems.
