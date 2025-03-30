---
# MARP
theme: default
paginate: true

# Jekyll
title: "8_The Scene Graph"
date: 2025-05-28
categories:
  - Notes
classes: wide
toc_sticky: false
---

*This lesson explains the scene graph which is a useful structure for 3D scenes that contain many objects and relationships. 
In the [3D Objects](#3d-objects) section we introduce fundamental components for the CG framework to render a scene, 3D objects in the scene, and a camera to view it all.*

# The Scene Graph Framework

Now that we have enough core components in our graphics framework, we can begin building 3D scenes. 
The leading star of our framework is the *scene graph* which is a tree-like data structure that connects all the objects in a hierarchical graph. 
Each object is a *node* in the graph and has its own place in the hierarchy. 
The top node from which all other nodes extend is called the *root node*. 
A node that extends from another node is called a *child node* and the node above it is its *parent node*.

As an example, consider modelling the planets of our solar system revolving around the Sun. 
The scene graph might look something like this:

![Objects in a scene graph are arranged relative to other objects and the scene itself.](/software-engineering-lab/assets/images/scene_graph_example.png)

The **scene** itself is the root node and the Sun, which doesn't move, is its direct child. 
All the planets then are children of the Sun node. 
We can add further celestial bodies to their respective positions in the scene graph. 
For example, moons would be children of the planets around which they revolve. 
In a scene graph, *ancestor nodes* include everything from of a given node's parent node up to the root node. 
Likewise, the *descendants* of a node are all the nodes that extend from it. 
In this example, the ancestors of the Moon are the Earth, Sun, and Scene. 
Meanwhile the descendants of the Sun include all of the planets and moons.

As mentioned in the lesson on [Local Transformations](https://robsonger.dev/software-engineering-lab/notes/geometric_transformations/#local-transformations),  the global transformation of an object is calculated by accumulating all the individual scaling, rotation, and translations applied to the object and stored in its *model matrix*. 
In a scene graph, we can store the model matrix of a node relative to its parent node. 
Then, we can calculate the global transformation of the object as the product of its own model matrix with the model matrices of all its ancestors. 
For example, the Moon's model matrix would be a translation and rotation matrix relative to the Earth while the Earth's model matrix would be a translation and rotation matrix relative to the Sun. 
We can then calculate the world transform of the Moon by recursively combining the model matrices of the Moon, Earth, and Sun. 
How convenient!

We can also **group** together nodes in a scene graph when we want to collectively transform them. 
For example, a basic table might be a long, flat box for the surface and four tall, narrow boxes for the legs. 
Then we can arrange these five boxes relative to each other in a "table" group and apply transformations to the table as a whole.

![A table is just a large flat box and four long, narrow boxes](/software-engineering-lab/assets/images/simple_table_3d.png)

Now let's take a look at the classes we will use to construct our scene graph.

## Overview of Class Structure

The base class for every node will be called `Object3D` since it represents any object in three-dimensional space. The class will hold the object's model matrix (as `transform`), a reference to its `parent` object, and a list of references to `children` objects.

![A class diagram showing all the components of our scene graph framework](/software-engineering-lab/assets/images/scene_graph_uml.png)

Then, the `Scene`, `Camera`, `Group`, and `Mesh` classes all extend the `Object3D` class. 
The `Scene` class will enforce a no-parent rule so it will always be the root node of the scene graph. 
The `Camera` class stores a projection matrix and a view matrix which are used to transform world objects based on the position and direction of the viewer. 
The `Group` class actually has no additional functionality compared to `Object3D`. 
We only create it so we can have a semantic parent of a collection of nodes. 

The `Mesh` class represents objects that can be rendered. 
It will store geometric data about its vertices, appearance data such as its shader program, and a vertex array object that binds its geometric data to its shader program. 
Later we will implement the `Geometry` class which manages a mesh's vertex data in `Attribute` objects. 
Another class called `Material` will manage `Uniform` objects and OpenGL render settings for the mesh. 
In the next lesson, we will create a `BoxGeometry` class and a `SurfaceMaterial` class from these base classes to render a spinning cube and demonstrate how all these things work together.

Finally, the `Renderer` class applies the matrices from the `Camera` to every `Mesh` object in the `Scene` to render the final image. 
In addition to rendering every viewable mesh in the scene through the camera, `Renderer` also manages OpenGL settings for depth testing, antialiasing, and the clear color.

Now let's start coding these classes!

# 3D Objects

The `Object3D` class implements the logic for a node in a tree graph structure. 
To this end, it must have the ability to add and remove other nodes as children. 
It will also enforce a rule for having only a single parent in its lifetime whenever its parent is assigned.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, create a new file called `scene_graph.py`.  
<input type="checkbox" class="checkbox inline"> Open `scene_graph.py` for editing and add the following code:  

```python
# graphics/core/scene_graph.py
from core.matrix import Matrix

class Object3D:
    """ A node in the scene graph tree structure """
    def __init__(self):
        self._transform = Matrix.identity()
        self._parent = None
        self._children = []

    @property
    def parent(self):
        return self._parent

    @parent.setter
    def parent(self, node):
        """ Set or remove the parent of this node """
        if not isinstance(node, Object3D):
            raise RuntimeError("Parent node must be an instance of Object3D.")
        if self._parent is not None and node is not None:
            raise RuntimeError("Cannot replace an existing parent.")
        self._parent = node

    def add(self, child):
        """ Add an object as the child to this object in the scene graph """
        if not isinstance(child, Object3D):
            raise RuntimeError("Child node must be an instance of Object3D.")
        child.parent = self
        self._children.append(child)

    def remove(self, child):
        """ Remove a child object from this object """
        self._children.remove(child)
        child.parent = None
```

In order for our scene graph to function properly, each node can only have a single parent node. 
We enforce this rule by creating a **setter** for the `parent` property which will run whenever a program tries to assign a new value to the `parent` property. 
We can remove the `parent` by calling this setter with the `None` value for the new parent. 
Otherwise, if `parent` already has a value and the new value is not `None`, we raise an exception to explain the problem.

<input type="checkbox" class="checkbox inline"> Add the next code to the `Object3D` class for accessing an object's global transformation matrix and all its descendants.  

```python
    @property
    def world_matrix(self):
        """ Calculate the transformation of this node relative to the root node """
        if self._parent is None:
            return self._transform
        else:
            # recursion!
            return self._parent.world_matrix @ self._transform

    @property
    def descendant_list(self):
        """ Get a single list containing all the descendants of this node """
        descendants = [self]
        for child in self._children:
            # more recursion!
            descendants += child.descendant_list
        return descendants
```

The `world_matrix` property will recursively apply its transformation matrix to each of its ancestors' matrices until it reaches a node with no more ancestors (the root node). 
At that point it returns the final product. 
The `descendant_list` property similarly uses recursion to build a list of each node below this one in the graph.

<input type="checkbox" class="checkbox inline"> Next, add this code to the `Object3D` class for applying both local and global transformations.  

```python
    def apply_matrix(self, matrix, local=True):
        """ Apply a geometric transformation to this object """
        if local:
            self._transform = self._transform @ matrix
        else:
            self._transform = matrix @ self._transform

    def translate(self, x, y, z, local=True):
        """ Calculate and apply a translation to this object """
        m = Matrix.translation(x,y,z)
        self.apply_matrix(m, local)
    
    def rotate_x(self, angle, local=True):
        """ Calculate and apply a rotation around the x-axis of this object """
        m = Matrix.rotation_x(angle)
        self.apply_matrix(m, local)
    
    def rotate_y(self, angle, local=True):
        """ Calculate and apply a rotation around the y-axis of this object """
        m = Matrix.rotation_y(angle)
        self.apply_matrix(m, local)
    
    def rotate_z(self, angle, local=True):
        """ Calculate and apply a rotation around the z-axis of this object """
        m = Matrix.rotation_z(angle)
        self.apply_matrix(m, local)
    
    def scale_uniform(self, s, local=True):
        """ Calculate and apply a scaling transformation to this object """
        m = Matrix.scale(s, s, s)
        self.apply_matrix(m, local)
```

First, we make a generic `apply_matrix` method which accepts a transformation matrix as its first parameter and an optional Boolean as its second parameter. 
A `True` value for the Boolean indicates that the transformation is local to the object's coordinate axes. 
Otherwise, we apply the matrix as a global transformation. 
Then, each method after that uses `apply_matrix` to apply a specific type of transformation matrix to this object.

<input type="checkbox" class="checkbox inline"> Finally, add this code to the `Object3D` class for getting and setting the position of an object.  

```python
    @property
    def position(self):
        """ Get the position of this object relative to its parent node """
        return [self._transform.item((0,3)),
                self._transform.item((1,3)),
                self._transform.item((2,3))]

    @position.setter
    def position(self, position):
        """ Set the position of this object relative to its parent node """
        if not isinstance(position, (list, tuple)) or len(position) != 3:
            raise ValueError("Object3D position must be a list or tuple in the form (x,y,z).")

        self._transform.itemset((0,3), position[0])
        self._transform.itemset((1,3), position[1])
        self._transform.itemset((2,3), position[2])

    @property
    def world_position(self):
        """ Get the position of this object relative to the root node """
        return [self.world_matrix.item((0,3)),
                self.world_matrix.item((1,3)),
                self.world_matrix.item((2,3))]
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Here, the `position` and `world_position` properties both use a `numpy.ndarray` method called [`item()`](https://numpy.org/doc/stable/reference/generated/numpy.ndarray.item.html). 
This method returns the value inside the NDArray matrix at the given location. 
Remember, we use a 4x4 matrix to represent 3D transformations and the numbers in the last column (index 3) are the translation values for $x$, $y$, and $z$.
So, we just extract the values from the last column and return them as the object's position. 

The `position` setter uses another `numpy.ndarray` method called [`itemset()`](https://numpy.org/doc/stable/reference/generated/numpy.ndarray.itemset.html) which inserts values into a NDArray matrix. 
In this way, we can directly set the position of an object without applying a translation matrix.

## Scene and Group

The `Scene` and `Group` classes have very little additional functionality, but they are conceptually important to our scene graph structure. 
The `Scene` class is the root node of the scene graph, so it will not accept any node as a parent.
We will use the `Group` class to store transformations that we want to apply to many objects together as a whole. For example, a bundle of balloons would not have a visible parent object but each balloon would need the same transformation matrix when the bundle moves. We could apply the translation matrix to each balloon individually, or we could apply the matrix to an instance of `Group` that has each balloon as a child.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open the `scene_graph.py` file in your `core` folder and go to the end of the file.  
<input type="checkbox" class="checkbox inline"> Add the following code after the last code of the `Object3D` class:  

```python
class Scene(Object3D):
    """ Represents the root node of the scene graph tree structure """
    def __init__(self):
        super().__init__()

    @Object3D.parent.setter
    def parent(self, node):
        if node is not None:
            raise RuntimeError("The root node cannot have a parent.")
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

This time, we use a special decorator (`@Object3D.parent.setter`) to override the setter for the `parent` property in the `Object3D` superclass.
Then we raise an exception if a program tries to assign a parent node to the scene node.

<input type="checkbox" class="checkbox inline"> Add the following code after the last code of the `Scene` class:  

```python
class Group(Object3D):
    """ A node that represents a base for transforming a collection of child nodes """
    def __init__(self):
        super().__init__()
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

## Camera

In the real world, a camera is a physical object with a position and orientation that change relative to the world around it. 
In 3D CG the relationship between the camera and the world is a little different. 
The camera itself is not rendered, but instead represents the perspective of the viewer. 
From the viewer's perspective, camera movements appear as if the entire world is moving and the viewer is staying still. 
Indeed, since the computer screen doesn't move, we can think of camera movements as global transformations applied to the entire scene. 

But if camera movements apply to the scene instead of the camera itself, that means we need to handle the camera's global transformations a little differently. 
For example, if the camera moves two units to the right, all of the objects in the world appear to move two units to the *left*. Or when the camera rotates 90° clockwise, the rest of the world appears to rotate 90° *counter*clockwise. The scene's movements are the inverse of the camera's movements!

To that end, we will store the position and orientation of the camera separately as a *view matrix* which we calculate to be the inverse of the camera's global transform matrix. 
In addition to the view matrix, we will also use the `Camera` class to manage the projection matrix for the scene.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Open `scene_graph.py` and add the following code to the imports at the top of the file:  

```python
from numpy.linalg import inv
```

<input type="checkbox" class="checkbox inline"> Next, scroll to the bottom of `scene_graph.py` and add the following code after the last code of the `Group` class.  

```python
class Camera(Object3D):
    """ Represents the virtual camera used to view the scene """
    def __init__(self, angle_of_view=60, aspect_ratio=1, near=0.1, far=1000):
        super().__init__()
        self._projection_matrix = Matrix.perspective(
            angle_of_view, aspect_ratio, near, far
        )
        self._view_matrix = Matrix.identity()

    @property
    def projection_matrix(self):
        return self._projection_matrix

    @property
    def view_matrix(self):
        self._view_matrix = inv(self.world_matrix)
        return self._view_matrix
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

## Mesh

The `Mesh` class represents objects that can be rendered in the scene. It will combine two essential classes for managing rendered objects: the `Geometry` that handles vertex data and the `Material` class that handles appearance data. We will complete those two classes in the next lesson, but for now we can make empty classes to avoid errors.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `graphics` folder, create a new folder called `geometries`.  
<input type="checkbox" class="checkbox inline"> In the `geometries` folder, create two new files called `__init__.py` and `geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open `__init__.py` and add the following code:  

```python
# graphics/geometries/__init__.py
from .geometry import *
```

This will allow us to import our `Geometry` class directly from the `geometries` module without needing to know the specific sub-module. 
Over the next few lessons we will create several kinds of geometries in different sub-modules, so this shortcut will be convenient for programming our apps.  

<input type="checkbox" class="checkbox inline"> Open `geometry.py` and add the following code:  

```python
# graphics/geometries/geometry.py
from core.openGL import Attribute

class Geometry(object):
    """ Manages vertex data and attribute variables """
    pass
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  
<input type="checkbox" class="checkbox inline"> In your `graphics` folder, create a new folder called `materials`.  
<input type="checkbox" class="checkbox inline"> In the `materials` folder, create two new files called `__init__.py` and `material.py`.  
<input type="checkbox" class="checkbox inline"> Open `__init__.py` and add the following code:  

```python
# graphics/materials/__init__.py
from .material import *
```

<input type="checkbox" class="checkbox inline"> Open `material.py` and add the following code:  

```python
# graphics/materials/material.py
import OpenGL.GL as GL

from core.openGLUtils import initialize_program
from core.openGL import Uniform

class Material(object):
    """ Manages shader program references, uniform variables, and OpenGL render settings """
    pass
```
<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Now we can create the `Mesh` class with `Geometry` and `Material` even though they are not fully implemented.

<input type="checkbox" class="checkbox inline"> Open the `scene_graph.py` file again from inside the `core` folder.  
<input type="checkbox" class="checkbox inline"> At the top of the `scene_graph.py` file, add the following code to the import statements:  

```python
import OpenGL.GL as GL

from geometries import Geometry
from materials import Material
```

<input type="checkbox" class="checkbox inline"> At the end of the `scene_graph.py` file, add the following code after the last code of the `Camera` class:  

```python
class Mesh(Object3D):
    """ A visible object in the scene with geometry and appearance data """
    def __init__(self, geometry: Geometry, material: Material):
        super().__init__()

        if not isinstance(geometry, Geometry):
            raise ValueError(f"Expected an instance of Geometry but got {type(geometry)} instead.")
        self._geometry = geometry

        if not isinstance(material, Material):
            raise ValueError(f"Expected an instance of Material but got {type(material)} instead.")
        self._material = material

        self._visible = True

        self._vao_ref = GL.glGenVertexArrays(1)
        GL.glBindVertexArray(self._vao_ref)

        # associate geometry attributes to the material's shader program
        for variable, attribute in geometry.attributes.items():
            attribute.associate_variable(material.program_ref, variable)

        # unbind this vertex array object
        GL.glBindVertexArray(0)

    @property
    def visible(self):
        return self._visible

    @visible.setter
    def visible(self, value):
        self._visible = bool(value)
```

Since this class heavily relies on the interfaces we will create in `Geometry` and `Material`, we use the `__init__()` method to make sure that we get instances of those two classes. 
By defining the parameters as `(geometry: Geometry, material: Material)`, we are using [*type hints*](https://pymbook.readthedocs.io/en/latest/typehinting.html) to let other developers know what kind of objects we expect for input.
We also make a `visible` property so we can easily indicate that this object should be rendered. 

The shader program will be stored in `Material` but the vertex attributes are stored in `Geometry` and the `Mesh` class manages the connection between the two. 
So naturally it should also maintain the vertex array object (VAO) that associates attribute data to shader program variables. 
With the `self._vao_ref` property, we can easily keep track of which VAO has the bindings for to which 3D object and it becomes a lot easier to draw multiple different shapes on the screen.

<input type="checkbox" class="checkbox inline"> Next, add the following code for rendering the 3D object to the `Mesh` class.  

```python
    def render(self, view_matrix, projection_matrix):
        """ Updates data and settings before drawing this 3D object """
        GL.glUseProgram(self._material.program_ref)
            
        GL.glBindVertexArray(self._vao_ref)

        # update matrix uniforms
        self._material.set_uniform("modelMatrix", self.world_matrix)
        self._material.set_uniform("viewMatrix", view_matrix)
        self._material.set_uniform("projectionMatrix", projection_matrix)

        # update the stored data and OpenGL settings before drawing
        self._material.upload_data()
        self._material.update_render_settings()
        GL.glDrawArrays(self._material.get_setting("drawStyle"), 0, 
                     self._geometry.vertex_count)

        GL.glBindVertexArray(0)
```
<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `render()` method will apply the model matrix, view matrix, and projection matrix before updating OpenGL render settings and drawing the object. It uses properties and methods from the `Material` and `Geometry` classes which we will complete next time. For now, we can see that `Mesh` controls the VAO and render process for drawing this object while `Material` handles the shader program, matrices as uniform variables, and OpenGL render settings such as draw style. 

There is still much more we need to add next time before we can render a 3D object, so look forward to it!  
