# Resources & Conclusion

This set of 13 introductory lessons barely scratched the surface on how to use the library. Things such as the audio context and the
animation context have not been covered as they are rather fairly trivial thin wrappers around existing third party API's such as 
oddio, and their API's do not require extensive knowledge of computer graphics to make efficient use of.

Hotham at its current stage of 0.2.0 is a library rather than a huge hulking framework, and this often means there is no right or
wrong way to do things with the library.  The key pattern is to mix and match the systems you need to develop your solution, and
implement those that do not exist or which are found wanting in their existing implementation.

This degree of freedom means the potential for novel shaders to be used in conjunction with the existing world/physics/model loading
systems, and to dynamically choose a series of created pipelines depending on the existing world-situation.

To take hotham to its higher potential, knowledge of existing emergent paradigms of GPU based rendering which have evolved over
the past 15-20 years is useful, including the standards that have arisen through the work of the Khronos Group, such as OpenGL,
and Vulkan, as well as file formats such as KTX2 and GLB/GLTF.

It can be extremely daunting when first encountering the world of GPU based rendering.  To help those new to the field, I have
included a short set of references below that I have found personally useful in one form or another, or that I am aware are somewhat
definitive within the field.  I highly recommend going full-immersion into this type of material for a month or more and getting
the mental muscle memory required to make these concepts less formidable.  Once you have developed the necessary abstractions,
your own personal libraries to complement Hotham's low level exposure of a number of complex APIs, you can move forward with
the real work of developing your perfect immersive UX.

If there is call for a video version of these tutorials to explain the concepts more in depth, these may be forthcoming in
the future, however in the meantime feel free to email me if you have any questions, or better yet, come on over to the Discord
and meet others who are using the library to get some help!

Thanks for hanging out with us here, and good luck on the rest of your journey!

Matt from Spark Reactor, September 2023.

![Spark Reactor](../images/spark-reactor-icon.png) ![Oculus](../images/quest2-icon.png) ![Hotham](../images/hotham-icon.png) ![Vulkan](../images/vulkan-logo-scaled.png)

## Books which develop skills in computer graphics

* **Pawel Lapinski**'s [Vulkan Cookbook](https://www.amazon.com/Vulkan-Cookbook-potential-generation-graphics/dp/1786468158) is a recipe based book on Vulkan that nevertheless also provides coherent explanations for all of its recipes.
* **Marco Castorina** and **Gabriel Sassone** created [Mastering Graphics Programming with Vulkan](https://www.amazon.com.au/Mastering-Graphics-Programming-Vulkan-state/dp/1803244798), a book on creating a rendering engine from first principles using Vulkan.
* **Graham Sellers** and **John Kessenich** created [The Vulkan Programming Guide](https://www.vulkanprogrammingguide.com/), the official guide to learning Vulkan
* Multiple authors including **Tomas Akenine-Moller** contributed to produce [Real Time Rendering](https://www.realtimerendering.com/), the resource page linked above includes chapters on collision detection and ray tracing.  They also provide a [book recommendations page](https://www.realtimerendering.com/books.html) which includes a lot of **free** books such as Principles of Digital Image Synthesis, Immersive Linear Algebra, and more.
* **Richard S. Wright*, **Nicholas Haemel** and others contributed to the [OpenGL SuperBible](https://www.opengl.org/sdk/docs/books/SuperBible/), now in its sixth edition.  This is *the book* on OpenGL, with principles that are relevant to Vulkan and GLSL shaders.
* Eric Lengyell produced [Foundations of Game Engine Development](https://foundationsofgameenginedev.com/), a four volume series dedicated to the algorithms and mathematical underpinnings of the craft.

## Computer graphics: Transformations

* GPU Open has a [Matrix Compendium](https://gpuopen.com/learn/matrix-compendium/matrix-compendium-intro/), a collection of information on matrix math for transformations in one place.
* [This Youtube tutorial](https://www.youtube.com/watch?v=zjMuIxRvygQ) links to an [interactive series](https://eater.net/quaternions) on visualising quaternions and 3d rotation.

## Shaders and OpenGL

* [The Book of Shaders](https://thebookofshaders.com/) is a good site for information on programmatic fragment shaders.
* [Inigo Quilez](https://iquilezles.org/articles/) has articles on numerous computer graphics related topics, also focusing on fragment shaders.
* [Learn OpenGL](https://learnopengl.com/) is a resource devoted to, as the name suggests, learning about OpenGL.  Both OpenGL and Vulkan were
developed by Khronos Group and share a number of key similarities especially with respect to the use of shaders and the application of GPU
parallelism concepts to these programs which lie at the heart of real time rendering.
* [glslEditor](https://github.com/patriciogonzalezvivo/glslEditor) is a project which lets you develop programmatic fragment shaders in real time, and
is available to use live at [this website address](http://editor.thebookofshaders.com/)

## Vulkan Resources

* The original resource for learning Vulkan is the well known [Vulkan tutorial](https://vulkan-tutorial.com/), which focuses on drawing a single
triangle before moving on to slightly more advanced topics such as mipmaps, multisampling and depth buffering.
* Another good written tutorial is [VulkanGuide](https://vkguide.dev/), which has an excellent selection of tutorials and links to relevant
websites such as GPU open, different sets of Vulkan samples and more.
* Khronos Group's [Github Page](https://github.khronos.org/) lists all of the official Khronos repositories including the [Vulkan Samples](https://github.com/KhronosGroup/Vulkan-Samples) github.  This includes multiple gigabytes of examples in different languages.
* [Vulkan Tutorial in Rust](https://github.com/unknownue/vulkan-tutorial-rust) is the above Vulkan tutorial which has been converted to Rust and Ash. This is worth a look to see the coding techniques used and the API calls translated into design patterns that can be replicated in your own applications.
* The GPU Open website has [a section](https://gpuopen.com/learn/developing-vulkan-apps/) dedicated to developing Vulkan applications.  This includes blog posts, sample code, libraries and tools.
* The [Vulkan Youtube channel](https://www.youtube.com/@Vulkan) has a variety of talks which help to shed light on difficult topics.  Within the last year, this channel has also posted a video from Vulkanised 2023 with a list of developer resources.  Other useful talks: 
  * [The low level mysteries of pipeline barriers](https://www.youtube.com/watch?v=aIR3x_X92y8)
  * [Vulkan subpasses](https://www.youtube.com/watch?v=M0upwYZqRMI)
  * [Render passes in Vulkan](https://www.youtube.com/watch?v=yeKxsmlvvus)
* Brendan Galea has a series of 31 videos on developing a game engine using Vulkan which covers the material of the Vulkan tutorial and more in a coherent, code-driven way.  You can find his tutorial [here](https://www.youtube.com/playlist?list=PL8327DO66nu9qYVKLDmdLW_84-yE4auCR)
* [Voxelphile](https://www.youtube.com/@Voxelphile/videos) on Youtube has a number of videos on Vulkan in Rust, including:
  * [Vulkan graphics pipeline and buffer creation](https://youtu.be/h0CvNOLIggY)
  * [How to make a Renderer (in Rust)](https://youtu.be/qkdy5yL-EjU)
* [Tantan](https://www.youtube.com/@Tantandev/videos)'s videos on writing a Voxel engine in Rust are IMO the best videos I've seen on the topic, and I recommend keeping an eye on this channel for useful content.  The author is using Bevy for their implementation, but the concepts are OpenGL/Vulkan related at their core.
* [Mike Bailey](https://web.engr.oregonstate.edu/~mjb/vulkan/)'s Vulkan page provides Vulkan material licensed under a CC4 ATT/NC/ND license.  He also links to a very good summary of Vulkan's structures called *Vulkan in 30 minutes*, available [here](https://renderdoc.org/vulkan-in-30-minutes.html)
