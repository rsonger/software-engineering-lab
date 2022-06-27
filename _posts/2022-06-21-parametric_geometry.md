---
# MARP
theme: default
paginate: true

# Jekyll
title: "Polygons, Planes, Spheres, and Cylinders"
date: 2022-06-21
categories:
  - Notes
classes: wide
toc_sticky: false
---

*This post adds more tools to our geometry toolset with the introduction of polygons, a  parametric geometry base class, and extensions for polygons, planes, ellipsoids, and cylindrical geometries.*  

As we build 3D scenes, we will need to use a variety of different shapes and geometries. Up to now, we have created classes for rectangles and boxes in addition to extending the generic `Geometry` class to create custom shapes. However, there are numerous other shapes that would be helpful for building objects in a 3D scene, such as polygons, spheres, ellipsoids, cylinders, cones, prisms, and pyramids. This lesson introduces a `PolygonGeometry` class that can render shapes with any number of sides of equal lengths. Then, we create a `ParametricGeometry` class which can render 3D surfaces in segments calculated from a given function. The `ParametricGeometry` class provides a good foundation for creating ellipsoids, spheres, cylinders, prisms, pyraminds, and cones. We extend the `ParametricGeometry` class for each new type of geometry based on the function that calculates the points of its surface.

# Polygons

Our `PolygonGeometry` class will provide the ability to render *regular polygons* which are 2D shapes where all sides and angle are equal. Regular polygons include equilateral triangles (3 sides), squares (4 sides), pentagons (5 sides), hexagons (6 sides), heptagons (7 sides), octogons (8 sides), and so on.

We can calculate the points of a polygon with radius $r$ as equally spaced points along the circumference of a circle with the same radius $r$. Recall that the parametric equations for the circumference of a circle are $x=r\cdot\cos(t)$ and $y=r\cdot\sin(t)$ where $t$ is the number of radians for each point drawn along the circle's circumference. When the number of points is small, we get hexagons (6 points) and octogons (8 points). As as the number of points increases, we get shapes that look closer and closer to a circle (imagine 32 points, for example).

As the image below shows, a polygon can be drawn with triangles by dividing $2\pi$ radians into equal angles. The number of divisions is the same as the number of sides for the polygon which is also the same as the number of vertices. The polygon pictured below is an octogon, so $\theta=\frac{2\pi}{8}=\frac{\pi}{4}$. Then, the equations for the circumference of a circle give us the vertices $(r\cdot\cos(\frac{\pi}{4}),r\cdot\sin(\frac{\pi}{4}))$, $(r\cdot\cos(\frac{\pi}{2}),r\cdot\sin(\frac{\pi}{2}))$, $(r\cdot\cos(\frac{3\pi}{4}),r\cdot\sin(\frac{3\pi}{4}))$, and so on up to $(r\cdot\cos(2\pi),r\cdot\sin(2\pi))$.

![A regular polygon is made up of triangles sharing the same point in the center.](/software-engineering-lab/assets/images/polygon_vertices.png)

Since we are drawing with trianlges in OpenGL, we need to group the vertices in sets of three and arrange them in counterclockwise order to indicate the front side for rendering. With this in mind, we can write an initialization method for the `PolygonGeometry` class that calculates all the vertices of a polygon from its number of sides and radius.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `geometry` folder, create a new file called `polygon_geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open `polygon_geometry.py` for editing and add the following code:  

```python
from math import sin, cos, pi

from geometry.geometry import Geometry

class PolygonGeometry(Geometry):
    """Renders a regular polygon with the given number of sides and radius."""
    def __init__(self, sides=3, radius=1):
        super().__init__()

        theta = 2 * pi / sides
        position_data = []
        color_data = []

        for n in range(sides):
            # calculate the vertices for the triangle of side n
            position_data += (
                (0, 0, 0),
                (radius*cos(n*theta), radius*sin(n*theta), 0),
                (radius*cos((n+1)*theta), radius*sin((n+1)*theta), 0)
            )
            # the center is white, the sides interpolate between red and blue
            color_data += ((1, 1, 1), (1, 0, 0), (0, 0, 1))

        self.setAttribute("vertexPosition", position_data, "vec3")
        self.setAttribute("vertexColor", color_data, "vec3")
        self.countVertices()
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `PolygonGeometry` inherits from the `Geometry` class, so it has everything it needs to manage its own `Attribute` objects. After calculating the vertices, we set attribute data for the `vertexPosition` and `vertexColor` shader variables to use when rendering this polygon.

Since the `__init__` method has the parameters `sides` and `radius`, it is easy to create any kind of regular polygon we like. For example, a hexagon would give the argument of `6` to the `sides` parameter, but a pentagon would use an argument of `5` instead. In code, this would look similar to the following:

```python
hexagon = PolygonGeometry(sides=6)
pentagon = PolygonGeometry(sides=5)
```

This class will be very useful later when we create cylinder and cone geometries.

# Parametric Geometries

Similar to the way we calculate polygon vertices above, we can use functions to calculate a variety of different surfaces in 3D. The simplest surface is a plane with $z$ coordinates calculated from the $x$ and $y$ coordinates of each vertex. That function would be $z=f(x,y)$, but then there are no functions for $f$ that would produce vertex coordinates for shapes like spheres and cylinders. Instead, we will use two variables $u$ and $v$, each with a set range, to define the coordinates $x$, $y$, and $z$. That is,

$$x=f(u,v) \text{,} \hspace{1cm} y=g(u,v) \text{,} \hspace{1cm} z=h(u,v)$$

Putting this together, we can say the *parametric function* $S$ graphs output values $(x,y,z)$ from a rectangular region defined by the range of $u$ and $v$ values.

$$S(u,v) = (x,y,z) = \left( f(u,v), g(u,v), h(u,v) \right)$$

So we can think of $u$ and $v$ as vectors that define a rectangle, and different values for $u$ and $v$ will produce different points somewhere on the surface of the rectangle.  

![Vectors u and v define the range of points on a rectangular plane.](/software-engineering-lab/assets/images/uv_plane.png)

Here the simplest function for $S$ would be $S(u,v) = (u,v,0)$ which maps each value of $u$ and $v$ directly to $x$ and $y$ coordinates of the same value and gives vertices for a flat 2D plane. However, more complicated functions are necessary to map the same values of $u$ and $v$ to the vertices of spheres and cylinders, for example.  

![A 2D plane mapped by different functions can produce surfaces for spheres and cylinders.](/software-engineering-lab/assets/images/spheres_and_cylinders.png)

The images above show a plane, a sphere, and a cylinder drawn with triangles that are calculated from sampling the ranges of $u$ and $v$ at set intervals. Here, the number of samples taken in each range is called the *resolution* and the step in the range between samples is the *delta*. Each parametric geometry will take the start and end values for the ranges $u$ and $v$ as well as their respective resolutions. It will also take a surface function and use it to calculate all the points in between by feeding it all the sample values at the given resolution.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `geometry` folder, create a new file called `parametric_geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open `parametric_geometry.py` for editing and add the following code:  

```python
from numpy import linspace

from geometry.geometry import Geometry

class ParametricGeometry(Geometry):
    """A geometric surface rendered with the given function for parameters u and v."""
    def __init__(self, u_start, u_stop, u_resolution,
                       v_start, v_stop, v_resolution, surface_function):
        super().__init__()
        
        # generate a matrix of vertex points for all values of (u,v)
        point_matrix = []
        for u in linspace(u_start, u_stop, u_resolution + 1):
            matrix_row = []
            for v in linspace(v_start, v_stop, v_resolution + 1):
                matrix_row.append(surface_function(u,v))
            point_matrix.append(matrix_row)
```

The `__init__` class accepts the minimum and maximum values for each range $u$ and $v$, defined by `u_start`, `u_stop`, `v_start`, and `v_stop`. In Python, we can store functions in variables and pass them around like any other value. This means we can define a parameter `surface_function` as a Python function that calculates vertex coordinates from $u$ and $v$ values.  

This class imports the NumPy method [`linspace`](https://numpy.org/doc/stable/reference/generated/numpy.linspace.html) which is very convenient for producing the range of values we need for $u$ and $v$. We use a nested `for` loop to run through each pair of $(u,v)$ sample values. These values are passed as parameters to the surface function and the result is stored in a 2D matrix of coordinate data that corresponds with each pair of $(u,v)$ values.

<input type="checkbox" class="checkbox inline"> Next add the following code to the `__init__` method of the `ParametricGeometry` class:  

```python
        # store vertex data
        position_data = []
        color_data = []

        # default vertex color data: red, green, blue, cyan, magenta, yellow
        C1, C2, C3 = [1, 0, 0], [0, 1, 0], [0, 0, 1]
        C4, C5, C6 = [0, 1, 1], [1, 0, 1], [1, 1, 0]

        # store vertices for each rectangular segment as a pair of triangles
        for n in range(u_resolution):
            for m in range(v_resolution):
                P1 = point_matrix[n + 0][m + 0].copy()
                P2 = point_matrix[n + 1][m + 0].copy()
                P3 = point_matrix[n + 1][m + 1].copy()
                P4 = point_matrix[n + 0][m + 1].copy()
                position_data += [P1,P2,P3, P1,P3,P4]
                color_data += [C1,C2,C3, C4,C5,C6]

        self.setAttribute("vertexPosition", position_data, "vec3")
        self.setAttribute("vertexColor", color_data, "vec3")
        self.countVertices()
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The remainder of the `__init__` method defines vertices for each rectangular segment of the surface as a pair of triangles. We do this in a way similar to the [`RectangleGeometry`](/software-engineering-lab/notes/geometry_and_material/#rectangles) class from the "Geometry and Material Objects" lesson. This time, our vertex data is stored in the `point_matrix` which we draw out according to the position of the rectangular segment on the surface of the object. Here we use the `copy` method to get an exact copy of the Python list of each point; otherwise, it will store a reference to the existing list which is not what we want.

Now we have a strong foundation for creating parametric geometries. Every new geometry we make from this point on will just require us to define a surface function and tweak the parameters as necessary.

## Planes

A plane is the simplest parametric object as it uses the parametric function that directly maps $u$ and $v$ values to $x$ and $y$ coordinates: $S(u,v) = (u,v,0)$. When rendered, it looks just like a `RectangleGeometry` instance, except that it is divided into a number of smaller rectangles as defined by the resolutions of $u$ and $v$. 

Let's create a `PlaneGeometry` class that renders a plane with its center at $(0,0)$. Here, we will refer to range $u$ as the width and range $v$ as the height for clarity.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `geometry` folder, create a new file called `plane_geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open `plane_geometry.py` for editing and add the following code:  

```python
from geometry.parametric_geometry import ParametricGeometry

class PlaneGeometry(ParametricGeometry):
    """A 2D plane divided into segments."""
    def __init__(self, width=1, height=1, width_segments=8, height_segments=8):
        
        # The surface function S(u,v) = (u,v,0)
        surface_function = lambda u,v: [u, v, 0]
        
        super().__init__( -width/2, width/2, width_segments, 
                          -height/2, height/2, height_segments, 
                          surface_function)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

When we define a plane, we can set its width and height as well as it segmentation along each dimension. We then define the surface function using a special Python function called `lambda`. Lambda functions are small anonymous functions that only evaluate a single expression. Here, our function `lambda u,v: [u, v, 0]` takes values from the two parameters `u` and `v` then returns a list containing those values and a `0`. In this way we can easily represent the parametric function $S(u,v)=(u,v,0)$.

With our surface function defined, then we just call the `__init__` method on the superclass `ParametricGeometry` with the range of width values for $u$, the range of height values for $v$, and the surface function itself.

# Ellipsoids

Rounded shapes such as spheres are essentially made up of several circles of different sizes (called *cross-sections*). At the center of the sphere, the radius of the circular cross-section is equal to the radius of the sphere. At the top and bottom of the sphere, the radius of the cross-section is $0$. Every cross-section in between has a radius somewhere in between. Here we can use a parametric function $S(u,v)=(x,y,z)$ where $u$ is the range of radians for each cross-section and $v$ is the range of radians from the bottom of the sphere to the top.

So for cross-sections that run along the $y$-axis with radius $r$, we can define $z=r\cdot\cos(u)$ and $x=r\cdot\sin(u)$ where $0\le u\le 2\pi$. But now the radius $r$ also relates to the value of the $y$-coordinate. For a sphere with radius $1$, when $y=0$ the cross-section radius $r=1$ and when $y=1$ or $y=-1$ then $r=0$. If we consider these values as the result of a parametric function for $v$ then we can express them as $y=cos(v)$ and $r=sin(v)$ where $-\frac{\pi}{2} \le v \le \frac{\pi}{2}$.

| $v$              | $r$ | $y$ |
| ---------------- | --- | --- |
| $-\frac{\pi}{2}$ | 0   | -1  |
| 0                | 1   | 0   |
| $\frac{\pi}{2}$  | 0   | 1   |

Putting this all together then we see our parametric function for a sphere is:

$$S(u,v) = \left( \sin(u)\cdot\cos(v), \sin(v), \cos(u)\cdot\cos(v) \right) \\
\text{where } 0\le u\le 2\pi \text{ and } -\frac{\pi}{2} \le v \le \frac{\pi}{2}$$

Given that this function gives us the coordinates for a perfect sphere of radius $1$, we can create an ellipsoid by simply applying the three dimensions of width, height, and depth, to the $x$, $y$, and $z$ coordinates. Now we can write an `EllipsoidGeometry` class with its center at $(0,0,0)$ based on everything above.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `geometry` folder, create a new file called `ellipsoid_geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open `ellipsoid_geometry.py` for editing and add the following code:  

```python
from math import sin, cos, pi

from geometry.parametric_geometry import ParametricGeometry

class EllipsoidGeometry(ParametricGeometry):
    """A unit sphere stretched by the given factors of width, height, and depth."""
    def __init__(self, width=1, height=1, depth=1, 
                       radial_segments=32, height_segments=16):

        # calculates points on the surface
        surface_function = lambda u,v: [
            width/2 * sin(u) * cos(v),
            height/2 * sin(v),
            depth/2 * cos(u) * cos(v)
        ]

        super().__init__(0, 2*pi, radial_segments, 
                            -pi/2, pi/2, height_segments, surface_function)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Since the width, height, and depth dimensions span across the entire object, we only apply half of their values to the coordinates of a unit sphere to get the final coordinates of the ellipsoid.

When we call the superclass `__init__` method, we give our values for $u$ and $v$ for the ranges previously mentioned. Then we can think of the `radial_segments` value as the number of triangles that each cross-section will be divided into, and the `height_segments` value is the number of total cross-sections.

## Spheres

When we want to create a perfect sphere, it is useful to have a simpler interface than the one for an ellipsoid. For a sphere, the width, height and depth are all the same value which is double the radius. 

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `geometry` folder, create a new file called `ellipsoid_geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open `ellipsoid_geometry.py` for editing and add the following code:  

```python
from geometry.ellipsoid_geometry import EllipsoidGeometry

class SphereGeometry(EllipsoidGeometry):
    """A perfect sphere with the given radius."""
    def __init__(self, radius=1, radial_segments=32, height_segments=16):
        super().__init__(2*radius, 2*radius, 2*radius, 
                         radial_segments, height_segments)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Now when we use a `SphereGeometry` instance, we only need to give it a radius instead of three values for the width, height, and depth.

# Cylindrical Geometries

A cylinder is not as complicated as a sphere because all of its cross-sections have the same radius. So we can once again adopt the equations $z=r\cdot\cos(u)$ and $x=r\cdot\sin(u)$ where $0\le u\le 2\pi$. As for the $y$-coordinate, it will depend on the height $h$ of the cylinder and fall in the range of $-\frac{h}{2} \le y \le \frac{h}{2}$ for a cylinder centered at the origin. We can express this range with the parameter $v$ as,

$$y=h\cdot \left( v - \frac{1}{2} \right) \text{, where } 0 \le v \le 1$$

Now, if we consider that the top of a cylindrical geometry can have a different radius than the bottom, then we can imagine a wider range of geometries such as cones and pyramids. For a cone standing upright, the radius of the top cross-section is the smallest at $0$ and the radius of the bottom cross-section is the widest. Given that our $v$ parameter expresses the $y$-coordinates from a range of values between $0$ and $1$ (above), we can also use it to express the cross-section radius $r$. If we have a bottom radius $s$, then $r=(1-v)\cdot s$ since $v=0$ at the bottom, $v=1$ at the top, and all the points in between are in a straight line. 

But what if we also have a non-zero radius at the top of the cylindrical object? In that case, with a top radius $t$ and a bottom radius $s$, then the cross-section radius $r$ can be expressed as $r=v\cdot t + (1-v)\cdot s$ where $0 \le v \le 1$.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `geometry` folder, create a new file called `cylindrical_geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open `cylindrical_geometry.py` for editing and add the following code:  

```python
from math import sin, cos, pi

from geometry.parametric_geometry import ParametricGeometry
from geometry.polygon_geometry import PolygonGeometry
from core.matrix import Matrix

class CylindricalGeometry(ParametricGeometry):
    """A cylindrical object with the given top and bottom radiuses."""
    def __init__(self, top_radius=1, bottom_radius=1, height=1,
                       radial_segments=32, height_segments=4, 
                       top_closed=True, bottom_closed=True):
        
        # calculates points on the surface
        surface_function = lambda u,v: [
            (v * top_radius + (1-v) * bottom_radius) * sin(u),
            height * (v - 0.5),
            (v * top_radius + (1-v) * bottom_radius) * cos(u)
        ]

        super().__init__(0, 2*pi, radial_segments, 
                         0, 1, height_segments, surface_function)
```

The `CylindricalGeometry` class has two new parameters: `top_closed` and `bottom_closed`. As it is now, this geometry object will only render the rounded surface of the cylindrical object's sides. In order to render the top and bottom surfaces, we can use our `PolygonGeometry` class that we wrote earlier with a couple alterations.

First, we need the ability to transform the vertices of the geometry object. Then we need a way to merge the attribute data of two different geometry objects so they can be combined into one. These are generic features that should not depend on the type of geometry, so let's add them to the `Geometry` class.

<input type="checkbox" class="checkbox inline"> In the `geometry` folder, open `geometry.py` for editing.  
<input type="checkbox" class="checkbox inline"> Scroll down to the bottom of the file and add the following method to the `Geometry` class:  

```python
    def applyMatrix(self, matrix, variableName="vertexPosition"):
        """Transform the data in an attribute using the given matrix."""
        if variableName not in self._attributes.keys():
            raise Exception(f"Unable to apply matrix to unknown attribute: {variableName}")

        old_position_data = self._attributes[variableName].data
        new_position_data = []

        for old_pos in old_position_data:
            # copy the data and add a homogeneous fourth coordinate
            new_pos = old_pos + (1,)

            # apply the matrix
            new_pos = matrix @ new_pos

            # remove the homogeneous coordinate and append to the new data
            new_pos = new_pos[:3]
            new_position_data.append(new_pos)

        self.setAttribute(variableName, new_position_data)
```

Remember that applying matrix transformations in 3D requires a fourth dimensional coordinate called the *homogeneous coordinate*. Since `Geometry` instances do not store vertex data in 4D, we need to add one before applying the matrix. In the code `old_pos + (1,)` we need the comma to indicate that we are creating a tuple with a single value. If there is no comma, then it will be treated simply as the value `1`.

<input type="checkbox" class="checkbox inline"> Next add the following method to the `Geometry` class:  

```python
    def merge(self, other_geometry):
        """Merge data from attributes of other geometries into this object.
           Both geometries must share attributes with the same names."""
        for variableName, attribute in self._attributes.items():
            attribute.data += other_geometry.attributes[variableName].data
            self.setAttribute(variableName, attribute.data)

        self.countVertices()
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `merge` method is pretty straightforward. It loops through each variable in the `self._attributes` dictionary, gets data for the variable from the other geometry object, and adds that data to its own variable before finally updating its vertex count.

Now we can complete the `CylindricalGeometry` class to give it top and bottom surfaces.

<input type="checkbox" class="checkbox inline"> Open `cylindrical_geometry.py` again and add the following code to the end of the `__init__` method:  

```python
        # add polygons to the top and bottom if requested
        if top_closed:
            top_geometry = PolygonGeometry(radial_segments, top_radius)
            rotation = Matrix.makeRotationY(-pi/2) @ Matrix.makeRotationX(-pi/2)
            transform = Matrix.makeTranslation(0, height/2, 0) @ rotation
            top_geometry.applyMatrix(transform)
            self.merge(top_geometry)

        if bottom_closed:
            bottom_geometry = PolygonGeometry(radial_segments, bottom_radius)
            rotation = Matrix.makeRotationY(-pi/2) @ Matrix.makeRotationX(pi/2)
            transform = Matrix.makeTranslation(0, -height/2, 0) @ rotation
            bottom_geometry.applyMatrix(transform)
            self.merge(bottom_geometry)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

We handle the top and bottom surfaces separately for maximum flexibility. For each one, we create a polygon with the same number of sides as the cylindrical geometry. Then we rotate it and translate it into position. Finally, we merge its data with this cylindrical geometry object.

## Cylinders

Now that all of the hard parts are complete, cylinders are really easy to make. The `CylinderGeometry` class will simply restrict the interface to the `CylindricalGeometry` superclass to comply with the fact that a cylinder's radius is the same everywhere along its height.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `geometry` folder, create a new file called `cylindrical_geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open `cylindrical_geometry.py` for editing and add the following code:  

```python
from geometry.cylindrical_geometry import CylindricalGeometry

class CylinderGeometry(CylindricalGeometry):
    "A cylindrical object with the same radius at the top and bottom."
    def __init__(self, radius=1, height=1, radial_segments=32,
                       height_segments=4, top_closed=True, bottom_closed=True):
        
        super().__init__(radius, radius, height, 
                         radial_segments, height_segments, 
                         top_closed, bottom_closed)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Now making a cylinder with `top_closed=False` will create a cup shape and additionally `bottom_closed=False` will create a tube shape.

Changing the `radial_segments` parameter to smaller values such as `8` or `3` will decrease the number of sides and effectively create prism objects.

## Cones

Finally, we can extend the `CylindricalGeometry` class and give it specific parameters to create cone-shaped objects. A cone is unique in that its top radius is $0$ but its bottom radius can be any non-zero value. As with cylinders, we will simplify the construction of a `CylindricalGeometry` object in the `__init__` method of the `ConeGeometry` class.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `geometry` folder, create a new file called `cylindrical_geometry.py`.  
<input type="checkbox" class="checkbox inline"> Open `cylindrical_geometry.py` for editing and add the following code:  

```python
from geometry.cylindrical_geometry import CylindricalGeometry

class ConeGeometry(CylindricalGeometry):
    """A cylindrical object that comes to a point at the top."""
    def __init__(self, radius=1, height=1, radial_segments=32,
                       height_segments=4, closed=True):

        super().__init__(0, radius, height, radial_segments, 
                         height_segments, False, closed)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `ConeGeometry` class is also useful for creating pyramids. Simply make a new instance with `radial_segments` set to the number of sides you want (like 3 or 4), and your cone will become a pyramid!