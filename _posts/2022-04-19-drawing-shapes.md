---
# MARP
theme: default
paginate: true

# Jekyll
title: "Drawing Shapes"
date: 2022-04-19
categories:
  - Notes
classes: wide
toc_sticky: false
---

*Chapter 2.3 adds a class to our framework which makes use of vertex buffers and then shows how to use the class to draw shapes with multiple vertices.*

Our [last application](/software-engineering-lab/notes/windows-points/#rendering-in-the-application) drew a single point using a single vertex. Every vertex shader from this point on will use multiple vertices stored in an array of data called **vertex buffers**. Then we can draw more complicated shapes like triangles (3 vertices), squares (4 vertices), and hexagons (6 vertices).

## Using Vertex Buffers

Recall from [`test_2_2.py`](/software-engineering-lab/notes/windows-points/#rendering-in-the-application) that the **application stage** of the graphics pipeline creates **vertex buffer objects** (VBOs) in memory, stores data in those buffers, and associates vertex buffers with shader variables. These associations are then stored in a bound **vertex array object** (VAO). We bound a single VAO in `test_2_2.py` but did not have any buffers for it. This time we will prepare an `Attribute` class which binds vertex buffers, uploads data to them, and links them to shader variables.

Here is an outline of the class and the OpenGL functions it will use:

- First, we use [`glGenBuffers`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glGenBuffers.xhtml){:target="_blank"} to get a reference to an available buffer for the attribute.
- Then we need to bind the buffer by passing its reference to [`glBindBuffer`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glBindBuffer.xhtml){:target="_blank"}.We also indicate that the the buffer is a vertex buffer by including the `GL_ARRAY_BUFFER` argument.
- Next, we upload the attribute's data to the buffer with [`glBufferData`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glBufferData.xhtml){:target="_blank"}. This function takes a binding target as a parameter, so passing `GL_ARRAY_BUFFER` will tell it to use the currently bound vertex buffer.
- Once the vertex buffer contains data and the GPU program is compiled, we need to get a reference to the attribute's variable in the GPU program so we can associate it with the data. We do this with the [`glGetAttribLocation`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glGetAttribLocation.xhtml){:target="_blank"} function, giving it a reference to the program and the name of the variable. (The variable will be declared with the `in` qualifier inside the vertex shader program.)
- Then we can associate the variable to the bound buffer with [`glVertexAttribPointer`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glVertexAttribPointer.xhtml){:target="_blank"}. This function needs the variable reference, the data type, and the number of components in the data. Basic data types like `int` and `float` have just 1 component, but vectors will have 2, 3, or 4 components.
- Finally, we enable the data to be read through this association by calling the [`glEnableVertexAttribArray`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glEnableVertexAttribArray.xhtml){:target="_blank"} function and giving it the reference to the variable.

## The Attribute Class

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, create a new file called `attribute.py`.  
<input type="checkbox" class="checkbox inline"> Open `attribute.py` for editing and add the following code:  

```python
from OpenGL.GL import *
import numpy

class Attribute(object):
    """Manages a single attribute variable that uses data from a vertex buffer."""
    def __init__(self, dataType, data):
        # data types can be int, float, vec2, vec3, or vec4
        self.dataType = dataType
        self.data = data
        # get a reference to a vertex buffer 
        self.bufferRef = glGenBuffers(1)

        # send the data to the GPU buffer
        self.uploadData()
```

When we create a new vertex attribute, we give it data to store in a vertex buffer and specify the data type. The `Attribute` instance will get an available vertex buffer when it initializes and then immediately upload its data using the `uploadData` method below.

<input type="checkbox" class="checkbox inline"> Add the `uploadData` method to the `Attribute` class.  

```python
    def uploadData(self):
        # convert data to numpy array of 32-bit floating point numbers
        data = numpy.array(self.data).astype(numpy.float32)

        # bind the buffer for use
        glBindBuffer(GL_ARRAY_BUFFER, self.bufferRef)

        # store data in the currently bound buffer as a flat array
        glBufferData(GL_ARRAY_BUFFER, data.ravel(), GL_STATIC_DRAW)
```

By putting this code in a separate `uploadData` method, we can easily update the data multiple times without needing to create a new attribute, which is useful for animations and interactive applications. Before uploading the data, we need to make sure it is in the right format: a one-dimensional array of 32-bit floating point numbers. The `numpy` library is very useful here with its array implementation and `ravel` function.

<input type="checkbox" class="checkbox inline"> After the `uploadData` method, insert the `associateVariable` method below.

```python
    def associateVariable(self, programRef, variableName):
        # get a reference for the program variable with the given name
        variableRef = glGetAttribLocation(programRef, variableName)

        # stop if the program does not use the variable
        if variableRef == -1:
            return

        # select buffer used by the following functions
        glBindBuffer(GL_ARRAY_BUFFER, self.bufferRef)

        # specify how data will be read from the currently bound buffer 
        # into the specified variable. These associations are stored by
        # whichever VAO is bound before calling this method.
        if self.dataType == "int":
            glVertexAttribPointer(variableRef, 1, GL_INT, False, 0, None)
        elif self.dataType == "float":
            glVertexAttribPointer(variableRef, 1, GL_FLOAT, False, 0, None)
        elif self.dataType == "vec2":
            glVertexAttribPointer(variableRef, 2, GL_FLOAT, False, 0, None)
        elif self.dataType == "vec3":
            glVertexAttribPointer(variableRef, 3, GL_FLOAT, False, 0, None)
        elif self.dataType == "vec4":
            glVertexAttribPointer(variableRef, 4, GL_FLOAT, False, 0, None)
        else:
            raise Exception(
                f"Attribute {variableName} has unknown type {self.dataType}"
            )
        
        # enable use of buffer data for this variable during rendering
        glEnableVertexAttribArray(variableRef)
```

Notice that we bind the vertex buffer with `glBindBuffer` before calling `glVertexAttribPointer`. This will ensure that the current VAO will store data from the vertex buffer and associate it with the program variable. OpenGL manages the VAOs and associations, so we do not need to store things like the variable reference in our `Attribute` objects.

## Hexagons, Triangles, and Squares

Now we are ready to draw shapes on the screen with multiple vertices and lines. By default, OpenGL draws lines with only 1 pixel width, which can be hard to see on high resolution displays. We can use a function called `glLineWidth` to set the thickness of the lines drawn by OpenGL in pixels. The function call can go after uploading the shader programs in our `initialize` method.

### A Single Buffer Test
Our first test application will use the `Attribute` class from above to draw lines between six points on the screen to create a hexagon. This time, the vertex shader program has a single variable `position` declared with the `in` qualifier so it will receive data from a vertex buffer. Instead of hardcoding the position data in the vertex buffer, we provide it through our own `positionData` variable and link that data with an instance of `Attribute`.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Make a new file in your main working folder called `test_2_3.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_2_3.py` and add the following test application source code.

```python
from core.base import Base
from core.openGLUtils import OpenGLUtils
from core.attribute import Attribute
from OpenGL.GL import *

class Test_2_3(Base):
    """Test core.Attribute by drawing lines between 6 points in a hexagon."""
    def initialize(self):
        print("Starting up Test 2-3")

        # the vertex shader will receive buffer data for its position variable
        vsCode = """
        in vec3 position;
        void main() {
            gl_Position = vec4(position.x, position.y, position.z, 1.0);
        }
        """

        # fragment shader code
        fsCode = """
        out vec4 fragColor;
        void main() {
            fragColor = vec4(1.0, 1.0, 0.0, 1.0);
        }
        """

        self.programRef = OpenGLUtils.initializeProgram(vsCode, fsCode)

        # set a wider line width so it is easier to see
        glLineWidth(4)

        # create and bind the vertex array object (VAO)
        vaoRef = glGenVertexArrays(1)
        glBindVertexArray(vaoRef)

        # initialize the vertex attribute data
        positionData = [
            [0.8, 0.0, 0.0], 
            [0.4, 0.6, 0.0], 
            [-0.4, 0.6, 0.0],
            [-0.8, 0.0, 0.0],
            [-0.4, -0.6, 0.0],
            [0.4, -0.6, 0.0]
        ]
        # set the number of vertices to be used in the draw function
        self.vertexCount = len(positionData)
        # create and link an attribute for the position variable
        positionAttribute = Attribute("vec3", positionData)
        positionAttribute.associateVariable(self.programRef, "position")

    def update(self):
        glUseProgram(self.programRef)
        # use the line loop drawing mode to connect all the vertices
        glDrawArrays(GL_LINE_LOOP, 0, self.vertexCount)

# instantiate and run this test
Test_2_3().run()
```
<input type="checkbox" class="checkbox inline"> Run the application with the `python test_2_3.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that a yellow hexagon outline appears on your screen.  

Notice that we create an `Attribute` instance with our `positionData` and and then link it to the vertex shader's `position` variable with the `associateVariable` method.

This time, the [`glDrawArrays`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glDrawArrays.xhtml) function uses the `GL_LINE_LOOP` mode to draw lines from one vertex to the next and then connect the last vertex to the first one. Here we use `vertexCount` from the `initialize` method to tell exactly how many vertices it should draw.

Now `glDrawArrays` can use a number of different [OpenGL primitives](https://www.khronos.org/opengl/wiki/Primitive) to render the lines in different ways. In `test_2_2.py`, we used `GL_POINTS` to render a single point. `GL_LINES` will draw a line between each pair of points, and `GL_LINE_STRIP` will connect each point to the next, stopping at the last point.

![OpenGL primitives for point and line drawing modes](/software-engineering-lab/assets/images/point-and-line-primitives.png)

When we want to fill in the area between lines, we can use one of the triangle drawing modes. `GL_TRIANGLES` will connect every three points without any overlap. `GL_TRIANGLE_STRIP` will include the last two points of the previous triangle with the next point to create an adjacent triangle. Finally, `GL_TRIANGLE_FAN` connects every two points with the first point to create a fan-like array of adjacent triangles all sharing the same point.

![OpenGL primitives for triangle drawing modes](/software-engineering-lab/assets/images/triangle-primitives.png)

It is also possible to combine draw modes with the same set of vertices by simply calling `glDrawArrays` multiple times. For example, if the `update` method had the next two lines of code, it would draw the lines of the hexagon and each of the points as well. Although if you do this, don't forget to increase the point size with `glPointSize` so the points are easy to see!

```python
        glDrawArrays(GL_LINE_LOOP, 0, self.vertexCount)
        glDrawArrays(GL_POINTS, 0, self.vertexCount)
```

### A Multi-Buffer Test

The `test_2_3.py` test application only uses a single buffer with a single set of vertices for a drawing a single shape. Drawing more than one shape will require more than one vertex buffer as demonstrated in the next test application which draws a triangle and a square at the same time. 

Even though the position data for the triangle and square will be stored in separate buffers, we will use the same vertex shader and fragment shader code to draw both shapes. We can do this because the rendering process for a triangle is essentially the same as the rendering process for a square. The only difference is the postion data. In order to use the same programs with the same `position` variable, we will make and store references to two different vertex arrays. Since VAOs store associations between buffers and variables, one VAO will associate the `position` variable to the triangle's position data in the buffer while the other VAO will associate `position` with the square's position data in the buffer. Then, in the `update` method, we use the stored VAO references to bind the associated VAO before calling `glDrawArrays`.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main folder, create a new file called `test_2_4.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_2_4.py` and add the following code.  

```python
from core.base import Base
from core.openGLUtils import OpenGLUtils
from core.attribute import Attribute
from OpenGL.GL import *

class Test_2_4(Base):
    """Test multiple VAOs by rendering a square and a triangle together."""
    def initialize(self):
        print("Starting up Test 2-4")

        vsCode = """
        in vec3 position;
        void main() {
            gl_Position = vec4(position.x, position.y, position.z, 1.0);
        }
        """

        fsCode = """
        out vec4 fragColor;
        void main() {
            fragColor = vec4(1.0, 1.0, 0.0, 1.0);
        }
        """

        self.programRef = OpenGLUtils.initializeProgram(vsCode, fsCode)

        # get a reference to a vertex array object for the triangle
        self.vaoTriangle = glGenVertexArrays(1)
        glBindVertexArray(self.vaoTriangle)
        posDataTriangle = [
            [-0.5, 0.8, 0.0],
            [-0.2, 0.2, 0.0],
            [-0.8, 0.2, 0.0]
        ]
        self.vertexCountTriangle = len(posDataTriangle)
        posAttributeTriangle = Attribute("vec3", posDataTriangle)
        posAttributeTriangle.associateVariable(self.programRef, "position")

        # get a reference to a vertex array object for the square
        self.vaoSquare = glGenVertexArrays(1)
        glBindVertexArray(self.vaoSquare)
        posDataSquare = [
            [0.8, 0.8, 0.0],
            [0.8, 0.2, 0.0],
            [0.2, 0.2, 0.0],
            [0.2, 0.8, 0.0]
        ]
        self.vertexCountSquare = len(posDataSquare)
        posAttributeSquare = Attribute("vec3", posDataSquare)
        posAttributeSquare.associateVariable(self.programRef, "position")

    def update(self):
        # the same program renders both shapes
        glUseProgram(self.programRef)

        # draw the triangle
        glBindVertexArray(self.vaoTriangle)
        glDrawArrays(GL_LINE_LOOP, 0, self.vertexCountTriangle)

        # draw the square
        glBindVertexArray(self.vaoSquare)
        glDrawArrays(GL_LINE_LOOP, 0, self.vertexCountSquare)

# instantiate this test and run it
Test_2_4().run()
```

<input type="checkbox" class="checkbox inline"> Run the application with the `python test_2_4.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that a yellow triangle outline and a yellow square outline appear on your screen.  

## Passing Data Between Shaders

We can also use vertex buffers to hold color data in addition to position data. This requires a few extra steps since buffer data is first passed into the vertex shader, but color data is only used in the fragment shader. So we need to pass the color data from the vertex shader to the fragment shader.

Remember, in the [OpenGL Shading Language (GLSL)](/software-engineering-lab/notes/windows-points/#the-opengl-shading-language-glsl), variables have *type qualifiers*. In the vertex shader, `in` means the variable data comes from a vertex buffer and `out` means the data will go to the fragment shader. In the fragment shader, `in` means the data comes from the vertex shader and `out` means the data will be stored in another memory buffer.

In order to send color data from our application to be rendered by the fragment shader, we need to send it through a variable in the vertex shader. So the vertex shader will declare a `vertexColor` variable with `in` and a `color` variable with `out`. Then, the fragment shader must also declare a variable with `in` using the exact same name and type as the `out` variable from the vertex shader.

![Data flow between shaders and buffers](/software-engineering-lab/assets/images/buffer-shader-dataflow.png)

Here is what our vertex shader will look like with the new variables:

```glsl
in vec3 position;
in vec3 vertexColor;
out vec3 color;
void main() {
    gl_Position = vec4(position.x, position.y, position.z, 1.0);
    color = vertexColor;
}
```

Note that the vertex shader does not change the color data in any way. It only assigns the values from the `in` variable to the `out` variable with `color = vertexColor;`.

Then, the fragment shader will look like this:

```glsl
in vec3 color;
out vec4 fragColor;
void main() {
    fragColor = vec4(color.r, color.g, color.b, 1.0);
}
```

The color data from the vertex shader is received with the `in` variable by the same name `color`.

(**NOTE**: The vertex shader uses [x, y, z] naming to access components while the fragment shader uses [r, g, b] naming. They both similarly access the components at index [0, 1, 2] respectively.)

Now let's create one more test application to demonstrate color data passing through atttribute variables alongside position data.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main folder, create a new file called `test_2_5.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_2_5.py` and add the following source code:

```python
from core.base import Base
from core.openGLUtils import OpenGLUtils
from core.attribute import Attribute
from OpenGL.GL import *

class Test_2_5(Base):
    """Test passing color data between shaders with a colorful hexagon."""
    def initialize(self):
        print("Starting up Test 2-5")

        vsCode = """
        in vec3 position;
        in vec3 vertexColor;
        out vec3 color;
        void main() {
            gl_Position = vec4(position.x, position.y, position.z, 1.0);
            color = vertexColor;
        }
        """

        fsCode = """
        in vec3 color;
        out vec4 fragColor;
        void main() {
            fragColor = vec4(color.r, color.g, color.b, 1.0);
        }
        """

        self.programRef = OpenGLUtils.initializeProgram(vsCode, fsCode)

        # make points larger so they are easy to see
        glPointSize(10)

        # create and bind a single VAO
        vaoRef = glGenVertexArrays(1)
        glBindVertexArray(vaoRef)

        # position data for each of the six points
        positionData = [
            [0.8, 0.0, 0.0], 
            [0.4, 0.6, 0.0], 
            [-0.4, 0.6, 0.0],
            [-0.8, 0.0, 0.0],
            [-0.4, -0.6, 0.0],
            [0.4, -0.6, 0.0]
        ]
        positionAttribute = Attribute("vec3", positionData)
        positionAttribute.associateVariable(self.programRef, "position")

        # color data for each of the six points
        colorData = [
            [0.5, 0.0, 0.0],
            [1.0, 0.5, 0.0],
            [1.0, 1.0, 0.0],
            [0.0, 1.0, 0.0],
            [0.0, 0.0, 1.0],
            [0.5, 0.0, 1.0]
        ]
        colorAttribute = Attribute("vec3", colorData)
        colorAttribute.associateVariable(self.programRef, "vertexColor")

        # both position and color VBOs have the same number of vertices
        self.vertexCount = len(positionData)

    def update(self):
        glUseProgram(self.programRef)
        glDrawArrays(GL_POINTS, 0, self.vertexCount)

# instantiate and run this test
Test_2_5().run()
```

<input type="checkbox" class="checkbox inline"> Run the application with the `python test_2_4.py` command in your terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see six different colored dots on your screen.  

In `test_2_4.py`, we used two different vertex arrays to bind different buffers to the same program variable. This time we have two different program variables associated with two different buffers, so we can use a single VAO to store all the associations.

Now what happens if we change the draw mode from `GL_POINTS` to something like `GL_LINE_LOOP` or `GL_TRIANGLE_FAN`? In that case, we can see OpenGL's **rasterization** process in action as it *interpolates* the color values in between each vertex. Here, interpolation is a mathematical calculation of the RGB components for each pixel based on how far it is from the original vertices. It weighs the values of each component differently and combines them to get each pixel's final color.

For example, if a point $P$ is halfway between a point with color $C_1=[1.0, 0.0, 0.0]$ (red) and a second point with color $C_2=[0.0, 0.0, 1.0]$ (green), then its color $C_P$ would be half of $C_1$ and half of $C_2$, which is purple:

$$\begin{aligned}
C_P &=0.5 \cdot C_1+0.5 \cdot C_2 \\
    &=0.5 \cdot [1.0, 0.0, 0.0] + 0.5 \cdot [0.0, 0.0, 1.0] \\
    &=[0.5, 0.0, 0.0] + [0.0, 0.0, 0.5] \\
    &=[0.5, 0.0, 0.5]
\end{aligned}$$

The result of filling a shape with triangle draw modes will interpolate the colors between vertices, effectively creating a gradient effect. Now if we want all the points and the shape to be filled with the same color, we just need to change all the vertices of our `colorData` variable to store the same values. 

Next time, we create a new class that makes it easy to create solid color shapes. This same class will also allow us to create animations and interactivity easily as well. Look forward to it!