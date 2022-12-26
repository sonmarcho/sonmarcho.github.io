---
layout: post
title: "Sculpting a head with ZBrush"
date: 2022-12-26
tags: art cg
published: true
---

<!-- For some reason, we can't put images in the _posts directory: I thus
     put them in the /assets directory, and use the following variable to
     make the links less cumbersome to write -->
{% assign images = "/assets/2022-12-25-sculpted-head/" %}

I recently decided to create a logo for my project
[Aeneas](https://github.com/AeneasVerif/aeneas). I knew creating logos requires
a specific set of skills but really wanted to give it a try:
what about sketching some concept on pen and paper, then vectorizing it on
computer?  It proved even harder than I had expected: after spending a couple of
evenings using various techniques to sketch ideas which all ended up in the
garbage bin, I completely discarded the idea of doing something in
2D. Definitely too hard for now, especially as I don't have a clear idea of what
I want to achieve.

I then got the following idea: what about doing some sort of an antique coin,
with a sculpted profile? I could model the coin in 3D then stylize the render a
bit. That would actually be an excellent opportunity of going back to 3D
sculpting, which is something I hadn't done in many, many years. I had also just
replaced my >12 year old graphic tablet with a new one, and 3D sculpture was a
perfect opportunity to play with it. I thus set off on this project.

If you don't know what 3D sculpting looks like, here is an overview.
Traditionally, surfaces of 3D objects are modelled with triangles, which are
themselves often grouped in quads. As the modeller has to manipulate each vertex
one by one, those models often have a relatively low polycount. This polycount
varies with the topic or the context, for instance whether you model an object
for a video game or a movie quality render, which goes in the foreground or the
background, etc. For a face, I would often end up with a model made of a few
thousands triangles. In contrast, with 3D sculpting we start with a base shape,
often created with the above method, subdivide this shape a lot until it has
from hundreds of thousands to tens of millions triangles (and even more!), then
use brushes to sculpt this mesh. There exists various tools to sculpt in 3D: I
personally use [ZBrush](https://pixologic.com/).

For instance, you can see several subdivision levels of a sphere below. It
starts with 482 vertices and ends with more than 100k vertices (ZBrush doesn't
give the polycount in triangles). Whenever the subdivision level increases,
every (quad) face is split into four, and the angle between the edges is
smoothed. At some point you actually stop noticing that the surface is made of
flat faces. Note that on low-poly models, we usually render the faces in such a
way that they *look* smooth, to hide the fact that models are actually made of a
limited number of flat faces.

<p><div style="text-align: center; margin: 0; caption-side: bottom">
<img src="{{images}}sphere-subdivid.gif"
     alt="Subdividing a sphere" title="Subdividing a sphere"
     style="" width="400" align="center"/>
<figcaption>Subdividing a sphere</figcaption>
</div></p>

Once there are enough polygons, we can use brushes to sculpt the model by moving
groups of vertices. For this particular project I decided to go back to basics
by using a limited selection of brushes: I did everything with the 6 brushes
shown below.

<p><div style="text-align: center; margin: 0; caption-side: bottom">
<table cellspacing="0" style="border-collapse: collapse">
<tr style="background:rgba(0,0,0,0);">
  <td style="text-align: center">
    <img src="{{images}}standard.gif"
         alt="Standard Brush" title="Standard Brush"
         style="" width="235" align="center"/>
    <figcaption>Standard brush</figcaption>
  </td>
  <td style="text-align: center">
    <img src="{{images}}pinch.gif"
         alt="Pinch Brush" title="Pinch Brush"
         style="" width="235" align="center"/>
    <figcaption>Pinch brush</figcaption>
  </td>
  <td style="text-align: center">
    <img src="{{images}}slash.gif"
         alt="Slash Brush" title="Slash Brush"
         style="" width="235" align="center"/>
    <figcaption>Slash brush</figcaption>
  </td>
</tr>
<tr style="background:rgba(0,0,0,0);">
  <td style="text-align: center">
    <img src="{{images}}clay-buildup.gif"
         alt="Clay Buildup Brush" title="Clay Buildup Brush"
         style="" width="235" align="center"/>
    <figcaption>Clay Buildup brush</figcaption>
  </td>
  <td style="text-align: center">
    <img src="{{images}}smooth1.gif"
         alt="Smooth1 Brush" title="Smooth1 Brush"
         style="" width="235" align="center"/>
    <figcaption>Smooth brush</figcaption>
  </td>
  <td style="text-align: center">
    <img src="{{images}}flatten.gif"
         alt="Flatten Brush" title="Flatten Brush"
         style="" width="235" align="center"/>
    <figcaption>Flatten brush</figcaption>
  </td>
</tr>
<caption>Using brushes</caption>
</table>
</div></p>

Of course, there are plenty of brushes available by default in ZBrush. It is
also possible to tune the behavior of each brush by playing with various
parameters: the size and intensity of course, but also the stroke (see the
projection on the coin at the end of the post), or the alpha mask, which
influences the shape of the brush.

<p><div style="text-align: center; margin: 0; caption-side: bottom">
<img src="{{images}}all-brushes.jpg"
     alt="The default brush selection" title="The default brush selection"
     style="" width="900" align="center"/>
<figcaption>The default brush selection in ZBrush</figcaption>
</div></p>

In order to start the project, I needed a base shape from where to start
sculpting the head. I usually do all my models myself, including body parts for
the various creatures I make, but this time I wanted to start fast and
downloaded a base mesh for the head. I ended up choosing [this
one](https://www.turbosquid.com/fr/3d-models/free-basic-male-head-3d-model/644257)
on turbosquid, by the artist
[twiesner](https://www.turbosquid.com/fr/Search/Artists/twiesner?fbclid=IwAR27LSdh0GAlMghkSRWtJdBb7BwhxMo2GgBUhBeGz0Sc_5GtGTi7-MHccF8):
all credits to him for this model. And I can't thank enough those artists who make
assets such as models and textures freely available online: it helps a lot!

I started by using the move tool to change the morphology of the face to make it
closer to what I had in mind.

<p><div style="text-align: center; margin: 0; caption-side: bottom">
<img src="{{images}}move.gif"
     alt="Deforming the face with the Move brush" title="Deforming the face with the Move brush"
     style="" width="400" align="center"/>
<figcaption>Deforming the base face with the Move brush</figcaption>
</div></p>


I then worked on the details. I had to experiment a bit because I hadn't
sculpted anything in a long time. As I wanted to create a sculpted head, I also
aimed at a stylized look. As a result, I slowly converged towards the
following process. First, I used the clay buildup brush to create the overall
shapes by adding or substracting volumes. I then used the flatten brush to make
the surfaces less roundish, and give them the look of a sculpted surface.  Of
course, I also used the smooth brush a lot between the various steps. The video
below gives a good overview of the process I applied to the various parts
of the head.

<video style="float:center" width="960" height="540" controls>
  <source src="{{images}}standard-flatten2.mp4" type="video/mp4">
Your browser does not support the video tag.
</video> 

I also used the slash brush to dig crevices, for instance at the corner of the mouth,
around the nose or the eyes, together with the standard brush to give a
bit of roundness in some places, for instance on the eyelids. Finally, I used
the pinch brush a bit, in particular around the lips.

You can see some of the sculpting steps below: I placed the general
shapes then focused on the details (mouth, nose, eyes, ears, etc.), regularly
switching from one part to the other and progressively polishing the surfaces.

<p><div style="text-align: center; margin: 0; caption-side: bottom">
<img src="{{images}}sculpt-head.gif"
     alt="Sculpting the head" title="Sculpting the head"
     style="" width="600" align="center"/>
<figcaption>Sculpting the head</figcaption>
</div></p>

  <!-- TODO: put that in the CSS -->
  <p><div style="background: lightgray; border: 2px solid gray;">
  <div style="margin: 15px;">

  <p>
  <b>When I draw,</b> I generally reproduce a subject I have under my eyes and
  focus on the <i>way</i> I reproduce it: by stylizing it, simplifying it, etc.
  In contrast, when I do 3D modelling, I generally don't reproduce a specific
  subject but rather invent something. This doesn't mean that I don't use
  references of course, and I pay attention to anatomy a lot.
  </p>
  
  <p><div style="text-align: center; margin: 0; caption-side: bottom">
  <img src="{{images}}muscles.jpg"
       alt="Face muscles" title="Face muscles"
       style="" width="500" align="center"/>
  <figcaption>Some face muscles</figcaption>
  </div></p>

  <p>
  There are definitely too many bones and muscles in a human face (42 muscles!).
  When sculpting I always try to keep in mind the main ones, focusing on
  plausibility rather than exactitude. In the present case I think it overall
  looks ok, meaning I probably neither forgot nor invented too many muscles...
  </p>

  </div></div></p>


  <!-- TODO: put that in the CSS -->
  <p><div style="background: lightgray; border: 2px solid gray;">
  <div style="margin: 15px;">

  <p>
  <b>I knew I should have closed the mouth before sculpting!</b> Using
  brushes on open edges can lead to weird results, especially in the
  presence of sharp angles. As a consequence, I ended up with the following
  artifacts at some point: as you can see, some polygons at the corner of
  the lips pointed outwards while they were supposed to point inwards.
  </p>

  <p><div style="text-align: center">
  <img src="{{images}}mouth-pb2.jpg" alt="Mouth artifact" title="Mouth artifact"
       width="350"/>
  <img src="{{images}}mouth-pb1.jpg" alt="Mouth artifact" title="Mouth artifact"
       width="350"/>
  <figcaption>Artifacts at the corner of the mouth</figcaption>
  </div></p>


  <p>
  Fortunately, I was able to fix the issue by going to the slowest subdivision
  level, switching to Blender, moving the vertices there, and finally
  propagating the changes back to ZBrush. A bit of smoothing, and everything was
  back in order.
  </p>
  </div></div></p>

After doing the face, I worked on what I really dreaded: the hair. A sculpture
is supposed to be stylized, and I was really not sure about how to proceed to
capture the texture of the hair without doing too much. Again, I experimented a
bit and finally came up with the following process.  I used the standard brush
with varying sizes to create hair strands, and played with the slash brush to
add some details. By regularly switching between the two I was able to make the
hair gradually take shape. The pitch brush helped me make the tips of the hair
crisper, and at the end I used the standard brush with a bigger size to give
volume to some of the strands.

The alternation process between the standard brush and the slash brush is shown
in the video below.

<video style="float:center" width="960" height="540" controls>
  <source src="{{images}}sculpt-hair.mp4" type="video/mp4">
Your browser does not support the video tag.
</video> 

You can see some of the steps of sculpting the hair in the animation below:
I first created some rough strands, then added details, and finally gave them
more volume.

<p><div style="text-align: center; margin: 0; caption-side: bottom">
<img src="{{images}}sculpt-head-hair1.gif"
     alt="Sculpting the hair" title="Sculpting the hair"
     style="" width="600" align="center"/>
<figcaption>Sculpting the hair</figcaption>
</div></p>

Now that the head was done, I was left with the coin. Rather than using brushes
to sculpt in a free hand style, it is possible to use them to apply a single
pattern to a surface by changing the stroke (from Free Hand to Drag
Rectangle, see below). I used this technique to apply a projection of the head I had just
modelled onto a coin.

<p><div style="text-align: center; margin: 0; caption-side: bottom">
<img src="{{images}}drag-rect.jpg"
     alt="The drag-rect stroke" title="The drag-rect stroke"
     style="" width="600" align="center"/>
<figcaption>The Drag Rectangle stroke</figcaption>
</div></p>

I created a very simple coin model with a cylinder, then used the depth-map of
the face profile (i.e., the distance between the camera and the various points
of the head) to create an alpha texture for the standard brush. By using the
Drag Rectangle stroke, I was able to apply this projection onto the coin.
As you can see, the whiter pixels are closer to the camera, and correspond to
places of bigger volume. By using the light intensity of the depth map to control the
amount of displacement of the vertices, you can in effect project a volume
(i.e., the face) onto a surface.

<p><div style="text-align: center; margin: 0; caption-side: bottom">
<table cellspacing="0" style="border-collapse: collapse">
<tr style="background:rgba(0,0,0,0);">
  <td style="text-align: center">
    <img src="{{images}}face-depth.jpg"
         alt="The depth map" title="The depth map"
         style="" height="300" align="center"/>
    <figcaption>The depth map</figcaption>
  </td>
  <td style="text-align: center">
    <img src="{{images}}project3.gif"
         alt="Projecting the head" title="Projecting the head"
         style="" width="300" align="center"/>
    <figcaption>Projecting the head on a coin</figcaption>
  </td>
</tr>
</table>
</div></p>


I then used the symmetry options to do the pourtour of the coin, before
leveraging the move tool and some other brushes to create a bit of wear and
tear. I also made a few attempts at adding finer details like erosion but
finally decided not to do so to keep the look relatively simple.

<p><div style="text-align: center; margin: 0; caption-side: bottom">
<img src="{{images}}coin-symmetry.jpg"
     alt="Sculpting the coin" title="Sculpting the coin"
     style="" width="600" align="center"/>
<figcaption>Sculpting the coin by using radial symmetry</figcaption>
</div></p>

I finally rendered the coin, applied a few filters and adjustments in
[GIMP](https://www.gimp.org/) to get the current look and I was done! I hope you
enjoyed this post.

<p><div style="text-align: center; margin: 0; caption-side: bottom">
<img src="{{images}}final-coin.jpg"
     alt="The final logo" title="The final logo"
     style="" width="400" align="center"/>
<figcaption>The final logo</figcaption>
</div></p>
