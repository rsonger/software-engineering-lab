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

*The remainder of Chapter 4 shows how to apply custom `Geometry` and `Material` objects in various ways, such as creating a visible grid and axes to help orient the viewer as they move through a 3D scene.*  

With the addition of `Geometry` and `Material` objects in the [previous lesson](/software-engineering-lab/_posts/geometry_and_material), we now have all the components necessary for rendering basic objects in a 3D scene. Now let's practice using these base classes to create custom objects with various effects. After experimenting with geometries and materials a little bit, then we will create some extra components to help the user view a 3D scene: a visible grid, axes, and a movable camera.

# Custom Geometries

We will create multiple new test applications throughout this lesson. Each one follows the same basic flow as the `test_4_1.py` application from the previous lesson. Each `startup` method will follow these steps:  
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
# test_4_2.py
from core.app import WindowApp
from core.renderer import Renderer
from core.scene_graph import Scene, Camera, Mesh

from geometry import Geometry
from material.basic_materials import SurfaceMaterial

class Test_4_2(WindowApp):
    """Test rendering a custom geometry with vertex data in the application program."""
    def startup(self):
        print("Starting up Test 4-2...")

        # initialize renderer, scene, and camera
        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspect_ratio=800/600)

        # set the camera's (x,y,z) coordinates
        self.camera.position = (0, 0, 1)

        # prepare the vertex data
        P0 = (-0.1,  0.1, 0.0)
        P1 = ( 0.0,  0.0, 0.0)
        P2 = ( 0.1,  0.1, 0.0)
        P3 = (-0.2, -0.2, 0.0)
        P4 = ( 0.2, -0.2, 0.0)
        position_data = (P0,P3,P1, P1,P3,P4, P1,P4,P2)
        R = (1, 0, 0)
        Y = (1, 1, 0)
        G = (0, 0.25, 0)
        color_data = (R,G,Y, Y,G,G, Y,G,R)

        # create a geometry with custom shape and coloring
        geometry = Geometry()
        geometry.set_attribute("vertexPosition", position_data, "vec3")
        geometry.set_attribute("vertexColor", color_data, "vec3")
        geometry.count_vertices()

        # create a material object to show vertex colors in a gradient
        material = SurfaceMaterial({"useVertexColors": True})

        # create a mesh from the geometry and material, then add it to the scene graph
        self.mesh = Mesh(geometry, material)
        self.scene.add(self.mesh)

    def update(self):
        # render the scene
        self.renderer.render(self.scene, self.camera)

# initialize and run this test
Test_4_2(screen_size=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_2.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a shape in the center of the screen of three adjacent triangles, connected at their yellow and green vertices.  

In order to create the custom shape, we needed to list out each point and color vertex before combining them in the correct order within `position_data` and `color_data`. This can be especially tedious for complicated objects, so it may be better to use functions to calculate this data instead.

## Geometric Functions

When the geometry requires a lot of vertex data, it will likely be easier to write a function that generates the vertices instead of listing them all manually. This next test application shows how we can use a `for` loop to create points for rendering a sine curve.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `test_4_3.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_3.py` for editing and add the following code:  

```python
# test_4_3.py
from math import sin, pi
from numpy import arange

from core.app import WindowApp
from core.renderer import Renderer
from core.scene_graph import Scene, Camera, Mesh
from geometry import Geometry
from material.basic_materials import PointMaterial, LineMaterial

class Test_4_3(WindowApp):
    """Test rendering a custom geometry that uses a function to generate points."""
    def startup(self):
        print("Starting up Test 4-3...")

        # initialize renderer, scene, and camera
        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspect_ratio=800/600)
        self.camera.position = (0, 0, 5)

        # create a geometry for a sine curve using an x range of [-pi, pi)
        position_data = []
        for x in arange(-pi, pi, pi / 12):
            position_data.append((x, sin(x), 0))

        geometry = Geometry()
        geometry.set_attribute("vertexPosition", position_data, "vec3")
        geometry.count_vertices()

        # create materials for drawing the points and a line connecting them
        points = PointMaterial({
            "baseColor": (1,1,0), 
            "pointSize": 10
        })
        line = LineMaterial({
            "baseColor": (1,0,1)
        })

        # create two meshes: one for the points and one for the line
        points_mesh = Mesh(geometry, points)
        line_mesh = Mesh(geometry, line)

        self.scene.add(points_mesh)
        self.scene.add(line_mesh)

    def update(self):
        # render the scene
        self.renderer.render(self.scene, self.camera)

# initialize and run this test
Test_4_3(screen_size=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_3.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a sine curve of thick yellow dots connected by thin magenta lines.  

Here, the point data is generated using the `sin` function from the `math` package. We use the NumPy [`arange`](https://numpy.org/doc/stable/reference/generated/numpy.arange.html) function to get a list of values from $-\pi$ (included) to $+\pi$ (not included) at increments of $\frac{\pi}{12}$ for a total of 24 values. Then the `for` loop steps through each of these values as $x$ and adds the vertex $(x, \sin(x), 0)$ to `position_data`. Altogether this effectively creates a sine curve with 24 vertices in just two lines of code!

# Custom Materials

Custom geometries are useful when we want to make our own vertex data directly inside the application. Likewise, custom materials are useful when we want to change the way the shader program renders colors or points. These next two applications demonstrate custom materials that define their own shader programs.

## Point-Based Coloring

Similar to the way we used a function to define points for a custom geometry above, we can also use functions to define color patterns on the surface of a material. This next test application will use the position data of a fragment to calculate its color. Specifically, it will take the fractional parts of the $(x,y,z)$ coordinates and apply them to the $(r,g,b)$ values of the fragment. We use the fractional parts (the numbers after the decimal points) of the values because they will always fall in between $0.0$ and $1.0$ which is necessary for color data.

We will apply this custom material to a large spinning cube so it is easy to see the pattern it creates.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `test_4_4.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_4.py` for editing and add the following code:  

```python
# test_4_4.py
from math import pi

from core.app import WindowApp
from core.renderer import Renderer
from core.scene_graph import Scene, Camera, Mesh
from geometry.basic_geometries import BoxGeometry
from material import Material

class Test_4_4(WindowApp):
    """Test a custom material that calculates color from position data."""
    def startup(self):
        print("Starting up Test 4-4...")

        # initialize renderer, scene, and camera
        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspect_ratio=800/600)
        self.camera.position = (0, 0, 7)

        # create a large box geometry to show off the color pattern
        geometry = BoxGeometry(3, 3, 3)

        # create a custom material from custom vertex and fragment shader code
        vs_code = """
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

        fs_code = """
        in vec3 position;
        
        out vec4 fragColor;

        void main() {
            vec3 color = mod(position, 1.0);
            fragColor = vec4(color, 1.0);
        }
        """

        material = Material(vs_code, fs_code)

        # create the mesh from the box geometry and custom material
        self.mesh = Mesh(geometry, material)
        self.scene.add(self.mesh)

        # set local constants
        self.rotate_speed_y = pi / 3
        self.rotate_speed_x = pi / 6

    def update(self):
        # animate the spinning cube
        y_distance = self.delta_time * self.rotate_speed_y
        x_distance = self.delta_time * self.rotate_speed_x
        self.mesh.rotateY(y_distance)
        self.mesh.rotateX(x_distance)

        # render the scene
        self.renderer.render(self.scene, self.camera)

# initialize and run this test
Test_4_4(screen_size=(800,600)).run()
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
# test_4_5.py
from math import pi

from core.app import WindowApp
from core.renderer import Renderer
from core.scene_graph import Scene, Camera, Mesh
from geometry.basic_geometries import BoxGeometry
from material import Material

class Test_4_5(WindowApp):
    """Test an animated material that offsets position and color vertices in a rolling wave."""
    def startup(self):
        print("Starting up Test 4-5...")

        # initialize renderer, scene, and camera
        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspect_ratio=800/600)
        self.camera.position = (0, 0, 7)

        # create a large box geometry that will be warped by the material
        geometry = BoxGeometry(6, 6, 6)

        # create a custom material from custom vertex and fragment shader code 
        vs_code ="""
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

        fs_code = """
        uniform float time;

        in vec3 color;

        out vec4 fragColor;

        void main() {
            float r = abs(sin(time));
            vec4 c = vec4(r, -r/2, -r/2, 0);
            fragColor = vec4(color, 1) + c;
        }
        """

        self.wavy_material = Material(vs_code, fs_code)
        self.wavy_material.set_uniform("time", 0, "float")

        # put the mesh together and rotate it before adding it to the scene graph
        self.mesh = Mesh(geometry, self.wavy_material)
        self.mesh.rotate_x(pi/6)
        self.mesh.rotate_y(pi/6)
        self.scene.add(self.mesh)

    def update(self):
        # update the shader time variable to animate the material
        self.wavy_material.set_uniform("time", self.time)

        # render the scene
        self.renderer.render(self.scene, self.camera)

# initialize and run this test
Test_4_5(screen_size=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_5.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a cube with its vertex positions rolling up and down like a wave, and its surface colors changing over time.  

This time we create a new `uniform` variable inside the shader program called `time`. The value for this veriable is set by the application inside its `update` method where it passes its own `self.time` value to the shader variable. Inside the vertex shader, this `time` variable is used in a sine function to calculate an offset value for the $y$ coordinate of each vertex. We also apply the $x$ coordinate value so that the vertices at different horizonal positions will move at different intervals.

Inside the fragment shader, the `time` uniform variable is used to calculate a red saturation value from a sine function and add that to the vertex colors. The same value is also subtracted from the green and blue values at half the magnitude in order to emphasize the red color. This effectively makes the sides of the cube appear to pulse red before returning to their original color again.

# Extra Components

After learning about how to customize geometry and material objects, we can now think about how to use them to create objects that help render a 3D scene. All of our scenes so far have just been shapes floating in a black void. Let's change that by creating a structure to represent the coordinate axes and a grid to help orient the user. After rendering those in a scene, we will then create a special object that handles inputs and allows the user to explore the scene by moving the camera around.

## Axes Helper

The coordinate axes will include three box geometries&mdash;one for each axis. Using the box geometry allows us to give a thickness to the axes without worrying about platform compatibility. (Line width cannot be changed with `glLineWidth` on MacOS.) Here we can use our `Group` class for the base mesh and add to it each of the separate axis meshes as child nodes.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new folder called `extras`.  
<input type="checkbox" class="checkbox inline"> Inside the `extras` folder, create a new file called `__init__.py` so we can import the folder as a package.  
<input type="checkbox" class="checkbox inline"> Inside the `extras` folder, create a new file called `helpers.py`.  
<input type="checkbox" class="checkbox inline"> Open `helpers.py` for editing and add the following code:  

```python
# extras.helpers.py
from core.scene_graph import Mesh, Group
from geometry import Geometry
from geometry.basic_geometries import BoxGeometry
from material.basic_materials import SurfaceMaterial, LineMaterial

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

Each axis is its own mesh with a `BoxGeometry` and a `SurfaceMaterial`. We use `length` for the size of the dimension that corresponds to the given axis while the other two dimensions take the `thickness` value. We use the `baseColor` attribute instead of `vertexColors` for the material since all the vertices will be the same color. Then we translate each mesh so it extends along the positive direction of its axis. All three axes are added to the group node stored in `self.mesh`, which we will use later in our applications.  

## Grid Helper

The next helper component renders lines in a square grid so the user can get a feel for their location when moving around a scene. 

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `helpers.py` file, add the following code after the `AxesHelper` class:  

```python
class GridHelper:
    """Handles the geometry, material, and mesh for a 2D grid initially on the XY plane."""
    def __init__(self, size=10, divisions=10, minor_color=(0,0,0), 
                       major_color=(0.5,0.5,0.5)):
        # prepare position and color data from the parameters
        position_data = []
        color_data = []

        ticks = []
        delta_tick = size/divisions
        for n in range(divisions + 1):
            ticks.append(-size/2 + n*delta_tick)

        # vertical lines
        for x in ticks:
            position_data.append((x, -size/2, 0))
            position_data.append((x,  size/2, 0))
        
            if x == 0:
                color_data.append(major_color)
                color_data.append(major_color)
            else:
                color_data.append(minor_color)
                color_data.append(minor_color)

        # horizontal lines
        for y in ticks:
            position_data.append((-size/2, y, 0))
            position_data.append(( size/2, y, 0))

            if y == 0:
                color_data.append(major_color)
                color_data.append(major_color)
            else:
                color_data.append(minor_color)
                color_data.append(minor_color)

        # put the vertex data into a Geometry
        geometry = Geometry()
        geometry.set_attribute("vertexPosition", position_data, "vec3")
        geometry.set_attribute("vertexColor", color_data, "vec3")
        geometry.count_vertices()

        # create a material for drawing line segments
        material = LineMaterial({
            "useVertexColors": True,
            "lineType": "segments"
        })

        self.mesh = Mesh(geometry, material)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

We calculate coordinates for the grid on the $xy$-plane by setting coordinates for vertical lines first and horizonal lines second. Applications that use the grid mesh will be able to easily rotate it $90°$ in any direction to place it along the desired plane.

The first `for` loop calculates coordinate values of the perpendicular axis for each line and stores those values in the `ticks` list. Then, the second `for` loop uses the values in `ticks` as $x$-coordinates for vertical lines. The third `for` loop also uses the same `ticks` values but sets them to the $y$-coordinates of horizontal lines.

The `major_color` parameter sets the color for the two center lines while the `minor_color` parameter sets the color for all the rest. Like the `AxesHelper` class, the `GridHelper` class also stores a `self.mesh` which our applications will use to render the grid.

Now let's make sure these two helper classes render correctly with a test application.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `test_4_6.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_6.py` for editing and add the following code:  

```python
# test_4_6.py
from math import pi

from core.app import WindowApp
from core.renderer import Renderer
from core.scene_graph import Scene, Camera
from extras.helpers import AxesHelper, GridHelper

class Test_4_6(WindowApp):
    """Test rendering a grid and axes with helper classes."""
    def startup(self):
        print("Starting up Test 4-6...")

        # initialize renderer, scene, and camera
        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspect_ratio=800/600)
        self.camera.position = (5, 2, 7)
        self.camera.rotate_y( pi/6)
        self.camera.rotate_x(-pi/10)

        # use the helper classes to create meshes for the grid and axes
        axes = AxesHelper(length=3)
        grid = GridHelper(size=20, minor_color=(1,1,1), major_color=(1,1,0))
        grid.mesh.rotate_x(-pi/2) # rotate from xy-plane to xz-plane

        self.scene.add(axes.mesh)
        self.scene.add(grid.mesh)

    def update(self):
        # render the scene
        self.renderer.render(self.scene, self.camera)

# initialize and run this test
Test_4_6(screen_size=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_6.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a white and yellow grid stretching out in front of the camera with red, green, and blue axes in the center.  

If all goes well, the application should show the following scene:

![Colorful coordinate axes and grid lines help orient the viewer.](/software-engineering-lab/assets/images/axes_and_grid.png)

## Camera Rig

The final component is a special object that carries the camera as it moves in three dimensions. In addition, the camera will be able to pan up and down without affecting the direction of movement. We can accomplish this by making the `CameraRig` class extend the `Group` class and then add to it the camera object as a child node. Inputs related to movement will apply to the `CameraRig` object itself while inputs for panning the camera will apply as local transformations to the camera. Keeping these as local transformations will make sure the camera always rotates with respect to its parent, the `CameraRig`.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `extras` folder, create a new file called `camera_rig.py`.  
<input type="checkbox" class="checkbox inline"> Open `camera_rig.py` for editing and add the following lines of code:

```python
# extras.camera_rig.py
from math import pi

from core.scene_graph import Group

class CameraRig(Group):
    """A camera that can look up and down while attached to a movable base."""
    def __init__(self, camera, inverted=True, units_per_second=1.5, 
                 degrees_per_second=60):
        super().__init__()

        # attach the camera as a child node of this group node
        self._camera = camera
        self.add(self._camera)

        self._move_speed = units_per_second
        self._rotate_speed = degrees_per_second

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

    def update(self, input, delta_time):
        # calculate distances for moving and rotating since the last frame
        move_amount = self._move_speed * delta_time
        rotate_amount = self._rotate_speed / 180 * pi * delta_time

        # move the body in all directions
        if input.iskeypressed(self.KEY_MOVE_FORWARD):
            self.translate(0, 0, -move_amount)
        if input.iskeypressed(self.KEY_MOVE_BACKWARD):
            self.translate(0, 0,  move_amount)
        if input.iskeypressed(self.KEY_MOVE_LEFT):
            self.translate(-move_amount, 0, 0)
        if input.iskeypressed(self.KEY_MOVE_RIGHT):
            self.translate( move_amount, 0, 0)
        if input.iskeypressed(self.KEY_MOVE_UP):
            self.translate(0,  move_amount, 0)
        if input.iskeypressed(self.KEY_MOVE_DOWN):
            self.translate(0, -move_amount, 0)

        # turn the body left and right
        if input.iskeypressed(self.KEY_TURN_RIGHT):
            self.rotate_y(-rotate_amount)
        if input.iskeypressed(self.KEY_TURN_LEFT):
            self.rotate_y( rotate_amount)

        # turn the camera to look up or down
        if input.iskeypressed(self.KEY_LOOK_UP):
            self._camera.rotate_x( rotate_amount)
        if input.iskeypressed(self.KEY_LOOK_DOWN):
            self._camera.rotate_x(-rotate_amount)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `CameraRig` class uses an instance of `Input` from `core.app` to handle keyboard inputs. The <kbd>W</kbd><kbd>A</kbd><kbd>S</kbd><kbd>D</kbd> and <kbd>Q</kbd><kbd>E</kbd> keys control moving the rig while the <kbd>J</kbd><kbd>L</kbd> keys will turn it left and right. The <kbd>I</kbd><kbd>K</kbd> keys then pan the camera up and down, depending on whether the `inverted` option is set or not. Notice that the `KEY_LOOK_UP` and `KEY_LOOK_DOWN` inputs apply a rotation to `self._camera` while all the others apply to `self`.

Finally, let's make our last test application to try out the camera rig. 

<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a copy of the file called `test_4_6.py` and change its name to `test_4_7.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_7.py` for editing and add the following import statement just before the class definition:

```python
from extras.camera_rig import CameraRig
```

<input type="checkbox" class="checkbox inline"> Scroll down to the `startup` method and **delete** the following lines of code:

```python
        self.camera.position = (5, 2, 7)
        self.camera.rotate_y( pi/6)
        self.camera.rotate_x(-pi/10)
```

<input type="checkbox" class="checkbox inline"> In place of the deleted code, add the following:

```python
        self.rig = CameraRig(self.camera, inverted=False)
        self.rig.position = (5, 2, 7)
        self.scene.add(self.rig)
```

<input type="checkbox" class="checkbox inline"> Scroll down to the `update` method and add the following code just before `# render the scene`:

```python
        # handle inputs and animations
        self.rig.update(self.input, self.delta_time)
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_7.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can move the camera in all directions in addition to panning it up and down.  

:congratulations: **Congratulations!** :tada:  
You have now completed enough components to start creating some interesting 3D scenes. What other components can you think of that might provide a benefit to programming interactive 3D applications? Try extending `Geometry` and `Material` classes and using other core components to build something fun or practical.