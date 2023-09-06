# The Vulkan Context

`engine.vulkan_context` exposes the Vulkan context, type `VulkanContext`, which provides access to the Vulkan instance for the engine.  It is one half of the equation which will provide assistance in developing custom rendering solutions.

Within the `VulkanContext`, the `device` attribute is the one you will likely make use of the most, providing access to the Vulkan device API for commands such as `create_pipeline_layout`, `create_render_pass`, and `create_graphics_pipelines`.  This is also the home of commands such as `cmd_begin_render_pass`, `cmd_bind_pipeline`, and `cmd_bind_descriptor_sets` along with `cmd_bind_vertex_buffers` and `cmd_bind_index_buffer`.  When recording commands you will be most likely using the `systems::rendering::draw_primitive` function which calls the `cmd_push_constants` and `cmd_draw_indexed` functions on the device.

While using the `VulkanContext`, you will most likely want to import `hotham::ash::vk` to ensure the version of ash you have loaded is correct.  Within the `vk` namespace, enums and builder patterns for many of the structs you will need to use in constructing and managing a pipeline are located.

The `VulkanContext` also has a number of key functions which will be useful.  We'll examine some of those now.

# Creating images

`engine.vulkan_context.create_image` is used for creating a new image that can be used in a framebuffer.  It does nothing other than creating the image.  This includes all of the Vulkan technical aspects of making assumptions of the type of image format to use, sharing mode, number of samples and so on.  It also deals with creating an image view for the image.  It handles the lower level technical implementation of allocating and binding memory for the Image as well.

The object returned is a `Result<Image>`, where `Image` is a `hotham::rendering::image::Image`.  This is essentially a wrapper around the local `vk::Image` which gives you access to the image view, the format, extent size, usage flags and any other information associated with the image creation.

Note that this function does absolutely nothing other than what is described above.  If you are creating an image for a particular purpose such as, lets say, for use as a texture, you probably want a more specific function like `Texture::empty` which makes calls to `create_texture_image` on the render context to add the newly created texture image to the descriptor set which can be accessed on the graphics pipeline.

# Loading/updating images

`engine.vulkan_context.upload_image` takes an `&[u8]` as its first parameter being the location of a buffer owned by the caller.  The second parameter, the mip count, usually remains as 1, unless you have reason to change it.  Similarly, the offsets array in parameter 3 is usually a `vec![0]` in most cases.  The final parameter is the `&Image` itself.  If you have created an empty texture, or want to modify an existing one, for example, the `rendering::texture::Texture` struct contains a field `image` containing the created Vulkan image to which you can upload content.  You will typically want to have created the image with appropriate flags.

This function creates a buffer using the data in the first parameter, with usage flags for the buffer being a `TRANSFER_SRC`, and then transitions the image layout to `TRANSFER_DST_OPTIMAL` before copying the buffer into the destination image, transitioning the image back to `SHADER_READ_ONLY_OPTIMAL` and destroying the staging buffer.

I recommend using this function where possible to update textures, as it takes care of the details of image transitioning and ensuring the image is in a state optimal to receive the data from your local buffer.

The functions `copy_buffer_to_image` and `copy_image_to_buffer` are specialized functions and should be used in conjunction with transitioning the image format and ensuring correct buffer usage flags.  You should also ensure any buffer used with these functions is of appropriate size.

# Buffers

Other than the previously mentioned `copy_buffer_to_image` and `copy_image_to_buffer`, there is also `create_buffer_with_data` and a deprecated function `update_buffer`.  However, in most cases, rather than using the `VulkanContext` itself to create buffers, you're best to use `hotham::rendering::buffer::Buffer` which includes functions for wrapping a buffer while providing access to the underlying vk::Buffer handle, determining if the buffer is empty, and updating descriptor sets with respect to a buffer.

# Vulkan Validation Layers

Although not part of the `vulkan_context` itself, many people developing in hotham may want to enable Vulkan validation layers and be unaware of how to do this.  On the Oculus Quest / Meta Quest and several other devices, this can be done by executing the following command:

```shell
adb shell setprop debug.oculus.loadandinjectpackagedvvl.my.package.name 1
```

In this example, `my.package.name` is the name of the installed package you are debugging.  It can be obtained by running `pm list packages -3` if you are unsure of the name of your program's package.

After setting this property, `adb logcat -s VALIDATION` will show all warnings and other messages from the validation layers.
