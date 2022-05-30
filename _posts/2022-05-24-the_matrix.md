---
# MARP
theme: default
paginate: true

# Jekyll
title: "Enter the Matrix"
date: 2022-05-24
categories:
  - Notes
classes: wide
toc_sticky: false
---

*Chapters 3.3 and 3.4 apply what we learned about calculating geometric transformations to build a `Matrix` class and integrate it with our CG framework.*

Now that we have learned about the usefulness of [matrix calculations](/software-engineering-lab/notes/ch3-1/) for [geometric transformations](/software-engineering-lab/notes/ch3-2/), we will create a new class in our CG framework that can create transformation matrices for us. Our framework will always use 4x4 matrices so we can easily handle 2D and 3D graphics. When rendering in 2D, we can simply use a value of $0.0$ for all the $z$ components so everything renders on the same plane.

# The `Matrix` Class

Our `Matrix` class will have static methods that return different kinds of transformation matrices from the given parameters. With static methods, we do not need to create any instances of `Matrix` and we do not need to manage any state variables either. Simply calling these methods will give us the matrix data we need to do geometric transformations.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, create a new file called `matrix.py`.  
<input type="checkbox" class="checkbox inline"> Open `matrix.py` for editing and add the following code:  

```python
import numpy
from math import sin, cos, tan, pi
```

We will use 2D [NumPy arrays](https://numpy.org/doc/stable/reference/generated/numpy.array.html){:target="_blank"} to represent our matrices. This gives us an advantage over memory, processing time, and convenience for matrix multiplication. NumPy lets us use the `@` operator to easily multiply matrices that are created as NumPy arrays. We also import the `math` functions for sine, cosine and tangent to use when calculating our matrices. The constant pi will also help us convert the angle of view to radians.

<input type="checkbox" class="checkbox inline"> Add the next code to `matrix.py` for creating the class along with a method for returning the identity matrix.  

```python
class Matrix(object):
    """Static methods for generating four-dimensional transformation matrices."""

    @staticmethod
    def makeIdentity():
        """Return the 4D identity matrix."""
        return numpy.array((
            (1, 0, 0, 0),
            (0, 1, 0, 0),
            (0, 0, 1, 0),
            (0, 0, 0, 1)
        )).astype(float)
```

The `@staticmethod` decorator defines the `makeIdentity()` method as a static method. This means it does not use an instance of `Matrix` (so it cannot access `self`) and we can call it directly on the name of the class itself (for example, `id_matrix = Matrix.makeIdentity()`)

The Python interpreter ignores whitespace inside the contents of parentheses (). We can take advantage of this to write the contents of our 2D NumPy array to appear as a 4x4 matrix. Additionally, all the values in a NumPy array must be the same type. So we will create each of our matrices with float values by calling the `astype()` method on the newly created array.

<input type="checkbox" class="checkbox inline"> Add the next code to `matrix.py` for creating a translation matrix.  

```python
    @staticmethod
    def makeTranslation(x, y, z):
        """Return a 4D matrix for the translation vector <x,y,z>."""
        return numpy.array((
            (1, 0, 0, x),
            (0, 1, 0, y),
            (0, 0, 1, z),
            (0, 0, 0, 1)
        )).astype(float)
```

Here the parameters `x`, `y`, and `z` are scalar values for their respective coordinates in the translation. The method creates an identity matrix for the scaling and rotation components of the transformation matrix then sets the translation coordinates to the values of the parameters.

<input type="checkbox" class="checkbox inline"> Next, add the following methods to `matrix.py` that create matrices for rotation around each of the 3 axes, $x$, $y$, and $z$.  

```python
    @staticmethod
    def makeRotationX(angle):
        """Return a 4D matrix for rotating around the x-axis by the given angle in radians."""
        c = cos(angle)
        s = sin(angle)
        return numpy.array((
            (1, 0,  0, 0),
            (0, c, -s, 0),
            (0, s,  c, 0),
            (0, 0,  0, 1)
        )).astype(float)

    @staticmethod
    def makeRotationY(angle):
        """Return a 4D matrix for rotating around the y-axis by the given angle in radians."""
        c = cos(angle)
        s = sin(angle)
        return numpy.array((
            ( c, 0, s, 0),
            ( 0, 1, 0, 0),
            (-s, 0, c, 0),
            ( 0, 0, 0, 1)
        )).astype(float)

    @staticmethod
    def makeRotationZ(angle):
        """Return a 4D matrix for rotating around the z-axis by the given angle in radians."""
        c = cos(angle)
        s = sin(angle)
        return numpy.array((
            (c, -s, 0, 0),
            (s,  c, 0, 0),
            (0,  0, 1, 0),
            (0,  0, 0, 1)
        )).astype(float)
```

These methods all take the angle of rotation in radians. Then we can simply calculate the cosine and sine values before constructing a matrix with the appropriate values.

<input type="checkbox" class="checkbox inline"> Add the `makeScale` method to `matrix.py` for scaling transformations.  

```python
    @staticmethod
    def makeScale(r, s, t):
        """Return a 4D matrix for scaling by the given magnitudes."""
        return numpy.array((
            (r, 0, 0, 0),
            (0, s, 0, 0),
            (0, 0, t, 0),
            (0, 0, 0, 1)
        )).astype(float)
```

Scaling can happen on any dimension, so we want to allow for scaling each dimension individually. **Uniform scaling** happens when all dimensions scale equally. In that case, we just use the same value for `r`, `s`, and `t`.

<input type="checkbox" class="checkbox inline"> Finally, add the `makePerspective` method to `matrix.py` for calculating the perspective projection matrix.  

```python
    @staticmethod
    def makePerspective(angleOfView=60, aspectRatio=1, near=0.1, far=1000):
        """Return a 4D matrix for a projection with the given perspective."""
        a = angleOfView * pi / 180.0
        d = 1.0 / tan(a/2)
        r = aspectRatio
        b = (near + far) / (near - far)
        c = 2 * near * far / (near - far)
        return numpy.array((
            (d/r, 0,  0, 0),
            (  0, d,  0, 0),
            (  0, 0,  b, c),
            (  0, 0, -1, 0)
        )).astype(float)
```

This method definition provides values for a default perspective so we do not need to enter them in every application. The angle of view parameter uses degrees for its units, so we need to convert it into radians before applying it to the tangent function for calculating the distance between the projection window and the camera. The depth components `b` and `c` are calculated from the *near clipping distance* and *far clipping distance* as explained in the previous chapter.

Before we can use the `Matrix` class, we need to update our `Uniform` class so it can handle 4x4 matrix data for uniform variables in our shader programs.

# Updating the `Uniform` Class

Remember that our `Uniform` class manages a link between a vertex buffer and a `uniform` variable in a shader program. The class uses [`glUniform`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glUniform.xhtml){:target="_blank"} functions to assign data based on its data type. Now that we have matrix data to use in our applications, we need to update the `Uniform` class so it can associate matrix data with variables as well.

GLSL uses the `mat4` data type for 4x4 matrices and we can use the `glUniformMatrix4fv` function to upload data for `mat4` shader variables.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open the `uniform.py` file from your `core` folder.  
<input type="checkbox" class="checkbox inline"> Find the long `if` statement inside the `uploadData` method and add the following code at the end:  

```python
        elif self.dataType == "mat4":
            glUniformMatrix4fv(self.variableRef, 1, GL_TRUE, self.data)
```

The second parameter is the number of matrices, which will always be `1` for our `Uniform` objects. The third parameter tells OpenGL that our matrix data is stored as an array of *row* vectors. If we ever give the data as an array of *column* vectors (we won't), then that parameter should be `GL_FALSE`.

# A Test of Transformations

Now we are ready to build an application that uses our `Matrix` class for geometric transformations. The test app will be a 2D triangle that moves and rotates according to user input. On the left side of the keyboard, the WASD keys will control global translation while Q and E control global rotation. On the right side, the IJKL keys will control local translation while U and O control local rotation.

The shader program we use will contain two `uniform mat4` variables&mdash;one for the projection matrix and one for the model matrix. Multiplying both of these matrices in order with the position vector will give us the object's position on the projection window. Our new vector shader program looks like this:

```glsl
in vec3 position;
uniform mat4 projectionMatrix;
uniform mat4 modelMatrix;
void main() {
    gl_Position = projectionMatrix * modelMatrix * vec4(position, 1.0);
}
```

Remember that the position vector stays the same and all of the transformations will apply to the `modelMatrix`. Then, the `projectionMatrix` will adjust the positions based on the perspective, which makes shapes look smaller as they move away from the camera.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main working folder, create a new file called `test_3.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_3.py` for editing and add the following code:  

```python
from core.base import Base
from core.openGLUtils import OpenGLUtils
from core.attribute import Attribute
from core.uniform import Uniform
from core.matrix import Matrix

from OpenGL.GL import *
from math import pi

class Test_3(Base):
    """Tests geometric transformations by moving a triangle around the screen."""

    def initialize(self):
        print("Starting up Test 3")

        ## Initialize program
        vsCode = """
        in vec3 position;
        uniform mat4 projectionMatrix;
        uniform mat4 modelMatrix;
        void main() {
            gl_Position = projectionMatrix * modelMatrix * vec4(position, 1.0);
        }
        """

        fsCode = """
        out vec4 fragColor;
        void main() {
            fragColor = vec4(1.0, 1.0, 0.0, 1.0);
        }
        """

        self.programRef = OpenGLUtils.initializeProgram(vsCode, fsCode)
```

Next, we create an `Attribute` object for the position data and two `Uniform` objects for the two uniform matrices in the shader program. We will use `makeTranslation` and `makeRotationZ` to update the model matrix whenever the user moves or rotates the triangle. The `makePerspective` method is only called once in our `initialize` method since the perspective will always stay the same.

<input type="checkbox" class="checkbox inline"> Inside the `initialize` method of `test_3.py`, add the following code:  

```python
        # one VAO for the single triangle
        vaoRef = glGenVertexArrays(1)
        glBindVertexArray(vaoRef)

        # triangle point data
        positionData = ( 
            ( 0.0,  0.3, 0.0 ),
            ( 0.2, -0.3, 0.0 ),
            (-0.2, -0.3, 0.0 )
        )
        self.vertexCount = len(positionData)
        positionAttribute = Attribute("vec3", positionData)
        positionAttribute.associateVariable(self.programRef, "position")

        # make model and perspective matrices as uniform objects
        mMatrix = Matrix.makeTranslation(0, 0, -5)
        self.modelMatrix = Uniform("mat4", mMatrix)
        self.modelMatrix.locateVariable(self.programRef, "modelMatrix")

        pMatrix = Matrix.makePerspective()
        self.projectionMatrix = Uniform("mat4", pMatrix)
        self.projectionMatrix.locateVariable(self.programRef, "projectionMatrix")

        # movement speed in units per second
        self.moveSpeed = 1.0
        # rotation speed in radians per second
        self.turnSpeed = pi / 2

        # render settings
        glClearColor(0.0, 0.0, 0.0, 1.0)
        glEnable(GL_DEPTH_TEST)
```

Our triangle will be taller than it is wide so that we can see its orientation as it rotates around the screen. When we create the model matrix, we make it from a translation matrix that shifts the triangle backwards down the $z$-axis. Since the camera is located at the origin, we would not be able to see the triangle if it was also on the $z=0$ plane, so we move it to $z=-5$ so it appears in front of the camera.

The movement speed is set to units in world space. This means that the greater the distance between the object and the camera, the slower it appears to move. On the other hand, the rotation speed is unaffected by the object's distance from the camera, but it appears to speed up as the object moves away from the $z$-axis. This will be clear when we compare local rotation (with the $z$-axis at the center of the triangle) to global rotation (with the $z$-axis at the center of the screen).

The last line of the `initialize` method is a function that enables OpenGL's depth testing feature. This app renders a 3D scene, so depth testing tells OpenGL to calculate whether objects in the scene will block each other from view. There is only one object in the scene for now, but we turn it on in case we want to add more objects later.

<input type="checkbox" class="checkbox inline"> Now add the `update` method to the `Test_3` class with the following code:  

```python
    def update(self):
        # first, update changes in position
        moveAmount = self.moveSpeed * self.deltaTime
        turnAmount = self.turnSpeed * self.deltaTime
```

As we did in the [Animations](/software-engineering-lab/notes/animations/#animations) chapter, we first calculate distances based on the time that has passed between frames. This time we also have a rotation distance to calculate separately from the move distance.

<input type="checkbox" class="checkbox inline"> Next, add code for global translations to the `update` method.  

```python
        # global translation
        # "w" is upward movement in the positive y direction
        if self.input.isKeyPressed("w"):
            m = Matrix.makeTranslation(0, moveAmount, 0)
            self.modelMatrix.data = m @ self.modelMatrix.data
        # "a" is leftward movement in the negative x direction
        if self.input.isKeyPressed("a"):
            m = Matrix.makeTranslation(-moveAmount, 0, 0)
            self.modelMatrix.data = m @ self.modelMatrix.data
        # "s" is downward movement in the negative y direction
        if self.input.isKeyPressed("s"):
            m = Matrix.makeTranslation(0, -moveAmount, 0)
            self.modelMatrix.data = m @ self.modelMatrix.data
        # "d" is rightward movement in the positive x direction
        if self.input.isKeyPressed("d"):
            m = Matrix.makeTranslation(moveAmount, 0, 0)
            self.modelMatrix.data = m @ self.modelMatrix.data
```

Similar to our [Test 2-11](/software-engineering-lab/notes/animations/#incorporating-with-graphics-programs) application, we move the triangle in the direction specified by the key press. Here, we make a translation matrix for the movement and then multiply it by the existing model matrix to get a new model matrix. We are using the `@` operator since both of the matrices are NumPy arrays.

<input type="checkbox" class="checkbox inline"> Now add code for global rotations and local translations to the `update` method.  

```python
        # global rotation
        # "q" is counterclockwise rotation around the global z-axis
        if self.input.isKeyPressed("q"):
            m = Matrix.makeRotationZ(turnAmount)
            self.modelMatrix.data = m @ self.modelMatrix.data
        # "e" is clockwise rotation around the global z-axis
        if self.input.isKeyPressed("e"):
            m = Matrix.makeRotationZ(-turnAmount)
            self.modelMatrix.data = m @ self.modelMatrix.data
```

This code is similar to global translations except we use `Matrix.makeRotationZ` to get a rotation matrix instead. The Q key rotates in the positive (counterclockwise) direction and E rotates in the negative (clockwise) direction.

<input type="checkbox" class="checkbox inline"> After that, add code for local translations to the `update` method.  

```python
        # local translation
        # "i" is movement in the triangle's positive y direction
        if self.input.isKeyPressed("i"):
            m = Matrix.makeTranslation(0, moveAmount, 0)
            self.modelMatrix.data = self.modelMatrix.data @ m
        # "j" is movement in the triangle's negative x direction
        if self.input.isKeyPressed("j"):
            m = Matrix.makeTranslation(-moveAmount, 0, 0)
            self.modelMatrix.data = self.modelMatrix.data @ m
        # "k" is movement in the triangle's negative y direction
        if self.input.isKeyPressed("k"):
            m = Matrix.makeTranslation(0, -moveAmount, 0)
            self.modelMatrix.data = self.modelMatrix.data @ m
        # "l" is movement in the triangle's positive x direction
        if self.input.isKeyPressed("l"):
            m = Matrix.makeTranslation(moveAmount, 0, 0)
            self.modelMatrix.data = self.modelMatrix.data @ m
```

Here we have changed from global coordinates to local coordinates. As discussed in the [Local Transformations](/software-engineering-lab/notes/ch3-2/#local-transformations) chapter, the only difference is the order in which we compose our model matrices. Local transformations are the reverse of global transformations so we apply the model matrix after the translation matrix.

<input type="checkbox" class="checkbox inline"> Now add the final code for local rotations to the `update` method.  

```python
        # local rotation
        # "u" is counterclockwise rotation around the triangle's center
        if self.input.isKeyPressed("u"):
            m = Matrix.makeRotationZ(turnAmount)
            self.modelMatrix.data = self.modelMatrix.data @ m
        # "o" is clockwise rotation around the triangle's center
        if self.input.isKeyPressed("o"):
            m = Matrix.makeRotationZ(-turnAmount)
            self.modelMatrix.data = self.modelMatrix.data @ m
```

As local transformations, these rotations will apply to the object's local coordinate axes, so the triangle will always spin in place around its own center with the U and O keys.

<input type="checkbox" class="checkbox inline"> Finally, add the code for clearing and drawing the scene with our matrix data at the end of the `update` method.  

```python
        # reset the color buffer and the depth buffer
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)

        glUseProgram(self.programRef)
        
        self.projectionMatrix.uploadData()
        self.modelMatrix.uploadData()
        glDrawArrays(GL_TRIANGLES, 0, self.vertexCount)

# instantiate and run this test
Test_3().run()
```

Since we enabled depth testing, we want to reset the depth buffer every frame like we do with the color buffer. Then we can specify our shader program, upload the matrix data, and use it to draw the triangle. Don't forget to also include the last line for running this test application!

<input type="checkbox" class="checkbox inline"> Save the file and run it with the `python test_3.py` command in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a yellow triangle on the screen.  
<input type="checkbox" class="checkbox inline"> Test that the WASD keys move the triangle globally.  
<input type="checkbox" class="checkbox inline"> Test that the Q and E keys rotate the triangle globally.  
<input type="checkbox" class="checkbox inline"> Test that the IJKL keys move the triangle locally.  
<input type="checkbox" class="checkbox inline"> And test that the U and O keys rotate the triangle locally.

