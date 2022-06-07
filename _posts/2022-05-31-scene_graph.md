---
# MARP
theme: default
paginate: true

# Jekyll
title: "The Scene Graph"
date: 2022-05-31
categories:
  - Notes
classes: wide
toc_sticky: false
---

*Chapter 4.1 explains the basic concepts for building a 3D scene using the scene graph.*  
*Chapter 4.2 introduces the fundamental components for rendering the scene, 3D objects, and the camera to view it all.*

# The Scene Graph Framework

Now that we have enough core components in our graphics framework, we can begin building 3D scenes. The heart of our framework is the *scene graph* which is a tree-like data structure that connects all the objects in a hierarchical graph. Each object is called a *node* and has its own place in the hierarchy. The top node is called the *root node* and it is the one node from which all other nodes extend. A node that extends from another node is called a *child node* and the node above it is its *parent node*.

For example, if we wanted to model the Sun and planets, the scene graph might look like the image below.

![Objects in a scene graph are arranged relative to other objects and the scene itself.](/software-engineering-lab/assets/images/scene_graph_example.png)

The **scene** itself is the root node and the Sun is its direct child. All the planets revolve around the Sun, so they are children of the Sun node. We can further add moons to the scene graph and their parent nodes would be the planets around which they revolve. In a scene graph, the *ancestor nodes* of a node are everything from the node's parent up to and including the root node. Likewise, the *descendants* of a node are all the nodes that extend from it. In this example, the ancestors of the Moon are the Earth, Sun, and Scene. Meanwhile the descendants of the Sun include all of the planets and moons.

Remember that the transformation of an object is calculated by accumulating all the individual scaling, rotation, and translations applied to the object and stored in its *model matrix*. In a scene graph, we can store the model matrix of a node relative to its parent node. Then, we can calculate the transformation of the object relative to the world as the product of all its ancestor model matrices with its own model matrix. For example, the Moon's model matrix would be a rotation matrix relative to the Earth while the Earth's model matrix would be a rotation matrix relative to the Sun. We can then calculate the world transform of the Moon by multiplying the model matrices of the Moon, Earth, and Sun. How convenient!

We can also **group** together nodes in a scene graph when we want to collectively transform them. For example, a basic table might be a long, flat box for the surface and four tall, narrow boxes for the legs. Then we can arrange these five boxes relative to each other in a "table" group and apply transforms to the table as a whole.

![A table is just a large flat box and four long, narrow boxes](/software-engineering-lab/assets/images/simple_table_3d.png)

Now let's take a look at the classes we will use to construct our scene graph!

## Overview of Class Structure

The base class for every node will be called `Object3D` as it represents an object in three-dimensional space. The class will hold the object's model matrix (as `transform`), a reference to its `parent` object, and a list of references to `children` objects.

![A class diagram showing all the components of our scene graph framework](/software-engineering-lab/assets/images/scene_graph_uml.png)

Then, the `Scene`, `Camera`, `Group`, and `Mesh` classes all extend the `Object3D` class. While `Scene` will enforce a no-parent rule, the `Group` class has no additional functionality compared to `Object3D` and we only create it to use its name.

The `Mesh` class represents objects that can be rendered. It will keep information about its vertices and other geometric attributes as well as uniform and OpenGL setting data related to its own appearance. The `Geometry` class will manage a mesh's vertex data in `Attribute` objects which we have already been using for the past few lessons. On the other hand, a `Material` class will manage `Uniform` objects and OpenGL render settings for the mesh. In a later lesson, we will create a `BoxGeometry` class and a `SurfaceMaterial` class from these base classes to render a spinning cube and demonstrate how all these things work together.

Finally, the `Renderer` class puts together the `Scene`, the `Camera`, and all the `Mesh` objects to render the final images. In addition to rendering each mesh in the scene through the camera, it also manages OpenGL settings for depth testing, antialiasing, and the clear color.

Now let's start coding these classes!

# 3D Objects

The `Object3D` class makes the tree graph structure possible. As a node in the graph, it must have the ability to add and remove other nodes as children. At the same time it will manage a one-parent-only rule when its own parent is assigned.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, create a new file called `object3d.py`.  
<input type="checkbox" class="checkbox inline"> Open `object3d.py` for editing and add the following code:  

```python
from core.matrix import Matrix

class Object3D(object):
    """A node in the scene graph tree structure."""
    def __init__(self):
        self.__transform = Matrix.makeIdentity()
        self.__parent = None
        self.__children = []

    @property
    def parent(self):
        return self.__parent

    @parent.setter
    def parent(self, node):
        """Sets or removes the parent of this node."""
        if self.__parent is not None and node is not None:
            raise Exception("Cannot add a child of another node.")
        self.__parent = node


    def add(self, child):
        child.parent = self
        self.__children.append(child)

    def remove(self, child):
        self.__children.remove(child)
        child.parent = None
```

In order for our scene graph to function properly, each node can only have a single parent node. We enforce this rule by creating a **setter** for the `parent` property which will run whenever a program tries to assign a new value to the `parent` property. We can remove the `parent` by calling this setter with the `None` value for the new parent. Otherwise, if `parent` already has a value and the new value is not `None`, we raise an exception to explain the problem.

<input type="checkbox" class="checkbox inline"> Add the next code to `object3d.py` for getting the world transform and all descendants of an object.  

```python
    def getWorldMatrix(self):
        """Calculate the transformation of this node relative to the root node."""
        if self.__parent == None:
            return self.__transform
        else:
            # recursion!
            return self.__parent.getWorldMatrix() @ self.__transform

    def getDescendantList(self):
        """Get a single list containing all the descendants of this node."""
        # more recursion!
        descendants = [self]
        for c in self.__children:
            descendants += c.getDescendantList()
        return descendants
```

The `getWorldMatrix()` method will recursively apply its transformation matrix to each of its ancestors' matrices until it reaches the root node (which has no parent) where it returns the final product. The `getDescendantList()` method similarly uses recursion to build a list of each node below this one in the graph.

<input type="checkbox" class="checkbox inline"> Next, add code to `object3d.py` for applying both local and global geometric transformations.  

```python
    def applyMatrix(self, matrix, localCoord=True):
        """Apply a geometric transformation to this object."""
        if localCoord:
            self.__transform = self.__transform @ matrix
        else:
            self.__transform = matrix @ self.__transform

    def translate(self, x, y, z, localCoord=True):
        """Calculate and apply a translation to this object."""
        m = Matrix.makeTranslation(x,y,z)
        self.applyMatrix(m, localCoord)
    
    def rotateX(self, angle, localCoord=True):
        """Calculate and apply a rotation around the x-axis of this object."""
        m = Matrix.makeRotationX(angle)
        self.applyMatrix(m, localCoord)
    
    def rotateY(self, angle, localCoord=True):
        """Calculate and apply a rotation around the y-axis of this object."""
        m = Matrix.makeRotationY(angle)
        self.applyMatrix(m, localCoord)
    
    def rotateZ(self, angle, localCoord=True):
        """Calculate and apply a rotation around the z-axis of this object."""
        m = Matrix.makeRotationZ(angle)
        self.applyMatrix(m, localCoord)
    
    def scaleUniform(self, s, localCoord=True):
        """Calculate and apply a scaling transformation to this object."""
        m = Matrix.makeScale(s, s, s)
        self.applyMatrix(m, localCoord)
```

First, we make a generic `applyMatrix()` method which accepts a transformation matrix as its first parameter and an optional Boolean as its second parameter. A `True` value for the Boolean indicates that the transformation is local to the object's coordinate axes. Otherwise, we apply the matrix as a global transformation. Then, each method after `applyMatrix()` generates and applies a specific type of transformation matrix to this object.

<input type="checkbox" class="checkbox inline"> Finally, add code to `object3d.py` for getting and setting the position of an object.  

```python
    def getPosition(self):
        """Get the position of this object relative to its parent node."""
        return [self.__transform.item((0,3)),
                self.__transform.item((1,3)),
                self.__transform.item((2,3))]

    def getWorldPosition(self):
        """Get the position of this object relative to the root node."""
        world_transform = self.getWorldMatrix()
        return [world_transform.item((0,3)),
                world_transform.item((1,3)),
                world_transform.item((2,3))]

    def setPosition(self, position):
        """Set the position of this object relative to its parent node."""
        if not type(position) in (list,tuple) or len(position) != 3:
            raise Exception("Object3D position must be in the form (x,y,z).")

        self.__transform.itemset((0,3), position[0])
        self.__transform.itemset((1,3), position[1])
        self.__transform.itemset((2,3), position[2])
```
<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Here, the `getPosition()` and `getWorldPosition()` methods both use a `numpy.ndarray` method called [`item()`](https://numpy.org/doc/stable/reference/generated/numpy.ndarray.item.html). This method returns the scalar value from inside the NDArray matrix at the given location. Remember, we use a 4x4 matrix to represent 3D transformations and the numbers in the last column (at index 3) are the translation values for $x$, $y$, and $z$. So, we just extract the values from the last column and return them as the object's position. 

The `setPosition()` method uses another `numpy.ndarray` method called [`itemset()`](https://numpy.org/doc/stable/reference/generated/numpy.ndarray.itemset.html) which inserts values into a NDArray matrix. In this way, we can directly set the position of an object without applying a translation matrix.

## Scene and Group

The `Scene` and `Group` classes have little to no additional functionality, but they are conceptually important to our scene graph structure. The `Scene` class is the root node of the scene graph and so it will not accept any node as a parent. We will use the `Group` class when we want to apply transformations to many objects together as a whole. For example, a bundle of balloons may not have a visible parent object even though they move together. We could create an instance of `Group` and then add each balloon to it as a child.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, create a new file called `scene.py`.  
<input type="checkbox" class="checkbox inline"> Open `scene.py` for editing and add the following code:  

```python
from core.object3d import Object3D

class Scene(Object3D):
    """Represents the root node of the scene graph tree structure."""
    def __init__(self):
        super().__init__()

    @Object3D.parent.setter
    def parent(self, node):
        if node is not None:
            raise Exception("The root node cannot have a parent.")
```
<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

This time, we use a special decorator (`@Object3D.parent.setter`) to access the setter for the `parent` property in the `Object3D` superclass. Then we can raise an exception if a program tries to assign a parent to the scene node.

<input type="checkbox" class="checkbox inline"> Next, create a create a new file called `group.py` also in the `core` folder.  
<input type="checkbox" class="checkbox inline"> Open `group.py` for editing and add the following code:  

```python
from core.object3d import Object3D

class Group(Object3D):
    """A node that can serve as a base for transforming many attached nodes."""
    def __init__(self):
        super().__init__()
```
<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

## Camera

In the real world, a camera is a physical object with its own position and orientation relative to the world around it. However, in 3D CG this relationship between the camera and the world is a little different. The camera itself is not rendered, but instead it represents the perspective of the viewer. From the viewer's perspective, camera movements can appear as if the entire world is moving and the viewer is staying still. Indeed, since the computer screen doesn't move, we can think of camera movements as inverted global transformations applied to the entire scene. 

For example, if the camera moves two units to the right, all of the objects in the world appear to move two units to the left. Or when the camera rotates 90° clockwise, the rest of the world appears to rotate 90° counterclockwise.

So, we apply the position and orientation of the camera as the inverse of the camera's global transform and store it in a *view matrix*. In addition to the view matrix, we will also manage the projection matrix for the scene with our newly created `Camera` class.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your `core` folder, create a new file called `camera.py`.  
<input type="checkbox" class="checkbox inline"> Open `camera.py` for editing and add the following code:  

```python
from numpy.linalg import inv

from core.object3d import object3d
from core.matrix import Matrix

class Camera(Object3D):
    """Represents the virtual camera used to view the scene."""
    def __init__(self, angleOfView=60, aspectRatio=1, near=0.1, far=1000):
        super().__init__()
        self.__projection_matrix = Matrix.makePerspective(angleOfView, 
                                    aspectRatio, near, far)
        self.__view_matrix = Matrix.makeIdentity()

    @property
    def projection_matrix(self):
        return self.__projection_matrix

    @property
    def view_matrix(self):
        return self.__view_matrix

    def updateViewMatrix(self):
        self.__view_matrix = inv(self.getWorldMatrix())
```
<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

## Mesh

The `Mesh` class represents objects that can be rendered in the scene. It stores vertex data in another class called `Geometry` and appearance data in another class called `Material`. We will complete those two classes in a later lesson, so for now let's make empty classes to avoid interpreter errors.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> In your main folder, create a new folder called `geometry`.  
<input type="checkbox" class="checkbox inline"> In the `geometry` folder, create a new file called `geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open `geometry.py` for editing and add the following code:  

```python
from core.attribute import Attribute

class Geometry(object):
    pass
```
<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  
<input type="checkbox" class="checkbox inline"> In your main folder, create a new folder called `material`.  
<input type="checkbox" class="checkbox inline"> In the `material` folder, create a new file called `material.py`.  
<input type="checkbox" class="checkbox inline"> Open `material.py` for editing and add the following code:  

```python
from OpenGL.GL import *

from core.openGLUtils import OpenGLUtils
from core.uniform import Uniform

class Material(object):
    pass
```
<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Now we can create `Mesh` with `Geometry` and `Material` even though they are not yet finished.

<input type="checkbox" class="checkbox inline"> In your `core` folder, create a new file called `mesh.py`.  
<input type="checkbox" class="checkbox inline"> Open `mesh.py` for editing and add the following code:  

```python
from OpenGL.GL import *

from core.object3d import Object3D
from geometry.geometry import Geometry
from material.material import Material

class Mesh(Object3D):
    """A visible object in the scene with geometric and appearance data.

    Mesh contains data geometric data related to the object vertices and material data about its appearance.
    It also creates and stores a vertex array object (VAO) reference, and associates variables between
    vertex buffers and shader varriables."""
    def __init__(self, geometry, material):
        super().__init__()

        if not isinstance(geometry, Geometry):
            raise Exception(f"Expecting an instance of Geometry but got {type(geometry)} instead.")
        self.__geometry = geometry
        
        if not isinstance(material, Material):
            raise Exception(f"Expecting an instance of Material but got {type(material)} instead.")
        self.__material = material

        self.__visible = True

        self.__vaoRef = glGenVertexArrays(1)
        glBindVertexArray(self.__vaoRef)

        # associate geometry attributes to the material shader program
        for variable, attribute in geometry.attributes.items():
            attribute.associateVariable(material.programRef, variable)

        # unbind this vertex array object
        glBindVertexArray(0)
        
    @property
    def visible(self):
        return self.__visible
    
    @visible.setter
    def visible(self, value):
        self.__visible = bool(value)
```

Since this class very heavily relies on the interfaces we will create in `Geometry` and `Material`, we use the `__init__()` method to make sure that we get instances of those two classes. We also make a `visible` property so we can easily indicate that this object should be rendered. 

The shader program will be stored in `Material` but the vertex attributes are stored in `Geometry`. The `Mesh` class manages both for this object, so naturally we should also maintain the vertex array object (VAO) that associates attribute data to shader program variables. With the `self.__vaoRef` property, we can easily keep track of which VAO binds to which 3D object and that makes it a lot easier to draw multiple shapes on the screen.

<input type="checkbox" class="checkbox inline"> Next, add code to `mesh.py` for drawing the 3D object.  

```python
    def render(self, view_matrix, projection_matrix):
        glUseProgram(self.__material.programRef)
            
        glBindVertexArray(self.__vaoRef)

        # update matrix uniforms
        self.__material.setUniform("modelMatrix", self.getWorldMatrix())        
        self.__material.setUniform("viewMatrix", view_matrix)
        self.__material.setUniform("projectionMatrix", projection_matrix)

        # update the stored data and settings before drawing
        self.__material.uploadData()
        self.__material.updateRenderSettings()
        glDrawArrays(self.__material.getSetting("drawStyle"), 0, 
                     self.__geometry.vertexCount)

        glBindVertexArray(0)
```
<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `render()` method will apply the model matrix, view matrix, and projection matrix before updating OpenGL render settings and drawing the object. It relies heavily on the interfaces of `Material` and `Geometry` which we will complete next time. For now, we can see that `Mesh` controls the VAO and render process for drawing this object while `Material` handles the shader program, uniform matrix variables, and OpenGL render settings such as draw style. 

There is still much more we need to add before we can render a 3D object, so look forward to it next time!