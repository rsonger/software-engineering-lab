---
# MARP
theme: default
paginate: true

# Jekyll
title: "7_Enter the Matrix"
date: 2025-05-21
categories:
  - Notes
classes: wide
toc_sticky: false
---

*In this lesson, we apply what we learned about calculating geometric transformations to build a `Matrix` class and integrate it with our CG framework.*

Now that we have learned about the usefulness of [matrix calculations](/software-engineering-lab/notes/ch3-1/) for [geometric transformations](/software-engineering-lab/notes/ch3-2/), we can add a new component to our CG framework that can provides matrices for transformations of objects in our apps. Our framework will always use 4x4 matrices so that we can easily handle both 2D and 3D graphics. When rendering in 2D, we simply use a consistent value for all the $z$ coordinates so everything renders on the same plane.

# The `Matrix` Class

Our `Matrix` class will have static methods that return different kinds of transformation matrices from the given parameters. With static methods, we do not need to create an instance of `Matrix` or manage state variables. Simply calling these methods will give us the matrix data we need to do geometric transformations.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, create a new file called `matrix.py`.  
<input type="checkbox" class="checkbox inline"> Open `matrix.py` for editing and add the following code:  

```python
# graphics/core/matrix.py
from math import sin, cos, tan, pi
import numpy as np
```

We first import the `math` functions for sine, cosine and tangent to use when calculating our matrices, and the constant pi for converting the angle of view to radians. We also use the [`numpy.array`](https://numpy.org/doc/stable/reference/generated/numpy.array.html){:target="_blank"} function to create NumPy arrays for each of our matrices. This gives us an advantages of less memory use, shorter processing times, and access to the `@` operator for easy matrix multiplication. Instead of unpacking each matrix and calculating the product manually, we can simply do `product = matrix1 @ matrix2`. 

<input type="checkbox" class="checkbox inline"> Add the next code to `matrix.py` for defining the class and a method for the identity matrix.  

```python
class Matrix:
    """ Provides four-dimensional matrices for various geometric transformations """

    # the 4D identity matrix
    __identity = np.array((
        (1, 0, 0, 0),
        (0, 1, 0, 0),
        (0, 0, 1, 0),
        (0, 0, 0, 1)
    )).astype(float)

    @classmethod
    def identity(cls):
        """ A copy of the 4D identity matrix """
        return cls.__identity.copy()
```

Here we create the identity matrix as a NumPy array and store it to a class variable. Then we make a class method to give access to the identity matrix with the `@classmethod` decorator. Class methods access class variables and methods using the `cls` parameter without creating an instance of the class. Our applications will be able to access these class methods directly on the class itself, for example `Matrix.identity()`.

We are using a class variable and a class method that returns copies of the idendity matrix in order to prevent the original value from being changed. If we do not give a copy of the matrix, then the method would give a reference to the value stored in the class. Then our apps would be able to accidentally change the original identity matrix itself. That would cause all kinds of confusion in our apps! So we make the matrix read-only using this approach.

Note that when we create a NumPy array, all of its values must be the same type. So we will fill each of our matrices with `float` values by calling the `astype()` method on each newly created array. The return type is a NumPy [ndarray](https://numpy.org/doc/stable/reference/generated/numpy.ndarray.html){:target="_blank"}.

<input type="checkbox" class="checkbox inline"> Add the next code to the `Matrix` class for creating a translation matrix.  

```python
    @staticmethod
    def translation(x, y, z):
        """ 4D matrix for the translating along vector <x, y, z> """
        return np.array((
            (1, 0, 0, x),
            (0, 1, 0, y),
            (0, 0, 1, z),
            (0, 0, 0, 1)
        )).astype(float)
```

Here the decorator defines the `translation` method as a *static method*. Static methods are like class methods but static methods do not access the class. We can still call static methods directly on the class itself (for example, `shift_matrix = Matrix.translation(1, 2, 3)`).

The parameters `x`, `y`, and `z` are the components of the translation vector. We simply plug those values into their respective locations within a numpy array representing the translation matrix and return it with all values converted to the `float` type.

<input type="checkbox" class="checkbox inline"> Next, add the following methods to the `Matrix` class that create matrices for rotation around each of the three axes $x$, $y$, and $z$.  

```python
    @staticmethod
    def rotation_x(angle):
        """ 4D matrix for rotating around the x-axis by the given angle in radians """
        c = cos(angle)
        s = sin(angle)
        return np.array((
            (1, 0,  0, 0),
            (0, c, -s, 0),
            (0, s,  c, 0),
            (0, 0,  0, 1)
        )).astype(float)

    @staticmethod
    def rotation_y(angle):
        """ 4D matrix for rotating around the y-axis by the given angle in radians """
        c = cos(angle)
        s = sin(angle)
        return np.array((
            ( c, 0, s, 0),
            ( 0, 1, 0, 0),
            (-s, 0, c, 0),
            ( 0, 0, 0, 1)
        )).astype(float)

    @staticmethod
    def rotation_z(angle):
        """ 4D matrix for rotating around the z-axis by the given angle in radians """
        c = cos(angle)
        s = sin(angle)
        return np.array((
            (c, -s, 0, 0),
            (s,  c, 0, 0),
            (0,  0, 1, 0),
            (0,  0, 0, 1)
        )).astype(float)
```

These methods all take the angle of rotation in radians. Then we can simply calculate sine and cosine of the angle before constructing a matrix with the appropriate values.

<input type="checkbox" class="checkbox inline"> Add the `scale` method to the `Matrix` class for scaling transformations.  

```python
    @staticmethod
    def scale(r, s, t):
        """ 4D matrix for scaling dimensions x, y, and z by magnitudes r, s, and t respectively """
        return np.array((
            (r, 0, 0, 0),
            (0, s, 0, 0),
            (0, 0, t, 0),
            (0, 0, 0, 1)
        )).astype(float)
```

Scaling can happen on any dimension, so we want to allow for scaling each dimension individually. **Uniform scaling** happens when all dimensions scale equally. In that case, we just need to use the same value for `r`, `s`, and `t`.

<input type="checkbox" class="checkbox inline"> Finally, add the `perspective` method to the `Matrix` class for calculating the perspective projection matrix.  

```python
    @staticmethod
    def perspective(angle_of_view=60, aspect_ratio=1, near=0.1, far=1000):
        """ 4D matrix for a projection transformation to the given perspective """
        a = angle_of_view * pi / 180.0
        d = 1.0 / tan(a/2)
        r = aspect_ratio
        b = (near + far) / (near - far)
        c = 2 * near * far / (near - far)
        return np.array((
            (d/r, 0,  0, 0),
            (  0, d,  0, 0),
            (  0, 0,  b, c),
            (  0, 0, -1, 0)
        )).astype(float)
```

This method definition provides values for a default perspective so we do not need to specify them in every app. The angle of view parameter uses degrees for its units, so we need to convert it into radians before applying it to the tangent function for calculating the distance between the projection window and the camera. The depth components `b` and `c` are calculated from the *near clipping distance* and *far clipping distance* as explained in the [previous lesson](http://127.0.0.1:4000/software-engineering-lab/notes/geometric_transformations/#perspective-projection).

Now we have everything we need in the `Matrix` class. But before we can it, we need to update our `Uniform` class so it can handle 4x4 matrix data for uniform variables in our shader programs.

# Updating the `Uniform` Class

Remember that our `Uniform` class manages a link between a vertex buffer and a `uniform` variable in a shader program. The class uses [`glUniform`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glUniform.xhtml){:target="_blank"} functions to assign data based on its data type. Now that we want to use matrix data in our framework, we need to update the `Uniform` class so it can associate matrix data with variables as well.

GLSL uses the `mat4` data type for 4x4 matrices and we can use the `glUniformMatrix4fv` function to upload data for `mat4` shader variables.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open the `openGL.py` file from your `core` folder and scroll down to the `Uniform` class.  
<input type="checkbox" class="checkbox inline"> Find the `_VALID_TYPES` variable inside the `Uniform` class and add ***ONLY*** the `'mat4'` value to the tuple.  

```python
    _VALID_TYPES = ('int','bool','float','vec2','vec3','vec4','mat4')
```

<input type="checkbox" class="checkbox inline"> Then, find the long `if` statement inside the `upload_data` method and add the following code at the end:  

```python
        elif self.data_type == "mat4":
            GL.glUniformMatrix4fv(self.variable_ref, 1, GL.GL_TRUE, self.data)
```

When calling the `glUniformMatrix4fv` function, the second parameter is the number of matrices which will always be `1` for our `Uniform` objects. The third parameter tells OpenGL that our matrix data is stored as an array of *row* vectors. If we ever give the data as an array of *column* vectors (we won't), then that parameter would be `GL.GL_FALSE` instead.

# A Test of Transformations

Now we are ready to build an app that uses our `Matrix` class for geometric transformations. The test app will be a 2D triangle that moves and rotates according to user input. On the left side of the keyboard, the <kbd>W</kbd><kbd>A</kbd><kbd>S</kbd><kbd>D</kbd> keys will control global translation while <kbd>Q</kbd> and <kbd>E</kbd> control global rotation. On the right side, the <kbd>I</kbd><kbd>J</kbd><kbd>K</kbd><kbd>L</kbd> keys will control local translation while <kbd>U</kbd> and <kbd>O</kbd> control local rotation.

Our shader program will use two `uniform mat4` variables&mdash;one for the projection matrix and one for the model matrix. Applying both of these matrices to the position vector will give us the object's position on the projection window. The new vector shader source code looks like this:

```glsl
# GLSL version 330
in vec3 position;
uniform mat4 projectionMatrix;
uniform mat4 modelMatrix;
void main() {
    gl_Position = projectionMatrix * modelMatrix * vec4(position, 1.0);
}
```

Remember that the position vector stays the same and all of the transformations will apply to the `modelMatrix`. Then, the `projectionMatrix` will adjust the positions based on the perspective, which makes shapes look smaller as they move away from the camera.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main working folder, create a new file called `test_7.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_7.py` for editing and add the following code:  

```python
# graphics/test_7.py
from math import pi
import OpenGL.GL as GL

from core.app import WindowApp
from core.openGLUtils import initialize_program
from core.openGL import Attribute, Uniform
from core.matrix import Matrix

class Test_7(WindowApp):
    """ Tests geometric transformations by moving a triangle around the screen """

    def startup(self):
        print("Starting up Test 7...")

        vs_code = """
        in vec3 position;
        uniform mat4 projectionMatrix;
        uniform mat4 modelMatrix;
        void main() {
            gl_Position = projectionMatrix * modelMatrix * vec4(position, 1.0);
        }
        """

        fs_code = """
        out vec4 fragColor;
        void main() {
            fragColor = vec4(1.0, 1.0, 0.0, 1.0);
        }
        """

        self.program_ref = initialize_program(vs_code, fs_code)
```

Next, we create an `Attribute` object for the position data and two `Uniform` objects for the two matrices. We will use `translation` and `rotation_z` to update the model matrix whenever the user moves or rotates the triangle. The `perspective` method is only called once in our `startup` method since the perspective will always stay the same.

<input type="checkbox" class="checkbox inline"> Inside the `startup` method of `test_7.py`, add the following code:  

```python
        # one VAO for the single triangle
        vao_ref = GL.glGenVertexArrays(1)
        GL.glBindVertexArray(vao_ref)

        # triangle vertices are local to the object and do not change
        position_data = ( 
            ( 0.0,  0.3, 0.0 ),
            ( 0.2, -0.3, 0.0 ),
            (-0.2, -0.3, 0.0 )
        )
        self.vertex_count = len(position_data)
        position_attribute = Attribute("vec3", position_data)
        position_attribute.associate_variable(self.program_ref, "position")

        # the model matrix
        m_matrix = Matrix.translation(0, 0, -5)
        self.model_matrix = Uniform("mat4", m_matrix)
        self.model_matrix.locate_variable(self.program_ref, "modelMatrix")

        # the perspective matrix
        p_matrix = Matrix.perspective()
        self.projection_matrix = Uniform("mat4", p_matrix)
        self.projection_matrix.locate_variable(self.program_ref, "projectionMatrix")

        # movement speed in world units per second
        self.move_speed = 1.0
        # rotation speed in radians per second
        self.turn_speed = pi / 2

        # render settings
        GL.glClearColor(0.0, 0.0, 0.0, 1.0)
        GL.glEnable(GL.GL_DEPTH_TEST)
```

Our triangle will be taller than it is wide so that we can see its orientation as it rotates around the screen. When we create the model matrix, we make it from a translation matrix that shifts the triangle backwards down the $z$-axis. Since the camera is located at the origin, we would not be able to see the triangle if it was also on the $z=0$ plane, so we move it in front of the camera at $z=-5$.

The movement speed is set to units in world space. This means that the greater the distance between the object and the camera, the slower it appears to move. On the other hand, the rotation speed is not influenced by the object's distance from the camera, but it does appear to speed up as the object moves away from the center of rotation, the $z$-axis. This will be clear when we compare local rotation (with the $z$-axis at the center of the triangle) to global rotation (with the $z$-axis at the center of the screen).

The last line of the `startup` method is a function that enables OpenGL's depth testing feature. Since we are rendering a 3D scene, we need depth testing to determine whether objects in the scene will block each other from view. There is only one object in the scene, but we turn it on now in case we want to add more objects to the scene later.

<input type="checkbox" class="checkbox inline"> Now add the `update` method to the `Test_7` class with the following code:  

```python
    def update(self):
        # first, get changes to position
        move_amount = self.move_speed * self.delta_time
        turn_amount = self.turn_speed * self.delta_time
```

As we did in the [Animations](/software-engineering-lab/notes/animations/#animations) lesson, we first calculate distances based on the time that has passed between frames. This time we also have a rotation distance to calculate separately from the move distance.

<input type="checkbox" class="checkbox inline"> Next, add code for handling translations in the global context to the `update` method.  

```python
        # global translations
        # "w" is upward movement in the positive y direction
        if self.input.iskeypressed("w"):
            m = Matrix.translation(0, move_amount, 0)
            self.model_matrix.data = m @ self.model_matrix.data
        # "a" is leftward movement in the negative x direction
        if self.input.iskeypressed("a"):
            m = Matrix.translation(-move_amount, 0, 0)
            self.model_matrix.data = m @ self.model_matrix.data
        # "s" is downward movement in the negative y direction
        if self.input.iskeypressed("s"):
            m = Matrix.translation(0, -move_amount, 0)
            self.model_matrix.data = m @ self.model_matrix.data
        # "d" is rightward movement in the positive x direction
        if self.input.iskeypressed("d"):
            m = Matrix.translation(move_amount, 0, 0)
            self.model_matrix.data = m @ self.model_matrix.data
```

Similar to our [Test 4-6](/software-engineering-lab/notes/animations/#incorporating-with-graphics-programs) application, we move the triangle in the direction specified by the key press. Here, we make a translation matrix for the movement and then multiply it by the existing model matrix to get a new model matrix. We are using the `@` operator since both of the matrices are NumPy arrays.

<input type="checkbox" class="checkbox inline"> Now add code for handling global rotations to the `update` method.  

```python
        # global rotations
        # "q" is counterclockwise rotation around the world z-axis
        if self.input.iskeypressed("q"):
            m = Matrix.rotation_z(turn_amount)
            self.model_matrix.data = m @ self.model_matrix.data
        # "e" is clockwise rotation around the world z-axis
        if self.input.iskeypressed("e"):
            m = Matrix.rotation_z(-turn_amount)
            self.model_matrix.data = m @ self.model_matrix.data
```

This code is similar to global translations except we use `Matrix.rotation_z` to get a rotation matrix instead. The <kbd>Q</kbd> key rotates in the positive (counterclockwise) direction and <kbd>E</kbd> rotates in the negative (clockwise) direction.

<input type="checkbox" class="checkbox inline"> After that, add code for handling local translations to the `update` method.  

```python
        # local translations
        # "i" is movement in the triangle's positive y direction
        if self.input.iskeypressed("i"):
            m = Matrix.translation(0, move_amount, 0)
            self.model_matrix.data = self.model_matrix.data @ m
        # "j" is movement in the triangle's negative x direction
        if self.input.iskeypressed("j"):
            m = Matrix.translation(-move_amount, 0, 0)
            self.model_matrix.data = self.model_matrix.data @ m
        # "k" is movement in the triangle's negative y direction
        if self.input.iskeypressed("k"):
            m = Matrix.translation(0, -move_amount, 0)
            self.model_matrix.data = self.model_matrix.data @ m
        # "l" is movement in the triangle's positive x direction
        if self.input.iskeypressed("l"):
            m = Matrix.translation(move_amount, 0, 0)
            self.model_matrix.data = self.model_matrix.data @ m
```

Here we handle local coordinates instead of global coordinates. As discussed in the [previous lesson](/software-engineering-lab/notes/ch3-2/#local-transformations), the only difference is the order in which we compose our model matrices. The order of composition for local transformations is the reverse of that for global transformations, so we apply the model matrix **after** the translation matrix.

<input type="checkbox" class="checkbox inline"> Now add the final code for handling local rotations to the `update` method.  

```python
        # local rotations
        # "u" is counterclockwise rotation around the triangle's center
        if self.input.iskeypressed("u"):
            m = Matrix.rotation_z(turn_amount)
            self.model_matrix.data = self.model_matrix.data @ m
        # "o" is clockwise rotation around the triangle's center
        if self.input.iskeypressed("o"):
            m = Matrix.rotation_z(-turn_amount)
            self.model_matrix.data = self.model_matrix.data @ m
```

As local transformations, these rotations will apply to the object's local coordinate axes and the triangle will always spin in place around its own center when pressing the <kbd>U</kbd> and <kbd>O</kbd> keys.

<input type="checkbox" class="checkbox inline"> Finally, add code for clearing the screen and drawing with our matrix data at the end of the `update` method.  

```python
        # reset the color buffer and the depth buffer
        GL.glClear(GL.GL_COLOR_BUFFER_BIT | GL.GL_DEPTH_BUFFER_BIT)

        # draw the scene
        GL.glUseProgram(self.program_ref)
        
        self.projection_matrix.upload_data()
        self.model_matrix.upload_data()
        GL.glDrawArrays(GL.GL_TRIANGLES, 0, self.vertex_count)

# initialize and run this test
Test_7().run()
```

Since we enabled depth testing, we want to reset the depth buffer every frame along with the color buffer. Then we can specify our shader program and upload the matrix data to draw the triangle. Don't forget to also include the last line for running this test app!

<input type="checkbox" class="checkbox inline"> Save the file and run it with the `python test_7.py` command in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a yellow triangle on the screen.  
<input type="checkbox" class="checkbox inline"> Test that the <kbd>W</kbd><kbd>A</kbd><kbd>S</kbd><kbd>D</kbd> keys move the triangle globally.  
<input type="checkbox" class="checkbox inline"> Test that the <kbd>Q</kbd> and <kbd>E</kbd> keys rotate the triangle globally.  
<input type="checkbox" class="checkbox inline"> Test that the <kbd>I</kbd><kbd>J</kbd><kbd>K</kbd><kbd>L</kbd> keys move the triangle locally.  
<input type="checkbox" class="checkbox inline"> Test that the <kbd>U</kbd> and <kbd>O</kbd> keys rotate the triangle locally.

Now that we have geometric transformations built in to our framework, we can start thinking about 3D objects. Next time we will set up the basic components for rendering a scene with multiple 3D objects in it. Look forward to it!