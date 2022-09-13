---
# MARP
theme: default
paginate: true

# Jekyll
title: "Animation and Interactivity"
date: 2022-05-02
categories:
  - Notes
classes: wide
toc_sticky: false
---

*[Working with Uniform Data](#working-with-uniform-data) introduces uniform variables that can be accessed by both the vertex shader and the fragment shader, then uses them to animate a triangle moving across the screen.*  
*[Adding Interactivity](#adding-interactivity) extends the `Input` class to handle various keyboard events and then uses the new features to give the user control of a triangle's movements.*

With vertex array objects, we can associate position and color data to GPU program variables. However, the structure of VAOs means that we cannot share the same data between vertices. Instead, we need to repeat data when we want different vertices to be the same color, for example. This is not ideal. Repeating data multiple times in our source code lowers the **maintainability** and **extensibility** of our software. To change the color of a solid-color hexagon, we would need to change the data for six different vertices.

There must be an easier way! Actually, there is. In situations like this we can use global variables called *uniform variables* that store values in a way that makes them uniformly accessible. That means both shaders use them for every vertex in the array.

# Working with Uniform Data

Uniform variables are useful when we want to send data directly from our application to variables in the GPU program. They do not store data in vertex buffers, and we do not need VAOs to manage their associations. Instead, we just get a reference to the uniform variable inside the GPU program with [`glGetUniformLocation`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glGetUniformLocation.xhtml){:target="_blank"} and then assign a value to it with one of the many [`glUniform`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glUniform.xhtml){:target="_blank"} functions. The `glUniform` function we use will depend on the data type with the number in the function name being the number of values and the letter (either `f` or `i`) being `float` or `int`. For example, sending a single integer (or Boolean value) would require `glUniform1i` but sending a `vec3` would require `glUniform3f`. Then the shader programs will reference that data for every draw.

## The Uniform Class

Similar to how we made the `Attribute` class for attribute variables in the previous post, now we will create a class that handles uniform variables called the `Uniform` class. The responsibilities of this class are to:
- store the data and data type of the uniform variable in the GPU program;
- get and store a reference to the uniform variable that will load the data;
- and load the data into the uniform variable as needed.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, open the file called `openGL.py`.  
<input type="checkbox" class="checkbox inline"> Scroll to the end of `openGL.py` and add the following code after the `Attribute` class:  

```python
class Uniform:
    """Manages a single uniform variable in a shader program."""
    def __init__(self, data_type, data):
        # check the given data type
        valid_types = ('int','bool','float','vec2','vec3','vec4')
        if data_type.lower() not in valid_types:
            raise Exception(f"Unsupported data type: {data_type}")
        self.data_type = data_type.lower()

        # data to be sent to uniform variable
        self.data = data

        # reference for variable location in program
        self.variable_ref = None
```

The `Uniform` class introduces some strict type checking that we didn't have in `Attribute`. It defines the valid data types with a tuple and raises an exception if the given data type is not one of them. This will make it easier to debug your code if you have a typo in your application code or try to use a type that we do not handle in our `upload_data` method later.

<input type="checkbox" class="checkbox inline"> Now add the `locate_variable` method to the `Uniform` class.  

```python
    def locate_variable(self, program_ref, variable_name):
        """Get and store a reference to a uniform variable with the given name."""
        self.variable_ref = GL.glGetUniformLocation(program_ref, variable_name)
        if self.variable_ref == -1:
          raise Exception(f"No uniform variable found for {variable_name}")
```

Here we use the [`glGetUniformLocation`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glGetUniformLocation.xhtml){:target="_blank"} function to get a reference to the uniform variable in the given program. That function will return `-1` if the variable cannot be found. We don't want our program to fail silently if it cannot find the variable, so we raise an exception with a descriptive message for the programmer.

<input type="checkbox" class="checkbox inline"> Next, add the `upload_data` method to the `Uniform` class.  

```python
    def upload_data(self):
        """Store data in a previously located uniform variable."""
        if self.variable_ref == None:
            # the variable has not been located in a program
            raise Exception("Unable to upload data. Must locate uniform variable first.")

        if self.data_type == "int":
            GL.lUniform1i(
                self.variable_ref, 
                self.data
            )
        elif self.data_type == "bool":
            GL.glUniform1i(
                self.variable_ref, 
                self.data
            )
        elif self.data_type == "float":
            GL.glUniform1f(
                self.variable_ref, 
                self.data
            )
        elif self.data_type == "vec2":
            GL.glUniform2f(
                self.variable_ref, 
                self.data[0], 
                self.data[1]
            )
        elif self.data_type == "vec3":
            GL.glUniform3f(
                self.variable_ref, 
                self.data[0], 
                self.data[1], 
                self.data[2]
            )
        elif self.data_type == "vec4":
            GL.glUniform4f(
                self.variable_ref, 
                self.data[0], 
                self.data[1], 
                self.data[2], 
                self.data[3]
            )
```

The first thing we do is confirm that we have a reference to the uniform variable. `self.variable_ref` is set to `None` when the `Uniform` object initializes, but then it receives a value in the `locate_variable` method. So our message to the programmer is a reminder to call `locate_variable` before `upload_data`. Then we check the data type of this uniform variable and upload the data with the appropriate [`glUniform`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glUniform.xhtml){:target="_blank"} function.

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the `openGL.py` file.

Next we will write a simple test program to confirm that the `Uniform` class works as expected.

## A Test of Two Triangles

The application that tests the `Uniform` class will show two triangles with the same shape, but different positions and colors. For this purpose, we will use shader programs that are slightly modified from before. Here is the vertex shader:

```glsl
# GLSL version 330
in vec3 position;
uniform vec3 translation;
void main() {
    vec3 pos = position + translation;
    gl_Position = vec4(pos.x, pos.y, pos.z, 1.0);
}
```
There are a couple of things to notice here. First, the `translation` variable has the `uniform` qualifier which defines it as a uniform variable. We will use our `Uniform` class to upload data to `translation` in the test application.

Second, `gl_Position` (which defines the location of each vertex) is the combination of `position` and `translation`. The `position` variable will reference three vertices centered around the origin that together represent the triangle's shape. The `translation` variable will receive values representing the change in position for each triangle. Combining `position` and `translation` will effectively move the center of the triangle before drawing it. The left triangle will move $-0.5$ units and the right triangle will move $0.5$ units along the $x$-axis.

The new fragment shader program will look like this:

```glsl
# GLSL version 330
uniform vec3 baseColor;
out vec4 fragColor;
void main() {
    fragColor = vec4(baseColor.r, baseColor.g, baseColor.b, 1.0);
}
```

This one is even simpler. Before drawing each triangle, we just upload its color to the `baseColor` uniform variable. The left triangle will be red and the right triangle will be blue.

In order to use the `translation` and `baseColor` uniform variables, our test application will create an instance of the `Uniform` class for each value we want to upload and store a reference to the respective variable. Then it will upload the data in the respective `Uniform` objects before drawing each triangle in the `update` method.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main working folder, create a new file called `test_2_6.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_2_6.py` for editing and add the following code:  

```python
# test_2_6.py
import OpenGL.GL as GL

from core.app import WindowApp
from core.openGL import Attribute, Uniform
from core.openGLUtils import OpenGLUtils

class Test_2_6(WindowApp):
    """Test Uniform by drawing the same triangles with different positions and colors."""
    def startup(self):
        print("Starting up Test 2-6...")

        # vertex shader code with a uniform variable
        vs_code = """
        in vec3 position;
        uniform vec3 translation;
        void main() {
            vec3 pos = position + translation;
            gl_Position = vec4(pos.x, pos.y, pos.z, 1.0);
        }
        """

        # fragment shader code with a uniform variable
        fs_code = """
        uniform vec3 baseColor;
        out vec4 fragColor;
        void main() {
            fragColor = vec4(baseColor.r, baseColor.g, baseColor.b, 1.0);
        }
        """

        self.program_ref = OpenGLUtils.initialize_program(vs_code, fs_code)

        # we only need one vertex array object this time
        vao_ref = GL.glGenVertexArrays(1)
        GL.glBindVertexArray(vao_ref)

        # initialize the triangle vertices relative to the origin (0.0, 0.0, 0.0)
        position_data = (
            ( 0.0,  0.2, 0.0), 
            ( 0.2, -0.2, 0.0), 
            (-0.2, -0.2, 0.0)
        )
        self.vertex_count = len(position_data)
        position_attribute = Attribute("vec3", position_data)
        position_attribute.associate_variable(self.program_ref, "position")

        # initialize a Uniform object for each translation and baseColor value
        self.left_translation = Uniform("vec3", [-0.5, 0.0, 0.0])
        self.left_translation.locate_variable(self.program_ref, "translation")

        self.right_translation = Uniform("vec3", [0.5, 0.0, 0.0])
        self.right_translation.locate_variable(self.program_ref, "translation")

        self.red_color = Uniform("vec3", [1.0, 0.0, 0.0])
        self.red_color.locate_variable(self.program_ref, "baseColor")

        self.blue_color = Uniform("vec3", [0.0, 0.0, 1.0])
        self.blue_color.locate_variable(self.program_ref, "baseColor")

    def update(self):
        GL.glUseProgram(self.program_ref)

        # load and draw the left, red triangle
        self.left_translation.upload_data()
        self.red_color.upload_data()
        GL.glDrawArrays(GL.GL_TRIANGLES, 0, self.vertex_count)

        # load and draw the right, blue triangle
        self.right_translation.upload_data()
        self.blue_color.upload_data()
        GL.glDrawArrays(GL.GL_TRIANGLES, 0, self.vertex_count)

# instantiate and run this test
Test_2_6().run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run the application with the `python test_2_6.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a red triangle on the left side of the screen and a blue triangle on the right side.  

In this application we create a new `Uniform` object and saves it to the application instance `self` for each value that we will want to use with a uniform variable. Then we can just reference the `Uniform` object and use its `upload_data` method when we want to apply its associated value in our `update` method. Note that the data MUST be uploaded BEFORE calling `glDrawArrays` or the shader program won't have access to it.

## Animations

Until now, all of our test applications have used the `update` method to draw still images, which it does 60 times a second. Remember that in the application runtime lifecycle, the `update` method runs in a continuous loop that updates data and renders the image at 60 FPS (see the `run` method in the `WindowApp` class). With static images, we get no real benefit from structuring our application in that way. The benefit comes when the images change in some way over time, which we will now learn how to do by creating animations.

### Clearing the Screen

One of the fundamental techniques in computer animation is the process of clearing the screen in between frames. If we do not clear the screen, then the images drawn in the previous frame will remain on the screen and the new images will draw on top of them. Clearing the screen will effectively erase the previously drawn image and prepare the screen for drawing the current frame. OpenGL provides two functions for this:  
- The [`glClearColor`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glClearColor.xhtml){:target="_blank"} function sets a RGBA color to use when clearing the color buffer. The color provided will effectively become the background color for the new frame.
- The [`glClear`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glClear.xhtml){:target="_blank"} function resets a buffer specified by the parameter. OpenGL constants can be used for the parameters, such as `GL_COLOR_BUFFER_BIT`, `GL_DEPTH_BUFFER_BIT`, `GL_STENCIL_BUFFER_BIT`, or any combination of these using the bitwise OR `|` operator.

Our next test application will create an animation of a single triangle moving to the right across the screen. When it moves off screen completely, it will reposition itself on the left side of the screen and continue moving to the right. This is called a *wrap-around effect*.

The source code is very similary to `test_2_6.py`, except this time we only use the red triangle and completely rewrite the `update` method. In the `update` method, we will increment the `translation` data representing the triangle's $x$-coordinates and clear the color buffer before redrawing the triangle.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main working folder, create a new file called `test_2_7.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_2_7.py` for editing and add the following code:  

```python
# test_2_7.py
import OpenGL.GL as GL

from core.app import WindowApp
from core.openGL import Attribute, Uniform
from core.openGLUtils import OpenGLUtils

class Test_2_7(WindowApp):
    """Test animations with uniform variables by moving a triangle across the screen."""

    def startup(self):
        print("Starting up Test 2-7...")

        # vertex shader code
        vs_code = """
        in vec3 position;
        uniform vec3 translation;
        void main() {
            vec3 pos = position + translation;
            gl_Position = vec4(pos.x, pos.y, pos.z, 1.0);
        }
        """

        # fragment shader code
        fs_code = """
        uniform vec3 baseColor;
        out vec4 fragColor;
        void main() {
            fragColor = vec4(baseColor.r, baseColor.g, baseColor.b, 1.0);
        }
        """

        self.program_ref = OpenGLUtils.initialize_program(vs_code, fs_code)

        # get and bind a vertex array object
        vao_ref = GL.glGenVertexArrays(1)
        GL.glBindVertexArray(vao_ref)

        # the triangle shape as three vertices around the origin
        position_data = (
            ( 0.0,  0.2, 0.0),
            ( 0.2, -0.2, 0.0), 
            (-0.2, -0.2, 0.0)
        )
        self.vertex_count = len(position_data)
        position_attribute = Attribute("vec3", position_data)
        position_attribute.associate_variable(self.program_ref, "position")

        # create initial values for translation and baseColor
        self.translation = Uniform("vec3", [-0.5, 0.0, 0.0])
        self.translation.locate_variable(self.program_ref, "translation")

        self.base_color = Uniform("vec3", [1.0, 0.0, 0.0])
        self.base_color.locate_variable(self.program_ref, "baseColor")

        # specify the color to use for clearing the screen
        glClearColor(0.0, 0.0, 0.0, 1.0)

    def update(self):
        # update translation to move the triangle to the right
        self.translation.data[0] += 0.01

        # if the triangle passes off screen, move it to the left side
        if self.translation.data[0] > 1.2:
            self.translation.data[0] = -1.2

        # reset color buffer with specified color
        GL.glClear(GL.GL_COLOR_BUFFER_BIT)

        GL.glUseProgram(self.program_ref)

        self.translation.upload_data()
        self.base_color.upload_data()
        GL.glDrawArrays(GL.GL_TRIANGLES, 0, self.vertex_count)

# instantiate and run this test
Test_2_7().run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run the application with the `python test_2_7.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a red triangle moving to the right of the screen and then wrapping around to the left side.  

This application only has one `Uniform` object for the `translation` variable and one for the `baseColor` variable. Here we use a list for each `Uniform` object's data because Python tuples are *immutable* (cannot be changed). If we used a tuple instead, we would not be able to change the data values (and would not be able to animate).

In the `update` method, we directly set the new data for the $x$-coordinate translation in the `Uniform` object before uploading its data. If the triangle moves more than `1.2` units to the right, we reposition it to the left side of the screen. Remember, the leftmost point in the triangle position data is at `-0.2` on the $x$-axis, so a translation value of `1.2` will put it at exactly `1.0`, the edge of the screen.

### Keeping Time

Another fundamental technique in computer animation is to calculate movements based on the amount of time that has passed since the previous frame. We will prepare two new instance variables for keeping track of time:
- The `time` variable will count the total number of seconds that the application has been running.
- The `delta_time` variable will store the number of seconds that have passed since the last interation of the run loop.

These variables will go in the `WindowApp` class so they can be inherited into each of our applications. 

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, open the file called `app.py`.  
<input type="checkbox" class="checkbox inline"> Look for the `__init__` method inside the `WindowApp` class and add the following code at the end of `__init__`:  

```python
        # timekeeper variables
        self.__time = 0
        self.__delta_time = 0
```

Here, the double underscores in the variable names will give complete control of the variable access to the `WindowApp` class. That means our applications can create their own `__time` and `__delta_time` variables without overwriting the timekeeper features from `WindowApp`.

<input type="checkbox" class="checkbox inline"> Inside the `WindowApp` class, add two `@property` definitions for the `time` and `delta_time` variables.  

```python
    @property
    def time(self):
        return self.__time

    @property
    def delta_time(self):
        return self.__delta_time
```

This makes the `time` and `delta_time` variables read-only so applications cannot mess around with time.

<input type="checkbox" class="checkbox inline"> Finally, go to the `run` method inside the `WindowApp` class and find the line that calls `self.update()`.  
<input type="checkbox" class="checkbox inline"> Just before the `self.update()` call inside the `while running` loop, add the following code:

```python
            # calculate seconds since the last iteration of the run loop
            self.__delta_time = self.clock.get_time() / 1000
            # increment time application has been running
            self.__time += self.__delta_time
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the `app.py` file.

Now let's test the timekeeper variables with an application that calculates the triangle's position based on time. We could make the triangle move back and forth across the screen with simple linear equations, or we can use sine and cosine functions from trigonometry to make the triangle move more smoothly between $1$ and $-1$ values. Together, sine and cosine can calculate the $(x,y)$ coordinates of a circular path from a time variable $t$:

$$\begin{aligned}
x&=cos(t) \\
y&=sin(t)
\end{aligned}$$

This creates a circle centered at the origin $(0,0)$ with a radius of $1.0$. We can change the radius $r$ and the center $(a,b)$ by altering the equations a little bit:

$$\begin{aligned}
x &= r \cdot cos(t) + a \\
y &= r \cdot sin(t) + b
\end{aligned}$$

If the triangle's path has a radius of $1.0$, then it partially disappear off the screen at the edges. So let's make the radius a little smaller with a value of $0.75$ instead.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Copy your `test_2_7.py` file and change its name to `test_2_8.py`.  
<input type="checkbox" class="checkbox inline"> Inside `test_2_8.py` find the lines with the class definition, document string, and print statement then change them to the following:

```python
class Test_2_8(WindowApp):
    """Test using timekeeper variables to animate a triangle moving in a circle."""
    def startup(self):
        print("Starting up Test 2-8...")
```

<input type="checkbox" class="checkbox inline"> On the last line of the file, change the code that runs the application to `Test_2_8().run()`.  
<input type="checkbox" class="checkbox inline"> At the top of the file, add an `import` statement to get functions for sine and cosine:  

```python
from math import sin, cos
```

<input type="checkbox" class="checkbox inline"> Inside the `update` method, delete the code before `glClear(GL_COLOR_BUFFER_BIT)` and add the following code:  

```python
        # update the (x,y) coordinates of the circular path
        self.translation.data[0] = 0.75 * sin(self.time)
        self.translation.data[1] = 0.75 * cos(self.time)
```

<input type="checkbox" class="checkbox inline"> Save the file and run the application with the `python test_2_8.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a red triangle moving in a circle around the center of the screen.  

This application takes `self.time` as the number of radians in the sine and cosine calculations, so the triangle will complete one full revolution every $2 \pi$ (about 6.28) seconds.

The last test application for animations will use our timekeeper variables to animate changing color. We will copy `test_2_8.py` and use the sine and cosine functions again, but this time we will change the red value of the `baseColor` variable instead of `translation`.

Color values must be in the range of $[0.0,1.0]$ but sine results are in the range $[-1.0,1.0]$, so we need to change our equation again. We can do this by shifting the results out of the negative values with $sin(t) + 1$ so the range becomes $[0.0,2.0]$. Then we just shorten the range of values by half to create the final equation $f(t) = 0.5(sin(t)+1)$.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Copy your `test_2_8.py` file and change its name to `test_2_9.py`.  
<input type="checkbox" class="checkbox inline"> Inside `test_2_9.py` find the lines with the class definition, document string, and print statement then change them to the following:

```python
class Test_2_9(WindowApp):
    """Test using timekeeper variables to animate shifting colors."""
    def startup(self):
        print("Starting up Test 2-9...")
```

<input type="checkbox" class="checkbox inline"> On the last line of the file, change the code that runs the application to `Test_2_9().run()`.  
<input type="checkbox" class="checkbox inline"> Inside the `update` method, delete the code before `glClear(GL_COLOR_BUFFER_BIT)` and add the following code:  

```python
        # update the baseColor red value based on time passed
        self.base_color.data[0] = 0.5 * (sin(self.time) + 1)
```

<input type="checkbox" class="checkbox inline"> Save the file and run the application with the `python test_2_9.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see the triangle fading from red to black and back again.  

The green and blue values of the triangle color are stored in `self.base_color.data[1]` and `self.base_color.data[2]`, respectively. If we wanted to shift through a wider range of colors, we would need to update those values also. But the equations for the green and blue values would need to be different so they peak at different times. In order to visualize this, it may be helpful to see a graph. Desmos.com has a very useful graphing tool and I have prepared one to show oscillating RGB values [here](https://www.desmos.com/calculator/uymwhpwc7x){:target="_blank"}. Try playing with the expressions to see what different effects you can create!

# Adding Interactivity

Up to now, our applications have only allowed one kind of input&mdash;closing the window. If we allow for keyboard inputs, it will greatly expand the varieties of applications we can create. 

## Keyboard Input with Pygame

With our current framework design, we handle input events in the `update` method of the `Input` class. We use Pygame's event features to detect input events and handle quit type events. This section will add keyboard events to be handled by our `Input` class. There are two types of keyboard events that Pygame recognizes:

- **Keydown events** happen when a key is first pressed down.
- **Keyup events** happen when a pressed key is released.

These Pygame events are *discrete*, which means they happen once in an instant of time. However, some user actions are *continuous* which means they happen over a period of time (such as holding down an arrow key or doing click-and-drag actions). In order to handle *continuous* keyboard inputs in addition to *discrete* ones, we will use lists to keep track of a key's state.

Each time we check for events in the `update` method, we will check the keyboard event type and then record the name of the key in lists that represent their state:  

- The `key_down_list` will hold names of keys have just been pressed with a **keydown** event in the current update cycle. This is a *discrete* state, so we will empty the list at the start of each update cycle.
- The `key_pressed_list` will hold names of keys that have been pressed but are not released yet. This is a *continuous* state, so the values will remain in the list until a **keyup** event is received.
- The `key_up_list` will hold names of keys that have just been released with a **keyup** event in the current update cycle. This is a *discrete* state, so we will empty the list at the start of each update cycle.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, open the file called `app.py`.  
<input type="checkbox" class="checkbox inline"> Look for the `__init__` method inside the `Input` class and add the following code at the end of `__init__`:  

```python
        # lists for the key states
        #   Down and Up are discrete states 
        #   Pressed states last continuously between Down and Up events
        self.__key_down_list = []
        self.__key_pressed_list = []
        self.__key_up_list = []
```

Again, we only want the `Input` class to manage these lists, so we name them with double underscores (`__`). But we will give read-only access from outside the class.

<input type="checkbox" class="checkbox inline"> Just before the `update` method in the `Input` class, add properties for each of the key state lists.  

```python
    @property
    def key_down_list(self):
        return self.__key_down_list

    @property
    def key_pressed_list(self):
        return self.__key_pressed_list

    @property
    def key_up_list(self):
        return self.__key_up_list
```

Now, it is common for applications to have a keyboard shortcut for closing the application. Currently, our `quit` property is read-only so we cannot use it to close an application with the `Input` class. Let's add a **setter** property for `quit` to give some control back to the applications.

<input type="checkbox" class="checkbox inline"> Just after the `quit` property inside the `Input` class, add a setter property with the code below.

```python
    @quit.setter
    def quit(self, value):
        self._quit = bool(value)
```

This will assign the Boolean value of the given `value` to `_quit`. This way we can make sure that the values assigned to the `_quit` variable will always be `True` or `False`.

<input type="checkbox" class="checkbox inline"> Find the `update` method inside the `Input` class. Add the following code just before the `for` loop in the `update` method.  

```python
        # reset all the discrete states
        self.__key_down_list = []
        self.__key_up_list = []
```

<input type="checkbox" class="checkbox inline"> Inside the `for` loop of the `update` method, add the following code at the end (after it handles quit events).  

```python
            # handle keyboard events
            #   keydown events initiate the pressed state
            #   keyup events terminate the pressed state
            if event.type == pygame.KEYDOWN:
                key_name = pygame.key.name(event.key)
                self.__key_down_list.append(key_name)
                self.__key_pressed_list.append(key_name)
            if event.type == pygame.KEYUP:
                key_name = pygame.key.name(event.key)
                self.__key_up_list.append(key_name)
                self.__key_pressed_list.remove(key_name)
```

<input type="checkbox" class="checkbox inline"> Finally, add three convenience methods inside the `Input` class for checking the state of a given key.  

```python
    def iskeydown(self, key_code):
        """Checks the down state of the given key"""
        return key_code in self.__key_down_list


    def iskeypressed(self, key_code):
        """Checks the pressed state of the given key"""
        return key_code in self.__key_pressed_list


    def iskeyup(self, key_code):
        """Checks the up state of the given key"""
        return key_code in self.__key_up_list
```

Now let's test our new `Input` features with a small program that outputs messages to the console about key states.

<input type="checkbox" class="checkbox inline"> In your main working folder, create a new file called `test_2_10.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_2_10.py` for editing and add the following code:  

```python
# test_2_10.py
from core.app import WindowApp

class Test_2_10(WindowApp):
    """Test key functionality of the Input class."""
    def startup(self):
        print("Starting up Test 2-10...")

    def update(self):
        if len(self.input.key_down_list) > 0:
            print(f"Keys down: {self.input.key_down_list}")

        if len(self.input.key_pressed_list) > 0:
            print(f"Keys pressed: {self.input.key_pressed_list}")

        if len(self.input.key_up_list) > 0:
            print(f"Keys up: {self.input.key_up_list}")

        # typical use case for key presses
        if self.input.iskeypressed("space"):
            print("The 'space' key is being pressed!!")

        # typical use case for keyboard shortcuts
        if self.input.iskeydown("escape"):
            print("The 'escape' key was pressed--Exiting the application!")
            self.input.quit = True

# initiate and run this test
Test_2_10().run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_2_10.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Try pressing different keys, including **Space** and **Esc**, to see the different event behaviors.  

## Incorporating with Graphics Programs

Now that we have an  `Input` class with working keyboard features, let's use them to control the movements of a triangle on the screen. We will begin with our first test application for animation `test_2_7.py` and change its `update` method to control the triangle's movement with keyboard input.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Copy your `test_2_7.py` file and change its name to `test_2_11.py`.  
<input type="checkbox" class="checkbox inline"> Inside `test_2_11.py` find the lines with the class definition, document string, and print statement then change them to the following:

```python
class Test_2_11(WindowApp):
    """Test interactivity features by using the arrow keys to move a triangle."""
    def startup(self):
        print("Starting up Test 2-11...")
```

<input type="checkbox" class="checkbox inline"> At the end of the `startup` method, add an instance variable for the triangle's speed.  

```python
        # triangle speed in units per second
        self.speed = 0.5
```

The speed is defined in the same units as the triangle and screen coordinates, so it should take 4 seconds to move from one edge of the screen ($-1.0$) to the opposite edge ($1.0$).

<input type="checkbox" class="checkbox inline"> Inside the `update` method, delete the code before `glClear(GL_COLOR_BUFFER_BIT)` and add the following code:  

```python
        # move the triangle based on its speed and the key being pressed
        distance = self.speed * self.delta_time
        # left ← is the negative x direction
        if self.input.iskeypressed("left"):
            self.translation.data[0] -= distance
        # right → is the positive x direction
        if self.input.iskeypressed("right"):
            self.translation.data[0] += distance
        # down ↓ is the negative y direction
        if self.input.iskeypressed("down"):
            self.translation.data[1] -= distance
        # up ↑ is the positive y direction
        if self.input.iskeypressed("up"):
            self.translation.data[1] += distance
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_2_11.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Try each of the arrow keys and confirm that the triangle moves in the correct direction.  

:warning: **NOTE:** Different operating systems might use different strings for they key events. For example, `"left"` might be `"left arrow"` instead. Just run your `test_2_10.py` application anytime you want to check which string is associated with which key.