---
# MARP
theme: default
paginate: true

# Jekyll
title: "Building a 3D Scene"
date: 2050-06-14
categories:
  - Notes
classes: wide
toc_sticky: false
---

*This post shows how to apply custom `Geometry` and `Material` objects in various ways, such as creating a visible grid and axes to help orient the viewer as they move through a 3D scene.*  

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

