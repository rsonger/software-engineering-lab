---
# MARP
theme: default
paginate: true

# Jekyll
title: "4_Animation and Interactivity"
date: 2025-04-23
categories:
  - Notes
classes: wide
toc_sticky: false
---

*In this lesson, we introduce uniform data for use in animations and keyboard events for greater interaction with the application.*

In the section [Working with Uniform Data](#working-with-uniform-data), we introduce uniform variables that store shared values for both the vertex shader and the fragment shader, then use them to animate a triangle moving across the screen. In the section [Adding Interactivity](#adding-interactivity), we extend the `Input` class to handle various keyboard events and give the user control over the triangle's movements.  

With vertex array objects, we can associate position and color data to GPU program variables. However, we cannot share the same instances of data between vertices due to the way VAOs are structured. Instead, we need to give each vertex its own data value. This means we repeat data values for vertices that use the same value, such as when we want them to be the same color. This is not ideal. Repeating data multiple times in our source code lowers the **maintainability** and **extensibility** of our software. To change the color of a solid-color hexagon, we would need to change the data for six different vertices.

But there is an easier way! In situations like this we can use a special kind of global variable called a *uniform variable* which stores values in a way that makes them accessible across programs (i.e., *uniform* access). That means both shaders can use the same uniform variable for every vertex in the array.

# Working with Uniform Data

Uniform variables are useful when we want to send data directly from our application to variables in the GPU program. They do not store data in vertex buffers, and we do not need VAOs to manage their associations. Instead, we just use [`glGetUniformLocation`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glGetUniformLocation.xhtml){:target="_blank"} to get a reference to the uniform variable inside the GPU program and then assign a value to it with one of the many [`glUniform`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glUniform.xhtml){:target="_blank"} functions. The `glUniform` function we use will depend on the data type and number of values it holds as indicated in the function name. For example, sending a single integer (or Boolean value) would use `glUniform1i` (where `i` means `int`) but sending a `vec3` would require `glUniform3f` (where `f` means `float`). Then the shader programs will reference that data every time it draws a frame.

## The Uniform Class

Similar to how we made the `Attribute` class for attribute variables in the previous post, we will now create the `Uniform` class to handle uniform variables. The responsibilities of this class are to:  
- store the data and data type of the uniform variable in the GPU program
- get and store a reference to the uniform variable in the GPU program
- load the data into the uniform variable as needed

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `graphics/core` folder, open the file called `openGL.py`.  
<input type="checkbox" class="checkbox inline"> Scroll to the end of `openGL.py` and add the following code at the bottom:  

```python
class Uniform:
    """Manages a single uniform variable in a shader program"""

    _VALID_TYPES = ("int", "bool", "float", "vec2", "vec3", "vec4")  

    def __init__(self, data_type, data):
        # check the given data type
        if data_type.lower() not in self._VALID_TYPES:
            raise ValueError(f"Unsupported data type: {data_type}")
        self.data_type = data_type.lower()

        # data to be sent to uniform variable
        self.data = data

        # reference for the variable's location in the shader
        self.variable_ref = None
```

The `Uniform` class stores the valid data types with a tuple and raises an exception if the given data type is not one of them. This makes it easier to debug your code if you have a typo in your application code or try to use a type that we do not support.  

<input type="checkbox" class="checkbox inline"> Now add the `locate_variable` method to the `Uniform` class.  

```python
    def locate_variable(self, program_ref, variable_name):
        """Get and store a reference to a uniform variable with the given name"""
        self.variable_ref = GL.glGetUniformLocation(program_ref, variable_name)
        if self.variable_ref == -1:
          raise ValueError(f"No uniform variable found with name {variable_name}")
```

Here we use the [`glGetUniformLocation`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glGetUniformLocation.xhtml){:target="_blank"} function to get a reference to the uniform variable in the given program. That function will return `-1` if the variable cannot be found. We don't want our program to fail silently if it cannot find the variable, so we raise an exception with a descriptive message for the programmer.

<input type="checkbox" class="checkbox inline"> Next, add the `upload_data` method to the `Uniform` class.  

```python
    def upload_data(self):
        """Store data in a previously located uniform variable"""
        # check that the variable reference exists
        assert self.variable_ref is not None, (
            "No variable reference; get a reference with locate_variable() before uploading data."
        )

        if self.data_type in {"int", "bool"}:
            GL.glUniform1i(self.variable_ref, self.data)
        elif self.data_type == "float":
            GL.glUniform1f(self.variable_ref, self.data)
        elif self.data_type == "vec2":
            GL.glUniform2f(self.variable_ref, *self.data)
        elif self.data_type == "vec3":
            GL.glUniform3f(self.variable_ref, *self.data)
        elif self.data_type == "vec4":
            GL.glUniform4f(self.variable_ref, *self.data)
```

The first thing we do is confirm that we have a reference to the uniform variable.  When the `Uniform` object initializes, `self.variable_ref` is set to `None` but then it receives a value when the `locate_variable` method is called. So our message to the programmer is a reminder to call `locate_variable` before `upload_data`. Then we check the data type of this uniform variable and upload the data with the appropriate [`glUniform`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glUniform.xhtml){:target="_blank"} function. If the data stores more than one value (as with a `list` or `tuple`), we [unpack the data](https://docs.python.org/3/tutorial/controlflow.html#tut-unpacking-arguments) to pass its values as separate arguments to the GL function.

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the `openGL.py` file.

Next we will write a simple test program to confirm that the `Uniform` class works as expected.

## A Test of Two Triangles

The application that tests the `Uniform` class will show two triangles with the same shape, but different positions and colors. For this purpose, we will use shader programs that are slightly modified from before. The new vertex shader code looks like this:

```glsl
# GLSL version 330
in vec3 position;
uniform vec3 translation;
void main() {
    vec3 pos = position + translation;
    gl_Position = vec4(pos.x, pos.y, pos.z, 1.0);
}
```
There are a couple of things to notice here. First, the `translation` variable has the `uniform` qualifier which declares it to be a uniform variable. We will use our `Uniform` class to upload data to `translation` when we write the the test app.

Second, the value we assign to `gl_Position` (which defines the location of each vertex) combines the values of `position` and `translation`. The `position` variable will reference three vertices centered around the origin that together represent the triangle's shape. The `translation` variable stores values representing the change in position for the triangle. Combining `position` and `translation` will effectively move the center of the triangle before drawing it. We will use this approach so that the left triangle moves $-0.5$ units along the $x$-axis and the right triangle moves $0.5$ units.

The new fragment shader code looks like this:

```glsl
# GLSL version 330
uniform vec3 baseColor;
out vec4 fragColor;
void main() {
    fragColor = vec4(baseColor.r, baseColor.g, baseColor.b, 1.0);
}
```

This one is even simpler. Before drawing each triangle, we just upload its color to the `baseColor` uniform variable. We will make the left triangle red and the right triangle blue.

Our test app will create an instance of the `Uniform` class for each of the `translation` and `baseColor` uniform variables so that we can upload their values and store their references. Setting up each `Uniform` instance happens in the `startup` method. Then, in the `update` method, it will upload the data to use for `translation` and `baseColor` before drawing each triangle.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main folder, create a new file called `test_4_1.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_1.py` and add the following code:  

```python
# test_4_1.py
import OpenGL.GL as GL

from graphics.core.app import WindowApp
from graphics.core.openGL import Attribute, Uniform
from graphics.core.openGLUtils import initialize_program

class Test_4_1(WindowApp):
    """Test Uniform by drawing two separate instances of the same triangle"""
    def startup(self):
        print("Starting up Test 4-1...")

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

        self.program_ref = initialize_program(vs_code, fs_code)

        # we only need one vertex array object this time
        vao_ref = GL.glGenVertexArrays(1)
        GL.glBindVertexArray(vao_ref)

        # initialize the triangle vertices around the origin (0, 0, 0)
        position_data = (
            ( 0.0,  0.2, 0.0), 
            ( 0.2, -0.2, 0.0), 
            (-0.2, -0.2, 0.0)
        )
        self.vertex_count = len(position_data)
        position_attribute = Attribute("vec3", position_data)
        position_attribute.associate_variable(self.program_ref, "position")

        # create a Uniform object for the translation of each triangle
        self.left_translation = Uniform("vec3", [-0.5, 0.0, 0.0])
        self.left_translation.locate_variable(self.program_ref, "translation")

        self.right_translation = Uniform("vec3", [0.5, 0.0, 0.0])
        self.right_translation.locate_variable(self.program_ref, "translation")

        # create a Uniform object for the color of each triangle
        self.red_color = Uniform("vec3", [1.0, 0.0, 0.0])
        self.red_color.locate_variable(self.program_ref, "baseColor")

        self.blue_color = Uniform("vec3", [0.0, 0.0, 1.0])
        self.blue_color.locate_variable(self.program_ref, "baseColor")

        GL.glUseProgram(self.program_ref)

    def update(self):
        # load and draw the left, red triangle
        self.left_translation.upload_data()
        self.red_color.upload_data()
        GL.glDrawArrays(GL.GL_TRIANGLES, 0, self.vertex_count)

        # load and draw the right, blue triangle
        self.right_translation.upload_data()
        self.blue_color.upload_data()
        GL.glDrawArrays(GL.GL_TRIANGLES, 0, self.vertex_count)

# initialize and run this test
Test_4_1().run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run the application with the `python test_4_1.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a red triangle on the left side of the screen and a blue triangle on the right side.  

This app saves a reference to each `Uniform` object associated with a uniform variable in the shader. Then we can reference the `Uniform` object in the `update` method and use its `upload_data` method to assign its value to the associated variable before drawing. Note that the data MUST be uploaded BEFORE calling `glDrawArrays` or the shader program will not use the correct data to draw.

## Animations

Until now, all of our test apps have used the `update` method to draw still images, which it does 60 times a second. Remember that in the application runtime lifecycle, the `update` method runs in a continuous loop that updates data and renders the image at 60 FPS (see the `run` method in the `WindowApp` class). With static images, we get no real benefit from structuring our application in this way. The benefit comes when the image changes over time to create animations. Let's take a look at the essential techniques for animating our rendered images.

### Clearing the Screen

One of the essential techniques in computer animation is to clear the screen in between frames. If we do not clear the screen, then the images drawn in the previous frame will remain on the screen and the new images will draw on top of them. Clearing the screen will effectively erase the previously drawn image and prepare the screen for drawing the new frame. OpenGL provides two functions for this:  
- The [`glClearColor`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glClearColor.xhtml){:target="_blank"} function sets a RGBA color to use when clearing the color buffer. The color provided will effectively become the background color for the new frame.
- The [`glClear`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glClear.xhtml){:target="_blank"} function resets a buffer specified by the parameter. OpenGL constants can be used for the parameters, such as `GL_COLOR_BUFFER_BIT`, `GL_DEPTH_BUFFER_BIT`, `GL_STENCIL_BUFFER_BIT`, or any combination of these using the bitwise OR `|` operator.

Our next test app will create an animation of a single triangle moving to the right across the screen. When it moves off screen completely, it will reposition itself on the left side of the screen and continue moving to the right. This is called a *wrap-around effect*.

The source code is very similar to `test_4_1.py`, except this time we only use the red triangle and completely rewrite the `update` method. In the `update` method, we will increment the `translation` data representing the triangle's $x$-coordinates and clear the color buffer before redrawing the triangle.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main folder, create a new file called `test_4_2.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_2.py` and add the following code:  

```python
# test_4_2.py
import OpenGL.GL as GL

from graphics.core.app import WindowApp
from graphics.core.openGL import Attribute, Uniform
from graphics.core.openGLUtils import initialize_program

class Test_4_2(WindowApp):
    """Test animations with uniform variables by moving a triangle across the screen"""

    def startup(self):
        print("Starting up Test 4-2...")

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

        self.program_ref = initialize_program(vs_code, fs_code)

        # get and bind a vertex array object
        vao_ref = GL.glGenVertexArrays(1)
        GL.glBindVertexArray(vao_ref)

        # the triangle shape is defined around the origin
        position_data = (
            ( 0.0,  0.2, 0.0),
            ( 0.2, -0.2, 0.0), 
            (-0.2, -0.2, 0.0)
        )
        self.vertex_count = len(position_data)
        position_attribute = Attribute("vec3", position_data)
        position_attribute.associate_variable(self.program_ref, "position")

        # the triangle will begin to the left of the origin
        self.translation = Uniform("vec3", [-0.5, 0.0, 0.0])
        self.translation.locate_variable(self.program_ref, "translation")

        self.base_color = Uniform("vec3", [1.0, 0.0, 0.0])
        self.base_color.locate_variable(self.program_ref, "baseColor")

        # specify the color to use for clearing the screen
        GL.glClearColor(0.0, 0.0, 0.0, 1.0)

        GL.glUseProgram(self.program_ref)

    def update(self):
        # update the translation value to move the triangle to the right
        self.translation.data[0] += 0.01

        # if the triangle passes off screen, move it to the left side
        if self.translation.data[0] > 1.2:
            self.translation.data[0] = -1.2

        # reset the color buffer to clear the screen
        GL.glClear(GL.GL_COLOR_BUFFER_BIT)

        # use the updated translation values
        self.translation.upload_data()
        self.base_color.upload_data()
        GL.glDrawArrays(GL.GL_TRIANGLES, 0, self.vertex_count)

# initialize and run this test
Test_4_2().run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run the application with the `python test_4_2.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a red triangle moving to the right of the screen and then wrapping around to the left side.  

Since this app only draws a single triangle, it only needs one `Uniform` object for the `translation` variable and one for the `baseColor` variable. Here we use a list for the data of each `Uniform` object because Python tuples are *immutable* (i.e., they cannot change). If we used a tuple instead, we would not be able to change the data values (and would not be able to animate the triangle).

In the `update` method, we directly set the new data for the $x$-coordinate translation in the `Uniform` object before uploading its data. If the triangle moves more than `1.2` units to the right, we reposition it to the left side of the screen. Remember, the leftmost point in the triangle position data is at `-0.2` on the $x$-axis, so a translation value of `1.2` will put it at exactly `1.0`, the edge of the screen.

### Keeping Time

Another essential technique in computer animation is to calculate movements based on the amount of time that has passed since the previous frame. For this purpose, we will introduce two new instance variables in `WindowApp` for keeping track of time:
- The `time` variable will count the total number of seconds that the application has been running.
- The `delta_time` variable will store the number of seconds that have passed since the last interation of the run loop.

These variables will go in the `WindowApp` class so they can be inherited into each of our applications. 

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, open the file called `app.py`.  
<input type="checkbox" class="checkbox inline"> Look inside the `WindowApp` class and add the following code to the end of the `__init__` method:  

```python
        # timekeeper variables
        self.__time = 0
        self.__delta_time = 0
```

Here, the double underscores in the variable names will give complete control of the variables to the `WindowApp` class. That means our apps could create their own `__time` and `__delta_time` variables without overwriting the timekeeper features in `WindowApp`.

<input type="checkbox" class="checkbox inline"> Inside the `WindowApp` class, add two `@property` definitions for the `time` and `delta_time` variables.  

```python
    @property
    def time(self):
        return self.__time

    @property
    def delta_time(self):
        return self.__delta_time
```

This makes the `time` and `delta_time` variables read-only so apps cannot mess around with time.

<input type="checkbox" class="checkbox inline"> Finally, go to the `run` method inside the `WindowApp` class and find the line that calls `self.update()`.  
<input type="checkbox" class="checkbox inline"> Just before the `self.update()` call inside the `while` loop, add the following code:

```python
            # calculate seconds since the last iteration of the run loop
            self.__delta_time = self.clock.get_time() / 1000
            # increment the total time that the app has been running
            self.__time += self.__delta_time
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the `app.py` file.

Now let's test the timekeeper variables with an application that calculates the triangle's position based on time. We could make the triangle move back and forth across the screen with simple linear equations, or we can use sine and cosine functions to make the triangle move more smoothly between $1$ and $-1$ values. If we assign cosine and sine to the $x$ and $y$ coordinates respectively, we can calculate a circular path from the time variable $t$:

$$\begin{aligned}
x&=cos(t) \\
y&=sin(t)
\end{aligned}$$

This gives coordinates for a circular path centered at the origin $(0,0)$ with a radius of $1.0$. We can change the radius to some value $r$ and make the center some coordinate $(a,b)$ by altering the formulas a little bit:

$$\begin{aligned}
x &= r \cdot cos(t) + a \\
y &= r \cdot sin(t) + b
\end{aligned}$$

If the triangle's path has a radius of $1.0$, then it will partially disappear off screen at the edges. So let's reduce the radius to $0.75$ instead.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Copy your `test_4_2.py` file and change its name to `test_4_3.py`.  
<input type="checkbox" class="checkbox inline"> At the top of the file, change the comment and add an `import` statement to get functions for sine and cosine:  

```python
# test_4_3.py
from math import sin, cos
```

<input type="checkbox" class="checkbox inline"> Find the lines with the class definition, document string, and print statement then change them to the following:

```python
class Test_4_3(WindowApp):
    """Test using timekeeper variables to animate a triangle moving in a circle"""
    def startup(self):
        print("Starting up Test 4-3...")
```

<input type="checkbox" class="checkbox inline"> Inside the `update` method, delete the code before `glClear(GL_COLOR_BUFFER_BIT)` and add the following code:  

```python
        # update the (x,y) position along a circular path
        self.translation.data[0] = 0.75 * sin(self.time)
        self.translation.data[1] = 0.75 * cos(self.time)
```

<input type="checkbox" class="checkbox inline"> On the last line of the file, change the code that runs the application to `Test_4_3().run()`.  
<input type="checkbox" class="checkbox inline"> Save the file and run the application with the `python test_4_3.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a red triangle moving in a circle around the center of the screen.  

This app uses `self.time` as the number of radians for the sine and cosine calculations, so the triangle will complete one full revolution every $2 \pi$ seconds (about 6.28).

The final test app for animations will use our timekeeper variables to animate changing color. We will copy `test_4_3.py` and use the sine and cosine functions again, but this time we will change the value of the `baseColor` shader variable instead of `translation`.

Color values must be in the range of $[0.0,1.0]$ but sine results are in the range $[-1.0,1.0]$, so we need to change our equation again. We can do this by shifting the results out of the negative values with $sin(t) + 1$ so the range becomes $[0.0,2.0]$. Then we just shrink the range of values in half to create the final formula $f(t) = 0.5(sin(t)+1)$.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Copy your `test_4_3.py` file and change its name to `test_4_4.py`.  
<input type="checkbox" class="checkbox inline"> At the top of the file, change the comment to reflect the new file name:  

```python
# test_4_4.py
```

<input type="checkbox" class="checkbox inline"> Find the lines with the class definition, document string, and print statement then change them to the following:

```python
class Test_4_4(WindowApp):
    """Test using timekeeper variables to animate shifting colors"""
    def startup(self):
        print("Starting up Test 4-4...")
```

<input type="checkbox" class="checkbox inline"> Inside the `update` method, delete the code before `glClear(GL_COLOR_BUFFER_BIT)` and add the following code:  

```python
        # update the red value based on time passed
        self.base_color.data[0] = 0.5 * (sin(self.time) + 1)
```

<input type="checkbox" class="checkbox inline"> On the last line of the file, change the code that runs the application to `Test_4_4().run()`.  
<input type="checkbox" class="checkbox inline"> Save the file and run the application with the `python test_4_4.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see the triangle fading from red to black and back to red again.  

The green and blue values of the triangle color are stored in `self.base_color.data[1]` and `self.base_color.data[2]`, respectively. If we wanted to shift through a wider range of colors, we would need to update those values also. But the equations for the green and blue values would need to be different so they peak at different times. In order to visualize this, it may be helpful to see a graph. Desmos.com has a very useful graphing tool and I have prepared one to show oscillating RGB values [here](https://www.desmos.com/calculator/uymwhpwc7x){:target="_blank"}. Try playing with the expressions to see what different effects you can create!

# Adding Interactivity

Up to now, our applications have only allowed one kind of input&mdash;closing the window. If our framework also allows for keyboard inputs, we will be able to create apps with much greater interactivity.  

## Keyboard Input with Pygame

In our current framework design, we handle input events with the `update` method of the `Input` class. We use Pygame to detect input events and process quit type events. Now let's add the ability to handle keyboard events in the `Input` class. There are two types of keyboard events that Pygame recognizes:

- **Keydown events** happen when a key is first pressed down.
- **Keyup events** happen when a pressed key is released.

These Pygame events are *discrete*, which means they happen only in a single moment. However, some user actions are *continuous* which means they happen over a period of time (such as holding down an arrow key or doing click-and-drag actions). In order to handle *continuous* keyboard inputs in addition to *discrete* ones, we will use sets to keep track of each key's state. Sets are useful here because we know each key is unique and the order we store them does not matter.  

When we check for events in the `update` method, we will look at the keyboard event type and then record the name of the key in sets that represent their state:  

- The `down_keys` set will hold names of keys that have just been pressed with a **keydown** event in the current update cycle.  
- The `pressed_keys` set will hold names of keys that have been pressed but are not released yet.  
- The `up_keys` set will hold names of keys that have just been released with a **keyup** event in the current update cycle.  

The **keydown** and **keyup** events represent *discrete* states, so we will clear their sets at the start of each update cycle. The `pressed_keys` set holds keys in a *continuous* state, so their names will remain in the set until their **keyup** event is received.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `graphics/core` folder, open the file called `app.py`.  
<input type="checkbox" class="checkbox inline"> Look for the `__init__` method inside the `Input` class and add the following code at the end of `__init__`:  

```python
        # key states:
        # - "down" and "up" are discrete states 
        # - "pressed" is the continous state between "down" and "up" events
        self.__down_keys = set()
        self.__up_keys = set()
        self.__pressed_keys = set()
```

Again, we want only the `Input` class to manage these sets, so we name them with double underscores (`__`). But we will give read-only access from outside the class.

<input type="checkbox" class="checkbox inline"> Just before the `update` method in the `Input` class, add getter properties for each of the key state sets.  

```python
    @property
    def down_keys(self):
        return self.__down_keys

    @property
    def pressed_keys(self):
        return self.__pressed_keys

    @property
    def up_keys(self):
        return self.__up_keys
```

Now, it is common for applications to have a keyboard shortcut for closing the application. Currently, our `quit` property is read-only so we cannot use it to close an application with the `Input` class. Let's add a **setter** property for `quit` to give some control back to the applications.

<input type="checkbox" class="checkbox inline"> Just after the `quit` property inside the `Input` class, add a setter property with the code below.

```python
    @quit.setter
    def quit(self, value):
        self._quit = bool(value)
```

Here we convert the given `value` to a `Boolean` type and assign it to `_quit`. This way we can make sure that the values assigned to the `_quit` variable will always be `True` or `False`.

<input type="checkbox" class="checkbox inline"> Find the `update` method inside the `Input` class. Add the following code just before the `for` loop in the `update` method.  

```python
        # reset all the discrete states
        self.__down_keys.clear()
        self.__up_keys.clear()
```

<input type="checkbox" class="checkbox inline"> Inside the `for` loop of the `update` method, add the following code at the end (after it handles quit events).  

```python
            # handle keyboard events
            # - keydown events initiate the pressed state
            # - keyup events terminate the pressed state
            if event.type == pygame.KEYDOWN:
                key_name = pygame.key.name(event.key)
                self.__down_keys.add(key_name)
                self.__pressed_keys.add(key_name)
            if event.type == pygame.KEYUP:
                key_name = pygame.key.name(event.key)
                self.__up_keys.add(key_name)
                self.__pressed_keys.remove(key_name)
```

<input type="checkbox" class="checkbox inline"> Finally, add three convenience methods inside the `Input` class for checking the state of a given key.  

```python
    def is_key_down(self, key_code):
        """Checks the down state of the given key"""
        return key_code in self.__down_keys


    def is_key_up(self, key_code):
        """Checks the up state of the given key"""
        return key_code in self.__up_keys


    def is_key_pressed(self, key_code):
        """Checks the pressed state of the given key"""
        return key_code in self.__pressed_keys
```

Now let's test this new feature with a small program that outputs messages to the console about key states.

<input type="checkbox" class="checkbox inline"> In your main working folder, create a new file called `test_4_5.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_5.py` for editing and add the following code:  

```python
# test_4_5.py
from graphics.core.app import WindowApp

class Test_4_5(WindowApp):
    """Test keyboard inputs with the Input class"""
    def startup(self):
        print("Starting up Test 4-5...")

    def update(self):
        if len(self.input.down_keys) > 0:
            print(f"Keys down: {self.input.down_keys}")

        if len(self.input.pressed_keys) > 0:
            print(f"Keys pressed: {self.input.pressed_keys}")

        if len(self.input.up_keys) > 0:
            print(f"Keys up: {self.input.up_keys}")

        # typical use case for key presses
        if self.input.is_key_pressed("space"):
            print("The 'space' key is being pressed!!")

        # typical use case for keyboard shortcuts
        if self.input.is_key_down("escape"):
            print("The 'escape' key was pressed; exiting the application!")
            self.input.quit = True

# initiate and run this test
Test_4_5().run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_5.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Try pressing different keys, including **Space** and **Esc**, to see the different event behaviors.  

## Incorporating with Graphics Programs

Now that we have an  `Input` class with working keyboard features, let's use them to control the movements of a triangle on the screen. We will begin by changing our first test app with animation `test_4_2.py` so that its `update` method allows control of the triangle's movement with keyboard input.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Copy your `test_4_2.py` file and change its name to `test_4_6.py`.  
<input type="checkbox" class="checkbox inline"> At the top of the file, change the comment to reflect the new file name:  

```python
# test_4_6.py
```

<input type="checkbox" class="checkbox inline"> Find the lines with the class definition, docstring, and print statement then change them to the following:

```python
class Test_4_6(WindowApp):
    """Test interactivity features by using the arrow keys to move a triangle"""
    def startup(self):
        print("Starting up Test 4-6...")
```
<input type="checkbox" class="checkbox inline"> At the end of the `startup` method, add an instance variable for the triangle's speed.  

```python
        # triangle speed in units per second
        self.speed = 0.5
```

The speed is defined in the same units as the screen coordinates, so it should take 4 seconds to move from one edge of the screen (at $-1.0$) to the opposite edge (at $1.0$).

<input type="checkbox" class="checkbox inline"> Inside the `update` method, delete the code before `glClear(GL_COLOR_BUFFER_BIT)` and add the following code:  

```python
        # calculate move distance and direction
        distance = self.speed * self.delta_time
        if self.input.is_key_pressed("left"):
            # left ← is the negative x direction
            self.translation.data[0] -= distance
        if self.input.is_key_pressed("right"):
            # right → is the positive x direction
            self.translation.data[0] += distance
        if self.input.is_key_pressed("down"):
            # down ↓ is the negative y direction
            self.translation.data[1] -= distance
        if self.input.is_key_pressed("up"):
            # up ↑ is the positive y direction
            self.translation.data[1] += distance
```

<input type="checkbox" class="checkbox inline"> On the last line of the file, change the code that runs the application to `Test_4_6().run()`.  
<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_6.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Try each of the arrow keys and confirm that the triangle moves in the correct direction.  

:warning: **NOTE:** Different operating systems might use different names for their key events. For example, `"left"` might be `"left arrow"` instead. This `test_4_5.py` app may be a useful tool for you in the future when you want to know which names are associated with different keys.

Now that we have all the basic components for drawing and interacting with animated shapes, we can turn our focus to creating 3D graphics. The next lesson will begin with a review of vector and matrix math concepts, which are essential to 3D CG. Look forward to it!