---
# MARP
theme: default
paginate: true

# Jekyll
title: "Building a 3D Scene"
date: 2022-06-14
categories:
  - Notes
classes: wide
toc_sticky: false
---

*The remainder of Chapter 4 shows how to apply our newest framework components in various ways, such as creating a visible grid and axes to help orient the viewer.*  

With the addition of `Geometry` and `Material` objects in the [previous lesson](/software-engineering-lab/_posts/geometry_and_material), we now have all the components necessary for rendering basic objects in a 3D scene. Now let's practice using these base classes to create custom objects with various effects. After experimenting with geometries and materials a little bit, then we will create some extra components to help the user view a 3D scene: a visible grid, axes, and a movable camera.

# Custom Geometries

We will create multiple new test applications throughout this lesson. Each one follows the same basic flow as the `test_4_1.py` application from the previous lesson. Each `initialize` method will follow these steps:  
1. Initialize the `Renderer`, `Scene`, and `Camera` objects.
2. Set the camera position.
3. Create and configure `Geometry` and `Material` instances for objects in the scene.
4. Initialize `Mesh` objects with the given geometries and materials.
5. Add each `Mesh` object to the scene graph.
6. Define constants to be used by the application (movement/rotation speeds, etc.)

Then, the `update` method of each application will do the following:
1. Calculate changes to the scene if necessary.
2. Handle inputs.
3. Update animations.
4. Render the scene. 

## Custom Shapes

Using just an instance of the base `Geometry` class, it is possible to create our own shapes. All we need to do is directly set the vertex data inside our application and set the attributes on our geometry object.

This first test application will create a custom shape from three adjacent triangles and render its vertices with a color gradient using the `SurfaceMaterial` class.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `test_4_2.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_2.py` for editing and add the following code:  

```python
from core.base import Base
from core.renderer import Renderer
from core.scene import Scene
from core.camera import Camera
from core.mesh import Mesh

from geometry.geometry import Geometry

from material.surface_material import SurfaceMaterial

class Test_4_2(Base):
    """Test rendering a custom geometry with vertex data in the application program."""
    def initialize(self):
        print("Starting up Test 4-2...")

        # initialize renderer, scene, and camera
        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspectRatio=800/600)

        # set up the camera's position
        self.camera.setPosition((0, 0, 1))

        # prepare the vertex data
        P0 = (-0.1,  0.1, 0.0)
        P1 = ( 0.0,  0.0, 0.0)
        P2 = ( 0.1,  0.1, 0.0)
        P3 = (-0.2, -0.2, 0.0)
        P4 = ( 0.2, -0.2, 0.0)
        positionData = (P0,P3,P1, P1,P3,P4, P1,P4,P2)
        R = (1, 0, 0)
        Y = (1, 1, 0)
        G = (0, 0.25, 0)
        colorData = (R,G,Y, Y,G,G, Y,G,R)

        # create a geometry with custom shape and coloring
        geometry = Geometry()
        geometry.setAttribute("vertexPosition", positionData, "vec3")
        geometry.setAttribute("vertexColor", colorData, "vec3")
        geometry.countVertices()

        # create a material object to show vertex colors in a gradient
        material = SurfaceMaterial({"useVertexColors": True})

        # create a mesh from the geometry and material, then add it to the scene graph
        self.mesh = Mesh(geometry, material)
        self.scene.add(self.mesh)

    def update(self):
        # render the scene
        self.renderer.render(self.scene, self.camera)

# initialize and run this test
Test_4_2(screenSize=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_2.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a shape in the center of the screen of three adjacent triangles, connected at their yellow and green vertices.  

In order to create the custom shape, we needed to list out each point and color vertex before combining them in the correct order within `positionData` and `colorData`. This can be especially tedious for complicated objects, so it may be better to use functions to calculate this data instead.

## Geometric Functions

When the geometry requires a lot of vertex data, it will likely be easier to write a function that generates the vertices instead of listing them all manually. This next test application shows how we can use a `for` loop to create points for rendering a sine curve.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `test_4_3.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_3.py` for editing and add the following code:  

```python
from math import sin, pi

from numpy import arange

from core.base import Base
from core.renderer import Renderer
from core.scene import Scene
from core.camera import Camera
from core.mesh import Mesh
from geometry.geometry import Geometry
from material.point_material import PointMaterial
from material.line_material import LineMaterial

class Test_4_3(Base):
    """Test rendering a custom geometry that uses a function to generate points."""
    def initialize(self):
        print("Starting up Test 4-3...")

        # initialize renderer, scene, and camera
        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspectRatio=800/600)
        self.camera.setPosition((0, 0, 5))

        # create a geometry for a sine curve using an x range of [-pi, pi)
        positionData = []
        for x in arange(-pi, pi, pi / 12):
            positionData.append((x, sin(x), 0))
        geometry = Geometry()
        geometry.setAttribute("vertexPosition", positionData, "vec3")
        geometry.countVertices()

        # create materials for drawing the points and a line connecting them
        points = PointMaterial({"baseColor": (1,1,0), "pointSize": 10})
        line = LineMaterial({"baseColor": (1,0,1)})

        # create two meshes: one for the points and one for the line
        pointsMesh = Mesh(geometry, points)
        lineMesh = Mesh(geometry, line)

        self.scene.add(pointsMesh)
        self.scene.add(lineMesh)

    def update(self):
        # render the scene
        self.renderer.render(self.scene, self.camera)

# initialize and run this test
Test_4_3(screenSize=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_3.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a sine curve of thick yellow dots connected by thin magenta lines.  

Here, the point data is generated using the `sin` function from the `math` package. We use the NumPy [`arange`](https://numpy.org/doc/stable/reference/generated/numpy.arange.html) function to get a list of values from $-\pi$ (included) to $+\pi$ (not included) at increments of $\frac{\pi}{12}$ for a total of 24 values. Then the `for` loop steps through each of these values as $x$ and appends the vertex $(x, \sin(x), 0)$ to `positionData`. This creates the shape of the curve with 24 vertices in just two lines of code!

# Custom Materials

Custom geometries are useful when we want to make our own vertex data directly inside the application. Likewise, custom materials are useful when we want to change the way the shader program renders colors or points. These next two applications demonstrate custom materials that define their own shader programs.

## Point-Based Coloring

Similar to the way we used a function to define points for a custom geometry above, we can also use functions to define color patterns on the surface of a material. This next test application will use the position data of a fragment to calculate its color. Specifically, it will take the fractional parts of the $(x,y,z)$ coordinates and apply them to the $(r,g,b)$ values of the fragment. We use the fractional parts (the numbers after the decimal points) of the values because they will always fall in between $0.0$ and $1.0$ which is necessary for color data.

We will apply this custom material to a large spinning cube so it is easy to see the pattern it creates.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `test_4_4.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_4.py` for editing and add the following code:  

```python
from math import pi

from core.base import Base
from core.renderer import Renderer
from core.scene import Scene
from core.camera import Camera
from core.mesh import Mesh

from geometry.box_geometry import BoxGeometry

from material.material import Material

class Test_4_4(Base):
    """Test a custom material that calculates color from position data."""
    def initialize(self):
        print("Starting up Test 4-4...")

        # initialize renderer, scene, and camera
        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspectRatio=800/600)
        self.camera.setPosition((0, 0, 7))

        # create a large box geometry to show off the color pattern
        geometry = BoxGeometry(3, 3, 3)

        # create a custom material from custom vertex and fragment shader code
        vsCode = """
        uniform mat4 modelMatrix;
        uniform mat4 viewMatrix;
        uniform mat4 projectionMatrix;

        in vec3 vertexPosition;
        
        out vec3 position;

        void main() {
            gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(vertexPosition, 1);
            position = vertexPosition;
        }
        """

        fsCode = """
        in vec3 position;
        
        out vec4 fragColor;

        void main() {
            vec3 color = mod(position, 1.0);
            fragColor = vec4(color, 1.0);
        }
        """

        material = Material(vsCode, fsCode)

        # create the mesh from the box geometry and custom material
        self.mesh = Mesh(geometry, material)
        self.scene.add(self.mesh)

        # set local constants
        self.rotate_speed_y = pi / 3
        self.rotate_speed_x = pi / 6

    def update(self):
        # animate the spinning cube
        y_distance = self.deltaTime * self.rotate_speed_y
        x_distance = self.deltaTime * self.rotate_speed_x
        self.mesh.rotateY(y_distance)
        self.mesh.rotateX(x_distance)

        # render the scene
        self.renderer.render(self.scene, self.camera)

# initialize and run this test
Test_4_4(screenSize=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_4.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a spinning cube with color patterns of square gradients along its surface.  

This test application includes the source code for new vertex shader and fragment shader programs. The vertex shader simply takes data from `vertexPosition` and applies the `modelMatrix`, `viewMatrix`, and `projectionMatrix` as we did before. The difference is that the vertex shader then sends the `vertexPosition` data back out to the fragment shader code so it can use the position data to calculate color.

The fragment shader takes in the `position` data and applies the [`mod`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/mod.xhtml) function with the parameter `1.0` to get the fractional parts of the $(x,y,z)$ coordinates. It then directly applies those values to the $(r,g,b)$ values of the fragment color. So when we run the program, we should see a pattern of square gradients going across the surfaces of the spinning cube.

## Animated Materials

The final test application for custom geometries and materials demonstrates how to create animations with the shader code itself. The application will render a cube that changes shape and color over time.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `test_4_5.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_5.py` for editing and add the following code:  

```python
from math import pi

from core.base import Base
from core.renderer import Renderer
from core.scene import Scene
from core.camera import Camera
from core.mesh import Mesh

from geometry.box_geometry import BoxGeometry

from material.material import Material

class Test_4_5(Base):
    """Test an animated material that offsets position and color vertices in a rolling wave."""
    def initialize(self):
        print("Starting up Test 4-5...")

        # initialize renderer, scene, and camera
        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspectRatio=800/600)
        self.camera.setPosition((0, 0, 7))

        # create a large box geometry that will be warped by the material
        geometry = BoxGeometry(6, 6, 6)

        # create a custom material from custom vertex and fragment shader code 
        vsCode ="""
        uniform mat4 modelMatrix;
        uniform mat4 viewMatrix;
        uniform mat4 projectionMatrix;
        uniform float time;

        in vec3 vertexPosition;
        in vec3 vertexColor;

        out vec3 color;

        void main() {
            float offset = sin(6.0 * vertexPosition.x + time);
            vec4 pos = vec4(vertexPosition, 1) + vec4(0, offset, 0, 1);
            gl_Position = projectionMatrix * viewMatrix * modelMatrix * pos;
            color = vertexColor;
        }
        """

        fsCode = """
        uniform float time;

        in vec3 color;

        out vec4 fragColor;

        void main() {
            float r = abs(sin(time));
            vec4 c = vec4(r, -r/2, -r/2, 0);
            fragColor = vec4(color, 1) + c;
        }
        """

        self.wavyMaterial = Material(vsCode, fsCode)
        self.wavyMaterial.setUniform("time", 0, "float")

        # put the mesh together and rotate it before adding it to the scene graph
        self.mesh = Mesh(geometry, self.wavyMaterial)
        self.mesh.rotateX(pi/6)
        self.mesh.rotateY(pi/6)
        self.scene.add(self.mesh)

    def update(self):
        # update the shader time variable to animate the material
        self.wavyMaterial.setUniform("time", self.time)

        # render the scene
        self.renderer.render(self.scene, self.camera)

# initialize and run this test
Test_4_5(screenSize=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_5.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a cube with its vertex positions rolling up and down like a wave, and its surface colors changing over time.  

This time we create a new `uniform` variable inside the shader programs called `time`. The value for this veriable is set by the application inside its `update` method where it passes its own `self.time` value to the shader variable. Inside the vertex shader, this `time` variable is used in a sine function to calculate an offset value for the $y$ coordinate of each vertex. We also apply the $x$ coordinate value so that the vertices at different horizonal positions will move at different intervals.

Inside the fragment shader, the `time` uniform variable is used to calculate a red saturation value from a sine function and add that to the vertex colors. The same value is also subtracted from the green and blue values at half the magnitude in order to emphasize the red color. This effectively makes the sides of the cube appear to pulse red before returning to their original color again.

# Extra Components

After learning about how to customize geometry and material objects, we can now think about how to use them to create objects that help render a 3D scene. All of our scenes so far have been just shapes floating in a black void. Let's change that by creating a structure to represent the coordinate axes and a grid to help orient the user. After rendering those in a scene, we will then create a special object that handles inputs and allows the camera to move around the scene to explore it.

## Axes Helper

The coordinate axes will include three box geometries--one for each axis. Using the box geometry allows us to give a thickness to the axes without worrying about platform compatibility. (The `glLineWidth` function is deprecated and does not work on MacOS.) Here we can use our `Group` class as the base mesh for the axes and add each separate axis as a child to it.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new folder called `extras`.  
<input type="checkbox" class="checkbox inline"> Inside the `extras` folder, create a new file called `__init__.py` so we can import the folder as a package.  
<input type="checkbox" class="checkbox inline"> Inside the `extras` folder, create a new file called `axes_helper.py`.  
<input type="checkbox" class="checkbox inline"> Open `axes_helper.py` for editing and add the following code:  

```python
from core.mesh import Mesh
from core.group import Group

from geometry.box_geometry import BoxGeometry

from material.surface_material import SurfaceMaterial

class AxesHelper:
    """Creates a mesh to render the 3 coordinate axes in different colors."""
    def __init__(self, length=1, thickness=0.1, 
                       colors=((1,0,0), (0,1,0), (0,0,1))):
        self.mesh = Group() # parent node for the three axes

        # how far to move the box in the positive direction of each axis
        offset = length/2 + thickness/2

        # the x-axis is a is a long, narrow, red box
        x_mesh = Mesh(
            BoxGeometry(length, thickness, thickness), 
            SurfaceMaterial({"baseColor": colors[0]})
        )
        x_mesh.translate(offset, 0, 0)

        # the y-axis is a long, narrow, green box
        y_mesh = Mesh(
            BoxGeometry(thickness, length, thickness), 
            SurfaceMaterial({"baseColor": colors[1]})
        )
        y_mesh.translate(0, offset, 0)

        # the z-axis is a long, narrow, blue box
        z_mesh = Mesh(
            BoxGeometry(thickness, thickness, length), 
            SurfaceMaterial({"baseColor": colors[2]})
        )
        z_mesh.translate(0, 0, offset)

        self.mesh.add(x_mesh)
        self.mesh.add(y_mesh)
        self.mesh.add(z_mesh)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Each axis is its own mesh with a `BoxGeometry` and a `SurfaceMaterial`. We set the size of the dimension that corresponds to the axis as the value of `length` while the other two dimensions are set to the `thickness` value. We use `baseColor` instead of `vertexColors` for the material since all the vertices will be the same color. Then we translate each mesh so it extends along the positive direction of its axis. All three axes are added to the group node stored in `self.mesh`, which we will use later in our applications.  

## Grid Helper

The next helper component renders lines in a square grid so the user can get a feel for their location when moving around a scene. 

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `grid_helper.py`.  
<input type="checkbox" class="checkbox inline"> Open `grid_helper.py` for editing and add the following code:  

```python
from core.mesh import Mesh

from geometry.geometry import Geometry

from material.line_material import LineMaterial

class GridHelper:
    """Handles the geometry, material, and mesh for a 2D grid initially on the XY plane."""
    def __init__(self, size=10, divisions=10, minorColor=(0,0,0), majorColor=(0.5,0.5,0.5)):
        # prepare position and color data from the parameters
        positionData = []
        colorData = []

        ticks = []
        delta_tick = size/divisions
        for n in range(divisions + 1):
            ticks.append(-size/2 + n*delta_tick)

        # vertical lines
        for x in ticks:
            positionData.append((x, -size/2, 0))
            positionData.append((x,  size/2, 0))
        
            if x == 0:
                colorData.append(majorColor)
                colorData.append(majorColor)
            else:
                colorData.append(minorColor)
                colorData.append(minorColor)

        # horizontal lines
        for y in ticks:
            positionData.append((-size/2, y, 0))
            positionData.append(( size/2, y, 0))

            if y == 0:
                colorData.append(majorColor)
                colorData.append(majorColor)
            else:
                colorData.append(minorColor)
                colorData.append(minorColor)

        # put the vertex data into a Geometry
        geometry = Geometry()
        geometry.setAttribute("vertexPosition", positionData, "vec3")
        geometry.setAttribute("vertexColor", colorData, "vec3")
        geometry.countVertices()

        # create a material for drawing line segments
        material = LineMaterial({
            "useVertexColors": True,
            "lineType": "segments"
        })

        self.mesh = Mesh(geometry, material)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

We calculate coordinates for the grid on the $xy$-plane by setting coordinates for vertical lines first and horizonal lines second. Our applications will be able to easily rotate the grid $90°$ in any direction and place it along the desired plane.

The first `for` loop calculates the coordinate values for each line that will be drawn with respect to its perpendicular axis and stores those values in the `ticks` list. Then, the second `for` loop uses each value in `ticks` as the $x$-coordinates for vertical lines. The third `for` loop also uses the same `ticks` values but sets them to the $y$-coordinates of the horizontal lines.

The `majorColor` parameter sets the color for only the center lines while the `minorColor` parameter sets the color for all the rest. Like the `AxesHelper` class, the `GridHelper` class also stores a `self.mesh` which our applications can use to render the grid.

Now let's make sure these two helper classes render correctly with a test application.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `test_4_6.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_6.py` for editing and add the following code:  

```python
from math import pi

from core.base import Base
from core.renderer import Renderer
from core.scene import Scene
from core.camera import Camera

from extras.axes_helper import AxesHelper
from extras.grid_helper import GridHelper

class Test_4_6(Base):
    """Test rendering a grid and axes with helper classes."""
    def initialize(self):
        print("Starting up Test 4-6...")

        # initialize renderer, scene, and camera
        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspectRatio=800/600)
        self.camera.setPosition((5, 2, 7))
        self.camera.rotateY( pi/6)
        self.camera.rotateX(-pi/10)

        # use the helper classes to create meshes for the grid and axes
        axes = AxesHelper(length=3)
        grid = GridHelper(size=20, minorColor=(1,1,1), majorColor=(1,1,0))
        grid.mesh.rotateX(-pi/2) # rotate from xy-plane to xz-plane

        self.scene.add(axes.mesh)
        self.scene.add(grid.mesh)

    def update(self):
        # render the scene
        self.renderer.render(self.scene, self.camera)

# initialize and run this test
Test_4_6(screenSize=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_6.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a white and yellow grid stretching out in front of the camera with red, green, and blue axes in the center.  

If all goes well, the application should show the following scene:

![Colorful coordinate axes and grid lines help orient the viewer.](/software-engineering-lab/assets/images/axes_and_grid.png)

## Camera Rig

The final component is a special object that can move in three dimensions with the scene's camera attached to it. The camera will also be able to pan up and down without changing the direction of movement. We can accomplish this by making the `CameraRig` class extend the `Group` class and then add the camera object as its child. Then, inputs related to movement will apply to the `CameraRig` object itself while inputs related to rotating the camera will apply only to the camera. Keeping these as local transformations will make sure the camera always rotates with respect to its parent, the `CameraRig`.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `grid_helper.py`.  
<input type="checkbox" class="checkbox inline"> Open `grid_helper.py` for editing and add the following lines of code:

```python
from math import pi

from core.group import Group
from core.camera import Camera

class CameraRig(Group):
    """A camera that can look up and down while attached to a movable base."""
    def __init__(self, camera, inverted=True, unitsPerSecond=1.5, degreesPerSecond=60):
        super().__init__()

        # attach the camera as a child node of this group node
        self._camera = camera
        self.add(self._camera)

        self._move_speed = unitsPerSecond
        self._rotate_speed = degreesPerSecond

        self.KEY_MOVE_FORWARD = 'w'
        self.KEY_MOVE_BACKWARD = 's'
        self.KEY_MOVE_LEFT = 'a'
        self.KEY_MOVE_RIGHT = 'd'
        self.KEY_MOVE_UP = 'e'
        self.KEY_MOVE_DOWN = 'q'
        self.KEY_TURN_LEFT = 'j'
        self.KEY_TURN_RIGHT = 'l'
        if inverted:
            self.KEY_LOOK_UP = 'k'
            self.KEY_LOOK_DOWN = 'i'
        else:
            self.KEY_LOOK_UP = 'i'
            self.KEY_LOOK_DOWN = 'k'

    def update(self, input, deltaTime):
        # calculate distances for moving and rotating since the last frame
        move_amount = self._move_speed * deltaTime
        rotate_amount = self._rotate_speed / 180 * pi * deltaTime

        # move the body in all directions
        if input.isKeyPressed(self.KEY_MOVE_FORWARD):
            self.translate(0, 0, -move_amount)
        if input.isKeyPressed(self.KEY_MOVE_BACKWARD):
            self.translate(0, 0,  move_amount)
        if input.isKeyPressed(self.KEY_MOVE_LEFT):
            self.translate(-move_amount, 0, 0)
        if input.isKeyPressed(self.KEY_MOVE_RIGHT):
            self.translate( move_amount, 0, 0)
        if input.isKeyPressed(self.KEY_MOVE_UP):
            self.translate(0,  move_amount, 0)
        if input.isKeyPressed(self.KEY_MOVE_DOWN):
            self.translate(0, -move_amount, 0)

        # turn the body left and right
        if input.isKeyPressed(self.KEY_TURN_RIGHT):
            self.rotateY(-rotate_amount)
        if input.isKeyPressed(self.KEY_TURN_LEFT):
            self.rotateY( rotate_amount)

        # turn the camera to look up or down
        if input.isKeyPressed(self.KEY_LOOK_UP):
            self._camera.rotateX( rotate_amount)
        if input.isKeyPressed(self.KEY_LOOK_DOWN):
            self._camera.rotateX(-rotate_amount)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `CameraRig` class uses an instance of `Input` from `core.input` to handle keyboard inputs. The **WASD** and **QE** keys control moving the rig while the **JL** keys will turn it left and right. The **IK** keys then pan the camera up and down, depending on whether the `inverted` option is set or not. Notice that the `KEY_LOOK_UP` and `KEY_LOOK_DOWN` inputs apply a rotation to `self._camera` while all the others apply to `self`.

Finally, let's make our last test application to try out the camera rig. 

<input type="checkbox" class="checkbox inline"> Inside your main working folder, copy the file called `test_4_6.py` and save it as `test_4_7.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_7.py` for editing and add the following import statement just before the class definition:

```python
from extras.camera_rig import CameraRig
```

<input type="checkbox" class="checkbox inline"> Scroll down to the `initialize` method and **delete** the following lines of code:

```python
        self.camera.setPosition((5, 2, 7))
        self.camera.rotateY( pi/6)
        self.camera.rotateX(-pi/10)
```

<input type="checkbox" class="checkbox inline"> In place of the deleted code, add the following:

```python
        self.rig = CameraRig(self.camera, inverted=False)
        self.rig.setPosition((5, 2, 7))
        self.scene.add(self.rig)
```

<input type="checkbox" class="checkbox inline"> Scroll down to the `update` method and add the following code just before `# render the scene`:

```python
        # handle inputs and animations
        self.rig.update(self.input, self.deltaTime)
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_7.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can move the camera in all directions in addition to panning it up and down.  

:congratulations: **Congratulations!** :tada:  
You have now completed enough components to start creating some interesting 3D scenes. What other components can you think of that might provide a benefit to programming interactive 3D applications? Try extending `Geometry` and `Material` classes and using other core components to build something fun or practical.