---
# MARP
theme: default
paginate: true

# Jekyll
title: "Geometry and Material Objects"
date: 2022-06-07
categories:
  - Notes
classes: wide
toc_sticky: false
---

*Chapter 4.3 outlines the `Geometry` class and its extensions which store attribute data related to the vertices of different objects.*  
*Chapter 4.4 outlines the `Material` class and its extensions which store the shader program, uniform data, and rendering settings related to the appearance of different objects.*

In the previous lesson, we looked at the overview of our scene graph framework and created many of the basic components, including `Object3D`, `Scene`, `Group`, `Camera`, and `Mesh`. At the time, we also created an empty `Geometry` class and `Material` class. This lesson covers those two classes in full detail along with some simple extensions of each one.

![](/software-engineering-lab/assets/images/scene_graph_uml-2.png)

The classes we make this time include a `BoxGeometry`, a `BasicMaterial`, and a `SurfaceMaterial` that will allow us to render a 3D box with sides of many colors. The `BasicMaterial` class maintains a reference to a basic shader program that renders vertices with a white base color if colors for specific vertices are not provided. The `BoxGeometry` class will define separate vertex colors so each side of the box will be a different color and we can see the box clearly. Extensions to the `BasicMaterial` class will manage render settings for different draw styles such as drawing points, lines, or surfaces.

Finally, we will create the `Renderer` class that uses `Scene` and `Camera` to draw each `Mesh`. This will be the last component we need to draw a 3D box by combining the `BoxGeometry` and `SurfaceMaterial` in a test application. 

# Geometry Objects

The `Geometry` class contains a dictionary for each `Attribute` of a specified geometric object as well as the number of vertices. It is intended to hold only the common behavior among all geometry objects, so each one of its extensions will define a certain type of geometric object with specific vertex data and attributes.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open your `geometry.py` file from the `geometry` folder inside your main working folder.  
<input type="checkbox" class="checkbox inline"> Delete the word `pass` inside the `Geometry` class.  
<input type="checkbox" class="checkbox inline"> Add the following code to the `Geometry` class:  

```python
    """Store attribute data and their total number of vertices."""
    def  __init__(self):
        self._attributes = {}
        self._vertexCount = None

    @property
    def attributes(self):
        return self._attributes

    @property
    def vertexCount(self):
        if self._vertexCount is None:
            self.countVertices()
        return self._vertexCount

    def setAttribute(self, variableName, data, dataType=None):
        """Add or update an attribute of this geometric object."""
        if variableName in self._attributes.keys():
            self._attributes[variableName].data = data
            self._attributes[variableName].uploadData()
        elif dataType is not None:
            self._attributes[variableName] = Attribute(dataType, data)
        else:
            raise Exception("A new Geometry attribute must have a data type.")
    
    def countVertices(self, variableName=None):
        """Count the number of vertices as the length of an attribute's data."""
        if len(self._attributes) == 0:
            self._vertexCount = 0
        if variableName is not None:
            self._vertexCount = len(self._attributes[variableName].data)
        else:
            self._vertexCount = len(list(self._attributes.values())[0].data)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `setAttribute` method stores an instance of `Attribute` for the specified shader program variable, or updates an existing one. The name of the shader variable is also the key to accessing the `Attribute` in the dictionary. After attributes have been set, we can use the `countVertices` method to get the number of vertices. Here we use the single underscore naming convention with `_attributes` and `_vertexCount` to show that these variables are for internal use to the `Geometry` class. Then, the getter properties will return the variable values for external use. Properties are useful in this way because we can confirm that `self._vertexCount` has a value before returning it.

Now that we have the base `Geometry` defined, we can create specific geometric objects such as rectangles and boxes.

## Rectangles

The first geometric object is a simple 2D rectangle with its origin at the center. The `RectangleGeometry` class will take two parameters&mdash;the width and height as they define two perpendicular sides of the rectangle. Then, with its origin at the center, we can get the $x$ and $y$ coordinates of the four points by halving the height and width values.

![A rectangle is drawn from two triangles defined by its width and height.](/software-engineering-lab/assets/images/rectangle_geometry.png)

Since OpenGL only provides triangles as a way to draw filled-in shapes, our rectangles will be drawn from two right triangles sharing the same hypotenuse. Because of this, we need to define our vertices in a way that draws each triangle as a separate group. That means our position data for each rectangle will have six vertices instead of four. Also, the vertices must be listed in counterclockwise order so that front sides of the triangles face in the positive $z$ direction.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `geometry` folder, create a new file called `rectangle_geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open the `rectangle_geometry.py` file for editing and add the following code:  

```python
from geometry.geometry import Geometry

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

        positionData = (P0,P1,P3, # first triangle
                        P0,P3,P2) # second triangle
        colorData = (C0,C1,C3, # first triangle
                     C0,C3,C2) # second triangle

        self.setAttribute("vertexPosition", positionData, "vec3")
        self.setAttribute("vertexColor", colorData, "vec3")

        self.countVertices()
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
<input type="checkbox" class="checkbox inline"> In the `geometry` folder, create a new file called `box_geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open the `box_geometry.py` file for editing and add the following code:  

```python
from geometry.geometry import Geometry

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

        positionData = (P5,P1,P3, P5,P3,P7, # right side
                        P0,P4,P6, P0,P6,P2, # left side
                        P6,P7,P3, P6,P3,P2, # top side
                        P0,P1,P5, P0,P5,P4, # bottom side
                        P4,P5,P7, P4,P7,P6, # front side
                        P1,P0,P2, P1,P2,P3) # back side
        
        # create a list of 36 RGB vertices
        colorData = C1 + C2 + C3 + C4 + C5 + C6 
        
        self.setAttribute("vertexPosition", positionData, "vec3")
        self.setAttribute("vertexColor", colorData, "vec3")
        self.countVertices()
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Again, the order of the vertex color data will decide the color of each side. The six red vertices are first, so they are assigned to the right side. Then yellow vertices for the left side, green for the top side, cyan for the bottom side, blue for the front, and magenta for the back. These sides will be solid colors, so we could take advantage of the `*` operator for lists and create six copies of each color vertex. If we wanted to see color gradients on each side, we would need to write out each color vertex similar to the way we write out the position vertices. 

# Material Objects

While the `Geometry` objects manage geometric data concerning the shape, position, and color of an object's vertices, the `Material` objects manage data related to rendering the object including the shader program itself, `Uniform` objects, and OpenGL render settings. 

## Base Class

The base class of `Material` will compile and initialize the shader program, store and manage uniform objects in a dictionary, and handle OpenGL-specific settings with another dictionary.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open your `material.py` file from the `material` folder inside your main working folder.  
<input type="checkbox" class="checkbox inline"> Delete the word `pass` inside the `Material` class.  
<input type="checkbox" class="checkbox inline"> Add the following code to the `Material` class:  

```python
    """Stores a shader program reference, Uniform objects, and OpenGL render settings."""
    def __init__(self, vertexShaderCode, fragmentShaderCode):
        self._programRef = OpenGLUtils.initializeProgram(
            vertexShaderCode, 
            fragmentShaderCode
        )

        self._uniforms = {}

        # common shader uniforms used in the render process
        self.setUniform("modelMatrix", None, "mat4")
        self.setUniform("viewMatrix", None, "mat4")
        self.setUniform("projectionMatrix", None, "mat4")

        # OpenGL render settings
        self._settings = {"drawStyle": GL_TRIANGLES}

    @property
    def programRef(self):
        return self._programRef

    def getSetting(self, setting_name):
        """Return a setting value if the setting exists; otherwise, return None."""
        return self._settings.get(setting_name, None)
```

Subclasses of `Material` will need to provide the vertex and fragment shader code when calling their superclass `__init__` method. Then it is assumed that the program will have `modelMatrix`, `viewMatrix`, and `projectionMatrix` uniform variables defined. We create and store those uniform objects with a method called `setUniform` which we define below. We also provide a getter for the program reference which is necessary in the `Mesh` class to associate geometric attribute data with program variables and then draw each object.

<input type="checkbox" class="checkbox inline"> Next, add the following code for managing uniforms and settings to the `Material` class:  

```python
    def setUniform(self, variableName, data, dataType=None):
        """Set or add a Uniform object representing a property of this material."""
        if variableName in self._uniforms.keys():
            self._uniforms[variableName].data = data
        elif dataType is not None:
            self._uniforms[variableName] = Uniform(dataType, data)
            self._uniforms[variableName].locateVariable(self._programRef, 
                                                        variableName)
        else:
            raise Exception("A new Material property must have a dataType.")

    def uploadData(self):
        """Upload the data of all stored uniform variables."""
        for uniformObject in self._uniforms.values():
            uniformObject.uploadData()

    def updateRenderSettings(self):
        pass

    def setProperties(self, properties):
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

The `setUniform` method will update the data of an existing variable similar to the `setAttribute` method in the `Geometry` class. The difference here is that uniforms variables must be located in the program before their data can be uploaded, so the `setUniform` method immediately locates any new `Uniform` object it creates. Then, `Material` provides a separate `uploadData` method for updating all the uniform variable data linked in the program, which we control in the `Mesh` class after setting the appropriate matrix data.

The `updateRenderSettings` method is empty here so that subclasses of `Material` can override it. The method will check for specific render settings in its `self._settings` dictionary and then call the relevant OpenGL functions based on the setting values.

## Basic Materials

In our hierarchy of materials, the `BasicMaterial` class is a direct child of `Material` and its purpose is to provide the vertex shader code and fragment shader code for rendering points, lines, and surfaces. The shader program will be relatively simple with uniform variables for the projection matrix, view matrix, and model matrix along with attribute variables for the vertex position, and vertex color data. 

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `material` folder, create a new file called `basic_material.py`.  
<input type="checkbox" class="checkbox inline"> Open `basic_material.py` for editing and add the following code:  

```python
from material.material import Material

class BasicMaterial(Material):
    
    def __init__(self):
        vertexShaderCode = """
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

        fragmentShaderCode = """
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

        super().__init__(vertexShaderCode, fragmentShaderCode)

        self.setUniform("baseColor", (1,1,1), "vec3")
        self.setUniform("useVertexColors", False, "bool")
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Here we define the program code for the vertex shader and fragment shader. Then we call the superclass `__init__` method. Remember from the `Material` class, this `__init__` method takes the shader source code and initializes the program.

Notice that we provide a default color (white) with the `baseColor` uniform variable if vertex color data is not given. Our applications can set the `useVertexColors` uniform variable to `True` when we want the fragment shader to apply our given vertex color data instead of just using the default `baseColor`. 

Now that the shader program is set, we can create subclasses to handle specific render settings by implementing the `updateRenderSettings` method.

### Point Material

The first extension of our `BasicMaterial` class will render vertices as disconnected points. It will use setting properties for `drawStyle` and `pointSize` to call the appropriate OpenGL functions.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `material` folder, create a new file called `point_material.py`.  
<input type="checkbox" class="checkbox inline"> Open `point_material.py` for editing and add the following code:  

```python
from OpenGL.GL import *

from material.basic_material import BasicMaterial

class PointMaterial(BasicMaterial):
    """Manages render settings for drawing vertices as rounded points.

    drawStyle: GL_POINTS to draw vertices without any connecting lines
    pointSize: 8 pixels default width and height of each point
    roundedPoints: True to render points with smooth corners
    """
    def __init__(self, properties={}):
        super().__init__()

        self._settings["drawStyle"] = GL_POINTS
        self._settings["pointSize"] = 8
        self._settings["roundedPoints"] = True

        self.setProperties(properties)

    def updateRenderSettings(self):
        glPointSize(self._settings["pointSize"])

        if self._settings["roundedPoints"]:
            glEnable(GL_POINT_SMOOTH)
        else:
            glDisable(GL_POINT_SMOOTH)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `drawStyle` setting is used by the `Mesh` class when it draws itself, but the `updateRenderSettings` method handles all other OpenGL settings. Here, it sets the point size with [`glPointSize`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glPointSize.xhtml){:target="_blank"} and enables smooth points with [`glEnable`](https://www.khronos.org/registry/OpenGL-Refpages/gl4/html/glEnable.xhtml){:target="_blank"}.

### Line Material

The next extension will draw a few different types of lines. The "connected" type draws lines from through each vertex, from the first one to the last one. The "loop" type will additionally draw a final line from the last vertex back to the first one again, and the "segments" type will draw separate lines between each consecutive pair of vertices.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `material` folder, create a new file called `line_material.py`.  
<input type="checkbox" class="checkbox inline"> Open `line_material.py` for editing and add the following code:  

```python
from OpenGL.GL import *

from material.basic_material import BasicMaterial

class LineMaterial(BasicMaterial):
    """Manages render settings for drawing lines between vertices.

    drawStyle: GL_LINE_STRIP by default, changes according to lineType
    lineType: "connected" to draw through all vertices from first to last.
    lineType: "loop" to draw through all vertices and connect last to first.
    lineType: "segments" to draw separate lines between each pair of vertices.
    """
    def __init__(self, properties={}):
        super().__init__()

        self._settings["drawStyle"] = GL_LINE_STRIP
        self._settings["lineType"] = "connected"

        self.setProperties(properties)

    def updateRenderSettings(self):
        if self._settings["lineType"] == "connected":
            self._settings["drawStyle"] = GL_LINE_STRIP
        elif self._settings["lineType"] == "loop":
            self._settings["drawStyle"] = GL_LINE_LOOP
        elif self._settings["lineType"] == "segments":
            self._settings["drawStyle"] = GL_LINES
        else:
            raise Exception("Unknown line type: must be one of [connected | loop | segments].")
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

We use the `lineType` setting to change the `drawStyle` so that the class is easier to use. The values "connected", "loop", and "segments" are easier to understand than the OpenGL draw style constants of `GL_LINE_STRIP`, `GL_LINE_LOOP`, and `GL_LINES`. Additionally, using the `lineType` property allows us to restrict the draw styles that can be used with this class.

### Surface Material

The last extension will draw a flat plane between every three vertices to create a triangular surface. The front side of a surface is the side in which the vertices appear to be listed in counterclockwise order. OpenGL will not render the back side of a surface by default, but we will create a control for that with the "doublSide" setting. Additionally, the "wireframe" setting will provide the option to only render the lines of the surfaces in color without any filling.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `material` folder, create a new file called `surface_material.py`.  
<input type="checkbox" class="checkbox inline"> Open `surface_material.py` for editing and add the following code:  

```python
from OpenGL.GL import *

from material.basic_material import BasicMaterial

class SurfaceMaterial(BasicMaterial):
    """Manages render settings for drawing vertices as a colored surface.

    drawStyle: GL_TRIANGLES to draw triangles between sets of 3 vertices
    doubleSide: False to render only the side where the vertices are in counterclockwise order
    wireframe: False to render triangles instead of lines between the vertices
    """
    def __init__(self, properties={}):
        super().__init__()

        self._settings["drawStyle"] = GL_TRIANGLES
        self._settings["doubleSide"] = False
        self._settings["wireframe"] = False

        self.setProperties(properties)

    def updateRenderSettings(self):
        if self._settings.get("doubleSide", False):
            glDisable(GL_CULL_FACE)
        else:
            glEnable(GL_CULL_FACE)

        if self._settings.get("wireframe", False):
            glPolygonMode(GL_FRONT_AND_BACK, GL_LINE)
        else:
            glPolygonMode(GL_FRONT_AND_BACK, GL_FILL)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Now that we have some geometry and material classes, we have all the information we need to render objects. The last thing required is a class that uses this information to render mesh objects in a scene.

# Rendering Scenes with the Framework

The `Renderer` class is the last component for rendering 3D scenes. It will initialize all the necessary components of the scene including the camera and mesh objects. Then it sets up and manages general tasks for the whole scene such as depth testing, antialiasing, and clearing each frame.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In the `core` folder, create a new file called `renderer.py`.  
<input type="checkbox" class="checkbox inline"> Open `renderer.py` for editing and add the following code:  

```python
from OpenGL.GL import *

from core.mesh import Mesh
from core.camera import Camera
from core.scene import Scene

class Renderer():

    def __init__(self, clearColor=(0,0,0)):
        """Initialize basic settings for depth testing, antialiasing and clear color."""
        glEnable(GL_DEPTH_TEST)
        glEnable(GL_MULTISAMPLE)
        glClearColor(*clearColor, 1) # unpack clearColor to pass its values separately

    def render(self, scene, camera):
        """Render the given scene as viewed through the given camera."""
        if not isinstance(scene, Scene):
            raise Exception("The given scene must be an instance of Scene.")
        if not isinstance(camera, Camera):
            raise Exception("The given camera must be an instance of Camera.")

        # clear buffers
        glClear(GL_COLOR_BUFFER_BIT | GL_DEPTH_BUFFER_BIT)

        # update camera view
        camera.updateViewMatrix()

        # draw all the viewable meshes
        for mesh in scene.getDescendantList():
            if isinstance(mesh, Mesh) and mesh.visible:
                mesh.render(camera.view_matrix, camera.projection_matrix)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

A `Renderer` object will enable depth testing and antialiasing when it is created as well as set the background color. Then an application will call `render` with a `Scene` and `Camera` object. Since the `Scene` is the root node of a scene graph, we can get every mesh in the scene with its `getDescendantList` method. Then for every visible mesh object, we call its `render` method and pass the camera's view matrix and projection matrix.

Remember, each individual mesh handles the steps for rendering itself. These steps include:
1. Specify the shader program in its material object to be used for rendering.
2. Bind the VAO object that handles its attribute data.
3. Set its model, view, and projection matrices.
4. Upload data to its uniform variables.
5. Update OpenGL render settings.
6. Draw with the draw style of its material object and vertex count of its geometry object.

All these steps we already programmed into the `render` method of the `Mesh` class in the previous lesson. (See the final code snippet in [The Scene Graph](/software-engineering-lab/notes/scene_graph/#mesh).)

With the `Renderer` object complete, now we can finally render a 3D scene. Let's make a simple spinning cube in the center of the screen using the `BoxGeometry` and `SurfaceMaterial` classes.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main working folder, create a new file called `test_4_1.py`.  
<input type="checkbox" class="checkbox inline"> Open `test_4_1.py` for editing and add the following code:  

```python
from math import pi

from core.base import Base
from core.renderer import Renderer
from core.scene import Scene
from core.camera import Camera
from core.mesh import Mesh
from geometry.box_geometry import BoxGeometry
from material.surface_material import SurfaceMaterial

class Test_4_1(Base):
    """Test the basic scene graph elements by rendering a spinning cube."""
    def initialize(self):
        print("Starting up Test 4-1")

        self.renderer = Renderer()
        self.scene = Scene()
        self.camera = Camera(aspectRatio=800/600)
        self.camera.setPosition((0, 0, 4))

        geometry = BoxGeometry()
        material = SurfaceMaterial({"useVertexColors": True})
        self.mesh = Mesh(geometry, material)
        self.scene.add(self.mesh)

        self.rotate_speed_Y = 2/3 * pi
        self.rotate_speed_X = 1/3 * pi

    def update(self):
        self.mesh.rotateY(self.rotate_speed_Y * self.deltaTime)
        self.mesh.rotateX(self.rotate_speed_X * self.deltaTime)

        self.renderer.render(self.scene, self.camera)

# initialize and run this test at 800 x 600 resolution
Test_4_1(screenSize=(800,600)).run()
```

<input type="checkbox" class="checkbox inline"> Save the file and run it with the command `python test_4_1.py` in the terminal.  
<input type="checkbox" class="checkbox inline"> Confirm that you can see a spinning cube with each side a different color in the center of the screen.  

Our test applications have become quite a bit shorter now that we encapsulated a lot of the rendering tasks in scene graph components! 

The `initialize` method simply creates a `Renderer`, `Scene`, `Camera` with 4:3 aspect ratio, geometry, and material with different colored vertices. Then it creates a mesh object with the box geometry and surface material before adding the mesh to the scene graph. We also set different rotation speeds for rotating around the $y$-axis and $x$-axis.  

After everything is set up, the `update` method simply applies the rotations to the mesh before rendering the entire scene.