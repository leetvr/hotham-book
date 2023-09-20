# Introduction

G'day, Matt from Spark Reactor here.  Welcome to Hotham!

![Spark Reactor](../images/spark-reactor-icon.png) ![Oculus](../images/quest2-icon.png) ![Hotham](../images/hotham-icon.png) ![Vulkan](../images/vulkan-logo-scaled.png)

I'll be your friendly tutor here as we guide you through the process of learning about the wonderful world of 3D graphics, OpenXR and Vulkan!

The goal of this series of tutorials is to provide an overview of how to accomplish a number of common tasks in developing virtual reality apps using Hotham.  These techniques will be useful not only for ordinary game development, but for other apps that require realistic physics and access to the Vulkan API.

As an example, an app wanting to display a pdf or view a wall of images sourced from the local device or the web will need to display 2 dimensional content on a planar or curved surface.  In the current version of Hotham, this is a non trivial task as it involves understanding the Vulkan context, the shaders and how the renderer deconstructs and uses meshes, materials and textures together.

The goal is therefore to demystify the processs by providing a clear explanation of how the multiple primitive components of Hotham work together.

I am working primarily with the Oculus Quest 2, but this tutorial aims to be as generic as possible and the concepts discussed within are applicable to any application utilising the 3D rendering and physics functionality which Hotham provides.

This version of the tutorial covers Hotham 0.2.0; if you are following
this further down the track, aspects of the process such as compilation
or minor aspects of the API may have changed.  I will endeavour to keep these
tutorials in line with the current version of Hotham, however, if you get
stuck, jump on the Discord linked in the [Getting Started](https://github.com/leetvr/hotham/wiki/Getting-started) guide to find someone to help you out :-)

I hope you enjoy your learning journey!
