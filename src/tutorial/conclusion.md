# Resources & Conclusion




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
  
* [Mike Bailey](https://web.engr.oregonstate.edu/~mjb/vulkan/)'s Vulkan page provides Vulkan material licensed under a CC4 ATT/NC/ND license.  He also links to a very good summary of Vulkan's structures called *Vulkan in 30 minutes*, available [here](https://renderdoc.org/vulkan-in-30-minutes.html)
