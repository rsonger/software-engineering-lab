---
# MARP
theme: default
paginate: true

# Jekyll
title: "9_3D Scenes with Geometry and Material Objects"
date: 2023-06-07
categories:
  - Notes
classes: wide
toc_sticky: false
---

*[Geometry Objects](#geometry-objects) outlines the `Geometry` class and its extensions which store attribute data related to the vertices of different objects.*  
*[Material Objects](#material-objects) outlines the `Material` class and its extensions which store the shader program, uniform data, and rendering settings related to the appearance of different objects.*

In the previous lesson, we looked at the overview of our scene graph framework and created many of the basic components, including `Object3D`, `Scene`, `Group`, `Camera`, and `Mesh`. At the time, we also created an empty `Geometry` class and `Material` class. This lesson covers those two classes in full detail along with some simple extensions of each one.

![A class diagram shows Renderer, Geometry, Material, BoxGeometry, BasicMaterial, and SurfaceMaterial as the targets for this lesson.](/software-engineering-lab/assets/images/scene_graph_uml-2.png)

The classes we make this time include `BoxGeometry`, `BasicMaterial`, and `SurfaceMaterial` which will allow us to render a 3D box with sides of many colors. The `BasicMaterial` class maintains a reference to a basic shader program that renders vertices with a white base color if colors for specific vertices are not provided. The `BoxGeometry` class will define separate vertex colors so each side of the box will be a different color and we can see the box clearly. Extensions to the `BasicMaterial` class will manage render settings for different draw styles such as drawing points, lines, or surfaces.

Finally, we will create the `Renderer` class that uses `Scene` and `Camera` to draw each `Mesh`. This will be the last component we need to draw a 3D box by combining the `BoxGeometry` and `SurfaceMaterial` in a test application. 

# Geometry Objects

The `Geometry` class contains a dictionary for each `Attribute` of a specified geometric object as well as the number of vertices. It is intended to hold only the common behavior among all geometry objects, so each one of its extensions will define a certain type of geometric object with specific vertex data and attributes.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open your `__init__.py` file from the `geometry` folder inside your main working folder.  
<input type="checkbox" class="checkbox inline"> Delete the word `pass` inside the `Geometry` class.  
<input type="checkbox" class="checkbox inline"> Add the following code to the `Geometry` class:  

```python
    """Store attribute data and their total number of vertices."""
    def  __init__(self):
        self._attributes = {}
        self._vertex_count = None

    @property
    def attributes(self):
        return self._attributes

    @property
    def vertex_count(self):
        if self._vertex_count is None:
            self.count_vertices()
        return self._vertex_count

    def set_attribute(self, variable_name, data, data_type=None):
        """Add or update an attribute of this geometric object."""
        if variable_name in self._attributes.keys():
            self._attributes[variable_name].data = data
            self._attributes[variable_name].upload_data()
        elif data_type is not None:
            self._attributes[variable_name] = Attribute(data_type, data)
        else:
            raise Exception("A new Geometry attribute must have a data type.")
    
    def count_vertices(self, variable_name=None):
        """Count the number of vertices as the length of an attribute's data."""
        if len(self._attributes) == 0:
            self._vertex_count = 0
        if variable_name is not None:
            self._vertex_count = len(self._attributes[variable_name].data)
        else:
            self._vertex_count = len(list(self._attributes.values())[0].data)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `set_attribute` method stores an instance of `Attribute` for the specified shader program variable, or updates an existing one. The name of the shader variable is also the key to accessing the `Attribute` in the dictionary. After attributes have been set, we can use the `count_vertices` method to get the number of vertices. Here we use the single underscore naming convention with `_attributes` and `_vertex_count` to show that these variables are for internal use to the `Geometry` class. Then, the getter properties will return the variable values for external use. Properties are useful in this way because we can confirm that `self._vertex_count` has a value before returning it.

Now that we have the base `Geometry` defined, we can create specific geometric objects such as rectangles and boxes.

## Rectangles

The first geometric object is a simple 2D rectangle with its origin at the center. The `RectangleGeometry` class will take two parameters&mdash;the width and height as they define two perpendicular sides of the rectangle. Then, with its origin at the center, we can get the $x$ and $y$ coordinates of the four points by halving the height and width values.

![A rectangle is drawn from two triangles defined by its width and height.](/software-engineering-lab/assets/images/rectangle_geometry.png)

Since OpenGL only provides triangles as a way to draw filled-in shapes, our rectangles will be drawn from two right triangles sharing the same hypotenuse. Because of this, we need to define our vertices in a way that draws each triangle as a separate group. That means our position data for each rectangle will have six vertices instead of four. Also, the vertices must be listed in counterclockwise order so that front sides of the triangles face in the positive $z$ direction.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `geometry` folder, create a new file called `basic_geometries.py`.  
<input type="checkbox" class="checkbox inline"> Open the `basic_geometries.py` file for editing and add the following code:  

```python
# geometry.basic_geometries.py
from geometry import Geometry

class RectangleGeometry(Geometry):
    """A rectangular geometric object centered at (0,0) with a given width and height."""
    def __init__(self, width=1, height=1):
        super().__init__()

        # position vertices
        P0 = (-width/2, -height/2, 0)
        P1 = ( width/2, -height/2, 0)
        P2 = (-width/2,  height/2, 0)
        P3 = ( width/2,  height/2, 0)

        # color data for white, red, green, and blue vertices
        C0, C1, C2, C3 = (1,1,1), (1,0,0), (0,1,0), (0,0,1)

        position_data = (P0,P1,P3, # first triangle
                        P0,P3,P2) # second triangle
        color_data = (C0,C1,C3, # first triangle
                     C0,C3,C2) # second triangle

        self.set_attribute("vertexPosition", position_data, "vec3")
        self.set_attribute("vertexColor", color_data, "vec3")

        self.count_vertices()
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The color vertices are listed in the same order as the position vertices. This will make the shared points of the two triangles the same color with a smooth transition between both sides. That is, the bottom-left corner will be completely white and the upper-right corner will be completely blue. If the color vertices do not align, then we will be able to see a clear divide between the colors running diagonally through the rectangle.

## Boxes

Our first three-dimensional shape will be a box with eight points and six sides.
Since each side must be drawn as two rectangles, we must group our vertices to form a total of 12 triangles. And since each triangle is a collection of three points, the position and color data will have a total of 36 vertices each. As with the rectangle, our `BoxGeometry` class will only take the shape's dimensions as parameters&mdash;the width, height, and depth as they define the lengths of the sides parellel to the $x$, $y$, and $z$ axes respectively. Again, we need to halve the values of each to get the $x$, $y$, and $z$ coordinates of the points since the center of the box will be at $(0,0,0)$.

![A box is drawn with eight points and six sides.](/software-engineering-lab/assets/images/box_geometry.png)

As before, we need to be careful about the ordering of the vertices. Each triangle group must list the vertices in counterclockwise order for the direction they are facing. For example, the front-facing triangles will be in order of $(P4, P5, P7)$ and $(P4, P7, P6)$ but the back-facing triangles will be in order of $(P1, P0, P2)$ and $(P1, P2, P3)$. An easier way to visualize this is by "unfolding" the 3D shape so all the surfaces are arranged facing towards the viewer on a flat plane. This is called a *net diagram* and it helps us to easily see the correct ordering of the vertices.

![A net diagram visualizes the correct ordering of the vertices by unfolding the box.](/software-engineering-lab/assets/images/box_net_diagram.png)

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `basic_geometries.py` file, add the following code after the `RectangleGeometry` class:  

```python
class BoxGeometry(Geometry):
    """A 3D box centered at the origin with given width, height, and depth."""
    def __init__(self, width=1, height=1, depth=1):
        super().__init__()

        # position vertices
        P0 = (-width/2, -height/2, -depth/2)
        P1 = ( width/2, -height/2, -depth/2)
        P2 = (-width/2,  height/2, -depth/2)
        P3 = ( width/2,  height/2, -depth/2)
        P4 = (-width/2, -height/2,  depth/2)
        P5 = ( width/2, -height/2,  depth/2)
        P6 = (-width/2,  height/2,  depth/2)
        P7 = ( width/2,  height/2,  depth/2)

        # color vertex data for each side
        C1 = [(1,0,0)] * 6  # six red vertices
        C2 = [(1,1,0)] * 6  # six yellow vertices
        C3 = [(0,1,0)] * 6  # six green vertices
        C4 = [(0,1,1)] * 6  # six cyan vertices
        C5 = [(0,0,1)] * 6  # six blue vertices
        C6 = [(1,0,1)] * 6  # six magenta vertices

        position_data = (P5,P1,P3, P5,P3,P7, # right side
                         P0,P4,P6, P0,P6,P2, # left side
                         P6,P7,P3, P6,P3,P2, # top side
                         P0,P1,P5, P0,P5,P4, # bottom side
                         P4,P5,P7, P4,P7,P6, # front side
                         P1,P0,P2, P1,P2,P3) # back side
        
        # create a list of 36 RGB vertices
        color_data = C1 + C2 + C3 + C4 + C5 + C6 
        
        self.set_attribute("vertexPosition", position_data, "vec3")
        self.set_attribute("vertexColor", color_data, "vec3")
        self.count_vertices()
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Again, the order of the vertex color data will decide the color of each side. The six red vertices are first, so they are assigned to the right side. Then yellow vertices for the left side, green for the top side, cyan for the bottom side, blue for the front, and magenta for the back. These sides will be solid colors, so we could take advantage of the `*` operator for lists and create six copies of each color vertex. If we wanted to see color gradients on each side, we would need to write out each color vertex similar to the way we write out the position vertices. 

# Material Objects

While the `Geometry` objects manage geometric data concerning the shape, position, and color of an object's vertices, the `Material` objects manage data related to rendering the object including the shader program itself, `Uniform` objects, and OpenGL render settings. 

## Base Class

The base class of `Material` will compile and initialize the shader program, store and manage uniform objects in a dictionary, and handle OpenGL-specific settings with another dictionary.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open your `__init__.py` file from the `material` folder inside your main working folder.  
<input type="checkbox" class="checkbox inline"> Delete the word `pass` inside the `Material` class.  
<input type="checkbox" class="checkbox inline"> Add the following code to the `Material` class:  

```python
    """Stores a shader program reference, Uniform objects, and OpenGL render settings."""
    def __init__(self, vertex_shader_code, fragment_shader_code):
        self._program_ref = OpenGLUtils.initialize_program(
            vertex_shader_code, 
            fragment_shader_code
        )

        self._uniforms = {}

        # common shader uniforms used in the render process
        self.set_uniform("modelMatrix", None, "mat4")
        self.set_uniform("viewMatrix", None, "mat4")
        self.set_uniform("projectionMatrix", None, "mat4")

        # OpenGL render settings
        self._settings = {"drawStyle": GL.GL_TRIANGLES}

    @property
    def program_ref(self):
        return self._program_ref

    def get_setting(self, setting_name):
        """Return a setting value if the setting exists; otherwise, return None."""
        return self._settings.get(setting_name, None)
```

Subclasses of `Material` will need to provide the vertex and fragment shader code when calling their superclass `__init__` method. Then it is assumed that the program will have `modelMatrix`, `viewMatrix`, and `projectionMatrix` uniform variables defined. We create and store those uniform objects with a method called `set_uniform` which we define below. We also provide a getter for the program reference which is necessary in the `Mesh` class to associate geometric attribute data with program variables and then draw each object.

<input type="checkbox" class="checkbox inline"> Next, add the following code for managing uniforms and settings to the `Material` class:  

```python
    def set_uniform(self, variable_name, data, data_type=None):
        """Set or add a Uniform object representing a property of this material."""
        if variable_name in self._uniforms.keys():
            self._uniforms[variable_name].data = data
        elif data_type is not None:
            self._uniforms[variable_name] = Uniform(data_type, data)
            self._uniforms[variable_name].locate_variable(self._program_ref, 
                                                          variable_name)
        else:
            raise Exception("A new Material property must have a dataType.")

    def upload_data(self):
        """Upload the data of all stored uniform variables."""
        for uniform_obj in self._uniforms.values():
            uniform_obj.upload_data()

    def update_render_settings(self):
        pass

    def set_properties(self, properties):
        """Set multiple uniform and setting values from a dictionary."""
        for name, data in properties.items():
            if name in self._uniforms.keys():
                self._uniforms[name].data = data
            elif name in self._settings:
                self._settings[name] = data
            else:
                raise Exception(f"Material has no property named {name}")
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `set_uniform` method will update the data of an existing variable similar to the `set_attribute` method in the `Geometry` class. The difference here is that uniforms variables must be located in the program before their data can be uploaded, so the `set_uniform` method immediately locates any new `Uniform` object it creates. Then, `Material` provides a separate `upload_data` method for updating all the uniform variable data linked in the program, which we control in the `Mesh` class after setting the appropriate matrix data.

The `update_render_settings` method is empty here so that subclasses of `Material` can override it. The method will check for specific render settings in its `self._settings` dictionary and then call the relevant OpenGL functions based on the setting values.

## Basic Materials

In our hierarchy of materials, the `BasicMaterial` class is a direct child of `Material` and its purpose is to provide the vertex shader code and fragment shader code for rendering points, lines, and surfaces. The shader program will be relatively simple with uniform variables for the projection matrix, view matrix, and model matrix along with attribute variables for the vertex position, and vertex color data. 

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `material` folder, create a new file called `basic_materials.py`.  
<input type="checkbox" class="checkbox inline"> Open `basic_materials.py` for editing and add the following code:  

```python
# material.basic_materials.py
import OpenGL.GL as GL

from material import Material

class BasicMaterial(Material):
    """A simple material for rendering objects in a solid color or vertex colors."""
    def __init__(self):
        vertex_shader_code = """
        uniform mat4 projectionMatrix;
        uniform mat4 viewMatrix;
        uniform mat4 modelMatrix;

        in vec3 vertexPosition;
        in vec3 vertexColor;
        
        out vec3 color;

        void main() {
            gl_Position = projectionMatrix * viewMatrix * modelMatrix * vec4(vertexPosition, 1.0);
            color = vertexColor;
        }
        """

        fragment_shader_code = """
        uniform vec3 baseColor;
        uniform bool useVertexColors;
        
        in vec3 color;
        
        out vec4 fragColor;

        void main() {
            vec4 tempColor = vec4(baseColor, 1.0);
            if (useVertexColors) tempColor *= vec4(color, 1.0);
            fragColor = tempColor;
        }
        """

        super().__init__(vertex_shader_code, fragment_shader_code)

        self.set_uniform("baseColor", (1,1,1), "vec3")
        self.set_uniform("useVertexColors", False, "bool")
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Here we define the program code for the vertex shader and fragment shader. Then we call the superclass `__init__` method. Remember from the `Material` class, this `__init__` method takes the shader source code and initializes the program.

Notice that we provide a default color (white) with the `baseColor` uniform variable if vertex color data is not given. Our applications can set the `useVertexColors` uniform variable to `True` when we want the fragment shader to apply our given vertex color data instead of just using the default `baseColor`. 

Now that the shader program is set, we can create subclasses to handle specific render settings by implementing the `update_render_settings` method.

### Point Material

The first extension of our `BasicMaterial` class will render vertices as disconnected points. It will use setting properties for `drawStyle` and `pointSize` to call the appropriate OpenGL functions.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open `basic_materials.py` for editing and add the following code after the `BasicMaterial` class:  

```python
class PointMaterial(BasicMaterial):
    """Manages render settings for drawing vertices as rounded points.

    drawStyle: GL_POINTS to draw vertices without any connecting lines
    pointSize: 8 pixels default width and height of each point
    roundedPoints: True to render points with smooth corners
    """
    def __init__(self, properties={}):
        super().__init__()

        self._settings["drawStyle"] = GL.GL_POINTS
        self._settings["pointSize"] = 8
        self._settings["roundedPoints"] = True

        self.set_properties(properties)

    def update_render_settings(self):
        GL.glPointSize(self._settings["pointSize"])

        if self._settings["roundedPoints"]:
            GL.glEnable(GL.GL_POINT_SMOOTH)
        else:
            GL.glDisable(GL.GL_POINT_SMOOTH)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `drawStyle` setting is used by the `Mesh` class when it draws itself, but the `update_render_settings` method handles all other OpenGL settings. Here, it sets the point size with [`glPointSize`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glPointSize.xhtml){:target="_blank"} and enables smooth points with [`glEnable`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glEnable.xhtml){:target="_blank"}.

### Line Material

The next extension will draw a few different types of lines. The "connected" type draws lines from through each vertex, from the first one to the last one. The "loop" type will additionally draw a final line from the last vertex back to the first one again, and the "segments" type will draw separate lines between each consecutive pair of vertices.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open `basic_materials.py` for editing and add the following code after the `PointMaterial` class:  

```python
class LineMaterial(BasicMaterial):
    """Manages render settings for drawing lines between vertices.

    drawStyle: GL_LINE_STRIP by default, changes according to lineType
    lineType: "connected" to draw through all vertices from first to last.
    lineType: "loop" to draw through all vertices and connect last to first.
    lineType: "segments" to draw separate lines between each pair of vertices.
    """
    def __init__(self, properties={}):
        super().__init__()

        self._settings["drawStyle"] = GL.GL_LINE_STRIP
        self._settings["lineType"] = "connected"

        self.set_properties(properties)

    def update_render_settings(self):
        if self._settings["lineType"] == "connected":
            self._settings["drawStyle"] = GL.GL_LINE_STRIP
        elif self._settings["lineType"] == "loop":
            self._settings["drawStyle"] = GL.GL_LINE_LOOP
        elif self._settings["lineType"] == "segments":
            self._settings["drawStyle"] = GL.GL_LINES
        else:
            raise Exception("Unknown line type: must be one of [connected | loop | segments].")
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

We use the `lineType` setting to change the `drawStyle` so that the class is easier to use. The values "connected", "loop", and "segments" are easier to understand than the OpenGL draw style constants of `GL_LINE_STRIP`, `GL_LINE_LOOP`, and `GL_LINES`. Additionally, using the `lineType` property allows us to restrict the draw styles that can be used with this class.

### Surface Material

The last extension will draw a flat plane between every three vertices to create a triangular surface. The front side of a surface is the side in which the vertices appear to be listed in counterclockwise order. OpenGL will not render the back side of a surface by default, but we will create a control for that with the "doublSide" setting. Additionally, the "wireframe" setting will provide the option to only render the lines of the surfaces in color without any filling.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open `basic_materials.py` for editing and add the following code after the `LineMaterial` class:  

```python
class SurfaceMaterial(BasicMaterial):
    """Manages render settings for drawing vertices as a colored surface.

    drawStyle: GL_TRIANGLES to draw triangles between sets of 3 vertices
    doubleSide: False to render only the side where the vertices are in counterclockwise order
    wireframe: False to render triangles instead of lines between the vertices
    """
    def __init__(self, properties={}):
        super().__init__()

        self._settings["drawStyle"] = GL.GL_TRIANGLES
        self._settings["doubleSide"] = False
        self._settings["wireframe"] = False

        self.set_properties(properties)

    def update_render_settings(self):
        if self._settings.get("doubleSide", False):
            GL.glDisable(GL.GL_CULL_FACE)
        else:
            GL.glEnable(GL.GL_CULL_FACE)

        if self._settings.get("wireframe", False):
            GL.glPolygonMode(GL.GL_FRONT_AND_BACK, GL.GL_LINE)
        else:
            GL.glPolygonMode(GL.GL_FRONT_AND_BACK, GL.GL_FILL)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Now that we have some geometry and material classes, we have all the information we need to render objects. The last thing required is a class that uses this information to render mesh objects in a scene.

# Rendering Scenes with the Framework

The `Renderer` class is the last component for rendering 3D scenes. It will initialize all the necessary components of the scene including the camera and mesh objects. Then it sets up and manages general tasks for the whole scene such as depth testing, antialiasing, and clearing each frame.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `core` folder, create a new file called `renderer.py`.  
<input type="checkbox" class="checkbox inline"> Open `renderer.py` for editing and add the following code:  

```python
# core.renderer.py
import OpenGL.GL as GL

from core.scene_graph import Mesh, Camera, Scene

class Renderer:
    """Manages the rendering of a given scene with basic OpenGL settings."""
    def __init__(self, clear_color=(0,0,0)):
        """Initialize basic settings for depth testing, antialiasing and clear color."""
        GL.glEnable(GL.GL_DEPTH_TEST)
        GL.glEnable(GL.GL_MULTISAMPLE)
        GL.glClearColor(*clear_color, 1) # unpack clear_color to pass its values separately

    def render(self, scene, camera):
        """Render the given scene as viewed through the given camera."""
        if not isinstance(scene, Scene):
            raise Exception("The given scene must be an instance of Scene.")
        if not isinstance(camera, Camera):
            raise Exception("The given camera must be an instance of Camera.")

        # clear buffers
        GL.glClear(GL.GL_COLOR_BUFFER_BIT | GL.GL_DEPTH_BUFFER_BIT)

        # update camera view
        camera.update_view_matrix()

        # draw all the viewable meshes
        for mesh in scene.descendant_list:
            if isinstance(mesh, Mesh) and mesh.visible:
                mesh.render(camera.view_matrix, camera.projection_matrix)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

A `Renderer` object will enable depth testing and antialiasing when it is created as well as set the background color. Then an application will call `render` with a `Scene` and `Camera` object. Since the `Scene` is the root node of a scene graph, we can get every mesh in the scene with its `descendant_list` property. Then for every visible mesh object, we call its `render` method and pass the camera's view matrix and projection matrix.

Remember, each individual mesh handles the steps for rendering itself. These steps include:
1. Specify the shader program in its material object to be used for rendering.
2. Bind the VAO object that handles its attribute data.
3. Set its model, view, and projection matrices.
4. Upload data to its uniform variables.
5. Update OpenGL render settings.
6. Draw the number of vertices in its goemetry object using the draw style of its material object.

All these steps we already programmed into the `render` method of the `Mesh` class in the previous lesson. (See the final code snippet in [The Scene Graph](/software-engineering-lab/notes/scene_graph/#mesh).)

With the `Renderer` object complete, now we can finally render a 3D scene. Let's make a simple spinning cube in the center of the screen using the `BoxGeometry` and `SurfaceMaterial` classes.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main working folder, create a new file called `test_9_1.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_9_1.py` for editing and add the following code:  

```python
# test_9_1.py
from math import pi

from core.app import WindowApp
from core.renderer import Renderer
from core.scene_graph import Scene, Camera, Mesh
from geometry.basic_geometries import BoxGeometry
from material.basic_materials import SurfaceMaterial

class Test_9_1(WindowApp):
    """Test the basic scene graph elements by rendering a spinning cube."""
    def startup(self):
        print("Starting up Test 9-1...")

        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspect_ratio=800/600)
        self.camera.position = (0, 0, 4)

        geometry = BoxGeometry()
        material = SurfaceMaterial({"useVertexColors": True})
        self.mesh = Mesh(geometry, material)
        self.scene.add(self.mesh)

        self.rotate_speed_Y = 2/3 * pi
        self.rotate_speed_X = 1/3 * pi

    def update(self):
        self.mesh.rotateY(self.rotate_speed_Y * self.delta_time)
        self.mesh.rotateX(self.rotate_speed_X * self.delta_time)

        self.renderer.render(self.scene, self.camera)

# initialize and run this test at 800 x 600 resolution
Test_9_1(screen_size=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_9_1.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a spinning cube with each side a different color in the center of the screen.  

Our test applications have become quite a bit shorter now that we encapsulated a lot of the rendering tasks in scene graph components! 

The `startup` method simply creates a `Renderer`, `Scene`, `Camera` with 4:3 aspect ratio, geometry, and material with different colored vertices. Then it creates a mesh object with the box geometry and surface material before adding the mesh to the scene graph. We also set different rotation speeds for rotating around the $y$-axis and $x$-axis.  

After everything is set up, the `update` method simply applies the rotations to the mesh before rendering the entire scene.

# Extra Components

After learning about how to render shapes with geometry and material objects, we can now think about making special objects that help render a 3D scene. All of our scenes so far have just been shapes floating in a black void. Let's change that by creating a structure to represent the coordinate axes and a grid to help orient the user. After rendering those in a scene, we will then create a special object that handles inputs and allows the user to explore the scene by moving the camera around.

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
<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a new file called `test_9_2.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_9_2.py` for editing and add the following code:  

```python
# test_9_2.py
from math import pi

from core.app import WindowApp
from core.renderer import Renderer
from core.scene_graph import Scene, Camera
from extras.helpers import AxesHelper, GridHelper

class Test_9_2(WindowApp):
    """Test rendering a grid and axes with helper classes."""
    def startup(self):
        print("Starting up Test 9-2...")

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
Test_9_2(screen_size=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_9_2.py` in the terminal.  
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

<input type="checkbox" class="checkbox inline"> Inside your main working folder, create a copy of the file called `test_9_2.py` and change its name to `test_9_3.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_9_3.py` for editing and add the following import statement just before the class definition:

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

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_9_3.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can move the camera in all directions in addition to panning it up and down.  

:congratulations: **Congratulations!** :tada:  
You have now completed enough components to start creating some interesting 3D scenes. What other components can you think of that might provide a benefit to programming interactive 3D applications? Try extending `Geometry` and `Material` classes and using other core components to build something fun or practical.