---
# MARP
theme: default
paginate: true

# Jekyll
title: "Windows and Points"
date: 2022-04-12
categories:
  - Notes
classes: wide
toc_sticky: false
---

*Chapters 2.1 and 2.2 introduce the basic framework components for creating an application window and rendering a point on the screen.*

The graphics pipeline model we will follow has the following four stages:
1. The **Application Stage** is where we create a window to display the rendered graphics and then send data to the GPU.
2. **Geometry Processing** is where we create a *vertex shader* program to determine the positions of each vertex of each object to be rendered.
3. **Rasterization** is where we determine which pixels will render which objects in the scene.
4. **Pixel Processing** is where we create a *fragment shader* program to find the color of each pixel.

The tasks of the **Application Stage** include:  
- **Create a window to read from the GPU framebuffer and render graphics.**
- **Maintain input-checking and animation loop.**
- Run algorithms for physics simulations and collision detection.
- **Send vertex attributes and shader source code to the GPU for rendering.**  

The first two we accomplish with Pygame in chapter 2.1 while the fourth is addressed in chapter 2.2 below. 

Then in **Geometry Processing**, the **vertex shader** calculates transformations on geometric objects and gets their final coordinates for rendering.

**Rasterization** then creates *fragments* (color data for each pixel) which includes their *raster position* (pixel coordinate).

Finally, the **Pixel Processing** step applies the **fragment shader** program to calculate the final color of each pixel.

The **vertex shader** we write in chapter 2.2 below will only render a single point while the **fragment shader** will set the pixel color to a fixed value, so we do not need to worry about **geometry processing** or **rasterization** for now.

The graphics framework we begin building here will follow good software engineering design practices, such as **reusability** and **extensibility**. Each component of the graphics framework is designed to have a single responsibility and be open to extensions.  

# 2.1 - Creating Windows with Pygame

In the **application stage**, we will use the Pygame library to create the application window.

The first component of the graphics framework will be a `WindowApp` class which handles the application lifecycle:
- **Startup** where we load external files, initialize values, and create programming objects.
- **Main Loop** which repeatedly checks input from the user, updates values of variables and objects, and renders graphics on the screen.
- **Shutdown** which cancels any running processes and closes the window.

![The interactive graphics application lifecycle](/software-engineering-lab/assets/images/igraphics-app-flow.png)

Let's create our first Python package to hold the core components of the framework.

## The `WindowApp` Class

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> First, make a folder on your computer to store all the source code. (I called mine `graphics` and put it inside a folder just for this class.) This will be your **main** working folder.  
<input type="checkbox" class="checkbox inline"> In your main folder, create a new folder called `core`.  
<input type="checkbox" class="checkbox inline"> Inside the `core` folder, create an empty file called `__init__.py` (with double underscores). This will let you import code from the `core` folder as Python modules.  
<input type="checkbox" class="checkbox inline"> Create another file called `app.py` inside `core` and open the file for editing.  
<input type="checkbox" class="checkbox inline"> Add the following code to the `app.py` file.

```python
# core.app.py
import pygame
import sys

class WindowApp:
    """A basic application window for rendering 3D graphics."""
    def __init__(self, screen_size=(512, 512)):
        # initialize all pygame modules
        pygame.init()
        #indicate rendering details
        display_flags = pygame.OPENGL | pygame.DOUBLEBUF
        # initialize buffers to perform antialiasing
        pygame.display.gl_set_attribute(pygame.GL_MULTISAMPLEBUFFERS, 1)
        pygame.display.gl_set_attribute(pygame.GL_MULTISAMPLESAMPLES, 4)
        # use a core OpenGL profile for cross-platform compatibility
        pygame.display.gl_set_attribute(pygame.GL_CONTEXT_PROFILE_MASK, 
                                        pygame.GL_CONTEXT_PROFILE_CORE)
        # create and display the window
        self.screen = pygame.display.set_mode(screen_size, display_flags)
        # set the text that appears in the title bar of the window
        pygame.display.set_caption("Graphics Window")

        # manage time-related data and operations
        self.clock = pygame.time.Clock()
```

This code creates an initialization method for our `WindowApp` class which prepares the necessary Pygame modules with specific buffer settings before creating a window. The window size is set by the `screen_size` parameter which has a default 512 x 512 resolution. We create the screen with the `pygame.display.set_mode()` method and save a reference to it in the `self.screen` property. 

Additional options for the display screen are set by `display_flags` which is currently set to create a display for OpenGL with double buffering. We can combine flags with the binary OR operator `|` when we want to use multiple options. For example, the following code would create an OpenGL display with double buffering that can change sizes:

```python
        display_flags = pygame.OPENGL | pygame.DOUBLEBUF | pygame.RESIZABLE
```

(See the [`pygame.display`](https://www.pygame.org/docs/ref/display.html#pygame.display.set_mode){:target="_blank"} documentation for more display modes and their descriptions.)

The `GL_MULTISAMPLEBUFFERS` and `GL_MULTISAMPLESAMPLES` attributes control *antialiasing*, which is a technique to make the edges of polygons appear smooth and not pixelated. The number of samples indicates how many times a pixel at the edge of a polygon is sampled. Sampling a pixel means we calculate the color of that pixel and each time we do it at a slight offset, so the pixel becomes a blend of the polygon color and other colors around it.

Finally, the `pygame.time.Clock()` object will be used to manage the frames per second (FPS). The main loop of the program will render a new frame everytime it completes a cycle. As a result, the FPS can be really high on good hardware and use up all the CPU power. But most computer displays only run at 60 Hz which means the image updates only 60 times per second, so we should conserve our CPU and set the FPS of our application to 60 as well.

Next, we will add two empty methods to the `WindowApp` class. The intention of these methods is to implement the application lifecycle in classes that extend the `WindowApp` class. 

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Add the following code inside the `WindowApp` class after the `__init__()` method.

```python
    def startup(self):
        pass
        
    def update(self):
        pass
```

Every new application will extend these two methods in the `WindowApp` class to define their behavior. By implementing `startup` and `update`, an application can define its own **Startup** and **Update** processes in its lifecycle (as shown in the flowchart at the beginning of this section).

The last method we will add to `WindowApp` is the one that will operate the entire application loop of startup, check input, update, and render.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Add the following code inside the `WindowApp` class.

```python
    def run(self):
        # control the main loop with a Boolean
        running = True

        # the "startup" process
        self.startup()

        # the main loop
        while running:

            # the "check inputs" process
            # !--not implemented yet--!

            # the "update" process
            self.update()

            # the "render" process
            pygame.display.flip()

            # maintain 60 FPS
            self.clock.tick(60)

        # the "shutdown" process
        pygame.quit()
        sys.exit()
```

We will call this `run` method when we want to start a new application. It contains all the primary processes of an interactive application except for checking inputs. 

## The `Input` Class

A separate class will handle inputs by implementing the default behavior for all applications. For now, that behavior is just to exit the program when the window closes.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `core` folder, open the file called `app.py`.  
<input type="checkbox" class="checkbox inline"> Add the following source code to `app.py` **before** the `WindowApp` class:

```python
class Input:
    """Handles the inputs for an application, such as keys from the keyboard."""
    def __init__(self):
        # tells whether the user has closed the application
        self._quit = False

    @property
    def quit(self):
        return self._quit
```

Here we created a private attribute in the `Input` class called `_quit`. We want to keep control the of this Boolean's value inside the `Input` class, so we created a special method called a **getter** with the `@property` decorator. Now other classes cannot change `_quit` but they can read its value. The `Input` class will change the Boolean to `True` when it receives an event of type `pygame.QUIT` in the source code below.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Add the following `update` method to the `Input` class.

```python
    def update(self):
        # pygame events come in a list, so we should check them all
        for event in pygame.event.get():
            # clicking the close button will send an event of type QUIT
            if event.type == pygame.QUIT:
                self._quit = True
```

Next, we need to use an instance of `Input` in our `WindowApp` class to implement the **check inputs** process and complete the **main loop**.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `app.py` file, scroll down to the `WindowApp` class.  
<input type="checkbox" class="checkbox inline"> At the end of the `__init__` method, create an instance of `Input` and save it to a class attribute.  

```python
        # handle user inputs
        self.input = Input()
```

<input type="checkbox" class="checkbox inline"> Delete the `# !--not implemented yet--!` comment inside the `run` method.  
<input type="checkbox" class="checkbox inline"> Add the following code in place under `# the "check inputs" process`:  

```python
            # the "check inputs" process
            self.input.update()
            if self.input.quit:
                running = False
                break
```

## Testing the `WindowApp` and `Input` Classes

Finally, let's check that everything is working together just fine with a simple test program. This program will only initialize the `WindowApp` class and run it to make a window appear on the screen.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main folder, create a file called `test_2_1.py`.  
<input type="checkbox" class="checkbox inline"> Inside that file, add the following source code:

```python
# test_2_1.py
from core.app import WindowApp

class Test_2_1(WindowApp):
    """Tests a basic window for rendering content."""
    def startup(self):
        print("Starting up Test 2-1...")

    def update(self):
        pass

# instantiate and run this test
Test_2_1().run()
```

<input type="checkbox" class="checkbox inline"> Save the file and open a terminal in your main folder.  
<input type="checkbox" class="checkbox inline"> Run the test program with the following command:  

```bash
$ python test_2_1.py
```

<input type="checkbox" class="checkbox inline"> Confirm that the program exits when you click the close button on the window.  

In this test, the `startup` and `update` methods don't really do anything. We included them here just to practice our approach of implementing those methods when developing applications with our framework. The next test we write will have much more interesting content for those two methods.

# 2.2 - Drawing a Point

With OpenGL, we render images on screen by running a program on the graphics processing unit (GPU). The program is a combination of a **vertex shader** and a **fragment shader** written in the OpenGL Shading Language (GLSL). This chapter shows how to create simple shaders, link them, and compile them into a GPU program. After that, we extend our graphics framework with components for these tasks.

## The OpenGL Shading Language (GLSL)

**GLSL** is a programming language with similarities to both C and Python. Like C, we must declare data types for every variable, such as `bool`, `int`, and `float`. In addition to these basic data types, GLSL also has vector data types to store two, three, or four float values in a single variable. These vector types are `vec2`, `vec3`, and `vec4`, respectively. Vectors are useful for storing data related to colors ($r$,$g$, $b$) and coordinates ($x$, $y$, $z$). GLSL also has matrix data types which provide the data structure required for many important CG computations, such as transformations.

Similar to Python, GLSL has all the basic control structures of `if` statements, loops, and functions with parameters. However, it uses arrays instead of lists and also provides structs for users to define their own data types.

We can access values in a vector using either an index or dot notation. For example, with a `vec4` named `v`, each value can be indicated by its index in the array (`v[0]`, `v[1]`, `v[2]`, `v[3]`) or with dot notation: (`v.x`, `v.y`, `v.z`, `v.w`) or (`v.r`, `v.g`, `v.b`, `v.a`) or (`v.s`, `v.t`, `v.p`, `v.q`). As a programmer, you can freely decide how you want to access each value. Just remember that dot notation methods are semantically associated with position coordinates $(x, y, z, w)$, colors $(r, g, b, a)$, and texture coordinates $(s, t, p, q)$.

Each program written in GLSL must have a `main` function to run, similar to C language. We always declare functions with a return type, which is `void` for the `main` function. An example of a simple GLSL program is the **vertex shader** for our first application. This program creates a single point on screen and saves its position to a global variable called [`gl_Position`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/gl_Position.xhtml){:target="_blank"}. This variable is defined by the OpenGL library and must be assigned a value in the vertex shader.

```glsl
# GLSL version 330
void main() {
    gl_Position = vec4(0.0, 0.0, 0.0, 1.0);
}
```

The position of our pixel is stored as a 4-dimensional vector with coordinates $x$, $y$, $z$, and $w$. The $x$, $y$, and $z$ coordinates describe the pixel's position on the respective axes. The $w$ coordinate is used for geometric transformations, but we don't need to worry about that now.

Next, our **fragment shader** simply sets the color of our pixel.

```glsl
# GLSL version 330
out vec4 fragColor;
void main() {
    fragColor = vec4(1.0, 1.0, 0.0, 1.0);
}
```

The `fragColor` variable is a 4-dimensional vector of color values $r$, $g$, $b$, and $a$ defined as `float` values. The range of possible values in shaders is between -1.0 and 1.0 and cannot be replaced with `int` values.

Here, the variable `fragColor` is declared with `out` which is a *type qualifier*. This means that the `vec4 fragColor` variable will be sent to the buffer after the fragment shader finishes. In general, the `out` qualifier shows that a variable will be sent to the next step, and the `in` qualifier shows that it is received from a previous step. For example, color data can be passed from our application through the color buffer to the vertex shader, then the fragment shader, and back to the buffer again for rendering.

![The ins and outs of a GPU program](/software-engineering-lab/assets/images/buffer-shader-dataflow.png)

This diagram shows how data can flow through the GPU program. The application program can define `colorData`, send it to the vertex shader as `vertexColor`, then pass that to the fragment shader as `color` where it becomes a `vec4` called `fragColor` and goes back to the color buffer for rendering. We will use this design much later when we render shapes with multiple colors, but for now our fragment shader defines its own fixed color value and sends it to the buffer.

## Compiling GPU Programs

Before we can create our first rendering of a yellow pixel, let's build some utilities to handle the common tasks of compiling and linking our shaders to create a GPU program. We will do this by creating *static methods* (methods that do not use an instance of their class) in a class called `OpenGLUtils`. These methods will load and compile shader code, link shaders, and compile the GPU program. We will use many OpenGL functions from the `GL` namespace which have `gl` at the beginning of their names.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your `core` folder, create a file called `openGLUtils.py`.  
<input type="checkbox" class="checkbox inline"> Open `openGLUtils.py` and add the following code:

```python
# core.openGLUtils.py
import OpenGL.GL as GL

class OpenGLUtils:
    """Static methods to compile OpenGL shaders and link them to create GPU programs."""
    @staticmethod
    def initialize_shader(shader_code, shader_type):
        # specify required OpenGL_GLSL version
        shader_code = f"#version 330\n{shader_code}"

        # create an empty shader object and return its reference value
        shader_ref = GL.glCreateShader(shader_type)
        # load the source code into the shader
        GL.glShaderSource(shader_ref, shader_code)
        # compile the shader
        GL.glCompileShader(shader_ref)

        # check if the shader compile was successful
        compile_success = GL.glGetShaderiv(shader_ref, GL.GL_COMPILE_STATUS)
        if not compile_success:
            # get an error message from the shader as a byte string
            error_message = GL.glGetShaderInfoLog(shader_ref)
            # convert the byte string to a character string
            error_message = f"\n{error_message.decode('utf-8')}"
            # free memory used to store the shader program
            GL.glDeleteShader(shader_ref)
            # raise an exception to top running and print the error message
            raise Exception(error_message)
        
        # compilation was successful, so return the shader reference
        return shader_ref
```

Here we use the `@staticmethod` decorator to make the `initialize_shader` method static. Inside the method, we use [`glCreateShader`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glCreateShader.xhtml){:target="_blank"} to initialize a shader object and get its reference. Then, we load the source code into the shader with [`glShaderSource`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glShaderSource.xhtml){:target="_blank"} and compile the code with [`glCompileShader`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glCompileShader.xhtml){:target="_blank"}. We then check the compiler status to see if it failed with [`glGetShaderiv`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glGetShader.xhtml){:target="_blank"}. If compilation failed, we get the error message with [`glGetShaderInfoLog`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glGetShaderInfoLog.xhtml){:target="_blank"} before deleting the shader program with [`glDeleteShader`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDeleteShader.xhtml){:target="_blank"}. Otherwise, our method returns a reference to the shader object.

Next we need a method for linking the shaders and compiling the GPU program.  

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Add the next method to the `OpenGLUtils` class:  

```python
    @staticmethod
    def initialize_program(vertex_shader_code, fragment_shader_code):
        # use our previous method to load and compile the shaders
        vertex_shader_ref = OpenGLUtils.initialize_shader(
            vertex_shader_code, 
            GL.GL_VERTEX_SHADER
        )
        fragment_shader_ref = OpenGLUtils.initialize_shader(
            fragment_shader_code, 
            GL.GL_FRAGMENT_SHADER
        )

        # create an empty program object and get its reference value
        program_ref = GL.glCreateProgram()

        # attach the previously compiled shaders
        GL.glAttachShader(program_ref, vertex_shader_ref)
        GL.glAttachShader(program_ref, fragment_shader_ref)

        # link the vertex shader to the fragment shader
        GL.glLinkProgram(program_ref)

        # check if the program link was successful
        link_success = GL.glGetProgramiv(program_ref, GL.GL_LINK_STATUS)
        if not link_success:
            # get an error message from the program compiler
            error_message = GL.glGetProgramInfoLog(program_ref)
            # convert byte string to a character string
            error_message = f"\n{error_message.decode('utf-8')}"
            # free memory used to store GPU program
            GL.glDeleteProgram(program_ref)
            # raise an exception to stop running and print the error message
            raise Exception(error_message)

        # linking was successful, so return the program reference
        return program_ref
```

This method uses our `initialize_shader` method to load and compile the vertex shader and fragment shader from strings of their source code. Then we create an object for the GPU program with [`glCreateProgram`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glCreateProgram.xhtml){:target="_blank"} and attach the shaders to be linked with [`glAttachShader`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glAttachShader.xhtml){:target="_blank"}. Then, [`glLinkProgram`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glLinkProgram.xhtml){:target="_blank"} uses the shaders to create the GPU program. Linking the program may fail, so we check its status with [`glGetProgramiv`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glGetProgram.xhtml){:target="_blank"}, similar to the shaders. If it failed we use [`glGetProgramInfoLog`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glGetProgramInfoLog.xhtml){:target="_blank"} to get the error message before deleting the program with [`glDeleteProgram`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDeleteProgram.xhtml){:target="_blank"}. Otherwise, we return a reference to the program object.

<!-- The last utility method prints information about the running system to tell which version of OpenGL/GLSL it supports.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Add the next method to the `OpenGLUtils` class:  

```python
    @staticmethod
    def printSystemInfo():
        print(f"  Vendor: {glGetString(GL_VENDOR).decode('utf-8')}")
        print(f"  Renderer: {glGetString(GL_RENDERER).decode('utf-8')}")
        print(f"  OpenGL version supported: {glGetString(GL_VERSION).decode('utf-8')}")
        print(f"  GLSL version supported: {glGetString(GL_SHADING_LANGUAGE_VERSION).decode('utf-8')}")
```

This last method may be interesting to run in the Python interpreter if you want to see the version information for yourself. -->

## Rendering in the Application

Finally, we can create an application that renders a point using our `OpenGLUtils` methods and a few others from the OpenGL library.

A typical application will have the following steps within its **startup** process:

1. Store vertex attributes in GPU memory buffers called *vertex buffer objects* (VBOs).  
2. Send source code for the vertex shader and fragment shader to the GPU.
3. Compile and load the shader source code.
4. Link vertex data in VBOs to variables in the the vertex shader.

In our applications, we do these steps in the `startup` method which is defined in the `WindowApp` class but implemented by each application. We will use special objects called *vertex array objects* (VAOs) that can hold multiple associations between VBOs and shader variables. OpenGL often requires at least one VAO. Our applications are simple and don't need to set up any buffers, so the vertex and color data will come straight from our code instead.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main folder, create a new file called `test_2_2.py`.  
<input type="checkbox" class="checkbox inline"> Open the `test_2_2.py` file and add the following code:

```python
# test_2_2.py
import OpenGL.GL as GL

from core.app import WindowApp
from core.openGLUtils import OpenGLUtils

class Test_2_2(WindowApp):
    """Test GPU program compiling and linking by rendering a single point."""
    def startup(self):
        print("Starting up Test 2-2...")

        # vertex shader code as a character string
        vs_code = """
        void main() {
            gl_Position = vec4(0.0, 0.0, 0.0, 1.0);
        }
        """

        # fragment shader code as a character string
        fs_code = """
        out vec4 fragColor;
        void main() {
            fragColor = vec4(1.0, 1.0, 0.0, 0.0);
        }
        """

        # compile the GPU program and save its reference
        self.program_ref = OpenGLUtils.initialize_program(vs_code, fs_code)

        # create and bind vertex array object (VAO)
        vao_ref = GL.glGenVertexArrays(1)
        GL.glBindVertexArray(vao_ref)

        # (optional) set point size (width and height in pixels)
        GL.glPointSize(10)
    
    def update(self):
        # select program to use when rendering
        GL.glUseProgram(self.program_ref)

        # render geometric objects using the selected program
        GL.glDrawArrays(GL.GL_POINTS, 0, 1)

# instantiate and run this test
Test_2_2().run()
```

<input type="checkbox" class="checkbox inline"> Save the file and open a terminal in your main folder.  
<input type="checkbox" class="checkbox inline"> Run the test program with the following command:  

```bash
$ python test_2_2.py
```

<input type="checkbox" class="checkbox inline"> Confirm that the program displays a yellow dot in the center of the window.  

This test application once again inherits from our `WindowApp` class. It implements the `startup` method for its startup process and the `update` method for its update process.

The `startup` method directly stores the source code for the shaders as multi-line strings and uses our `OpenGLUtils.initialize_program` method to compile and link the GPU program. Then it gets a VAO with [`glGenVertexArrays`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glGenVertexArrays.xhtml){:target="_blank"} and binds it for use with [`glBindVertexArray`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glBindVertexArray.xhtml){:target="_blank"}. The last line is to optionally change the rendering size of the point with [`glPointSize`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glPointSize.xhtml){:target="_blank"} so that it is easier to see. Here, the parameter of `10` means the point will be 10 pixels wide and 10 pixels tall.

In the `update` method, we indicate that the **render** process should use our program with [`glUseProgram`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glUseProgram.xhtml){:target="_blank"} and then [`glDrawArrays`](https://registry.khronos.org/OpenGL-Refpages/gl4/html/glDrawArrays.xhtml){:target="_blank"} draws the point on the screen using our program.

It was a lot of work, but we now have a good foundation for building more complicated CG apps. Next time we will look at the basics of rendering 2-dimensional shapes with different colors.