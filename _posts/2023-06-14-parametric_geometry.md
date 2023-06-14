---
# MARP
theme: default
paginate: true

# Jekyll
title: "10_Polygons, Planes, Spheres, and Cylinders"
date: 2023-06-14
categories:
  - Notes
classes: wide
toc_sticky: false
---

*In this lesson, we add more tools to our geometry toolset with the introduction of a polygon geometry class and a  parametric geometry base class. Then, we extend their capabilities to create various planes, ellipsoids, and cylindrical geometries.*  

As we build 3D scenes, we will need to use a variety of different shapes and geometries. Last time we created classes for basic rectangles and boxes, but these along will not be enough. Other types of objects such as polygons, spheres, ellipsoids, cylinders, cones, prisms, and pyramids are useful for complicated 3D scenes. This lesson introduces a `PolygonGeometry` class that can render 2D shapes from any number of sides with equal lengths. Then, we create a `ParametricGeometry` class which allows us to render 3D surfaces in segments defined by a parametric function. The `ParametricGeometry` class provides a good foundation for creating ellipsoids, spheres, cylinders, prisms, pyramids, and cones. Finally, we extend the `ParametricGeometry` class for each of those types of geometry by defining the parametric function that calculates vertices along its surface.

# Polygons

Our `PolygonGeometry` class will provide the ability to render *regular polygons* which are 2D shapes where all sides and angles are equal. Regular polygons include equilateral triangles (3 sides), squares (4 sides), pentagons (5 sides), hexagons (6 sides), heptagons (7 sides), octogons (8 sides), and so on.

We can calculate the points of a polygon with radius $r$ as equally spaced points along the circumference of a circle with the same radius $r$. Recall that previously we defined the circular path of a moving triangly by using $x=r\cdot\cos(t)$ and $y=r\cdot\sin(t)$ where $t$ is the number of radians. Together, these are the parametric equations for specifying points along the circumference of a circle. When the number of points is small and we draw straight lines between each consecutive point, we get common polygons such as hexagons (6 points) and octogons (8 points). As as the number of points increases, we get shapes that look closer and closer to a circle (imagine 32 points, for example).

The image below demonstrates how to draw a polygon with triangles by dividing $2\pi$ radians into equal angles. The number of divisions is the same as the number of sides for the polygon which is also the same as the number of vertices. The polygon pictured below is an octogon, so $\theta=\frac{2\pi}{8}=\frac{\pi}{4}$. Then, the equations for the circumference of a circle give us the vertices,

$$\begin{aligned}
P_0 &= (r\cdot\cos(\frac{\pi}{4}),r\cdot\sin(\frac{\pi}{4})) \\
P_1 &= (r\cdot\cos(\frac{\pi}{2}),r\cdot\sin(\frac{\pi}{2})) \\
P_2 &= (r\cdot\cos(\frac{3\pi}{4}),r\cdot\sin(\frac{3\pi}{4})) \\
... \\
P_7 &= (r\cdot\cos(2\pi),r\cdot\sin(2\pi))
\end{aligned}$$

![A regular polygon is made up of triangles sharing the same point in the center.](/software-engineering-lab/assets/images/polygon_vertices.png)

Since we are drawing with triangles in OpenGL, we need to list the vertices in sets of three and arrange them in counterclockwise order to indicate the front side for rendering. The initialization method for the `PolygonGeometry` class will do that after calculating all the vertices of the polygon from the given number of sides and radius.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `geometry` folder, open the file called `basic_geometries.py` and add the following import statement to the top of it.

```python
from math import sin, cos, pi
```

<input type="checkbox" class="checkbox inline"> Scroll to the end of `basic_geometries.py` and add the following code after the `BoxGeometry` class:  

```python
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

The `PolygonGeometry` class inherits from the `Geometry` class, so it has everything it needs to manage its own `Attribute` objects. After calculating the vertices, we set attribute data for the `vertexPosition` and `vertexColor` shader variables to use when rendering this polygon.

Since the `__init__` method has the parameters `sides` and `radius`, it is easy to create any kind of regular polygon we like. For example, `sides=6` will create a hexagon, but `sides=5` will create a pentagon instead. In code, this might look like the following:

```python
hexagon = PolygonGeometry(sides=6)
pentagon = PolygonGeometry(sides=5)
```

The `PolygonGeometry` class will become very useful later when we create the flat ends of cylinder and cone geometries.

# Parametric Geometries

Similar to the way we calculate polygon vertices above, we can use parametric functions to calculate a variety of different surfaces in 3D. A simple surface is a plane where the $z$ coordinates of each vertex are calculated directly from the $x$ and $y$ coordinates, as expressed by $z=f(x,y)$. However, there is no function $f$ that can produce $z$ coordinates for the vertices of shapes like spheres and cylinders. Those shapes have multiple vertices sharing the same $x$ and $y$ coordinates, so we cannot calculate $z$ coordinates from them alone. Instead, we will use two variables $u$ and $v$ with fixed ranges to define the coordinates $x$, $y$, and $z$. That is,

$$x=f(u,v) \text{,} \hspace{1cm} y=g(u,v) \text{,} \hspace{1cm} z=h(u,v)$$

Combined, we can say the *parametric function* $S$ graphs output values $(x,y,z)$ from a region of inputs defined by the ranges of $u$ and $v$ values.

$$S(u,v) = (x,y,z) = \left( f(u,v), g(u,v), h(u,v) \right)$$

So we can think of $u$ and $v$ as vectors that define a rectangle, and different values for $u$ and $v$ will produce different points somewhere on the surface of the rectangle.  

![Vectors u and v define the range of points on a rectangular plane.](/software-engineering-lab/assets/images/uv_plane.png)

Here the simplest function for $S$ would be $S(u,v) = (u,v,0)$ which maps each value of $u$ and $v$ directly to $x$ and $y$ coordinates, giving vertices for a flat 2D plane. More complicated functions are necessary to map the same values of $u$ and $v$ to the vertices of more complicated shapes such as spheres and cylinders.  

![A 2D plane mapped by different functions can produce surfaces for spheres and cylinders.](/software-engineering-lab/assets/images/spheres_and_cylinders.png)

The images above showing a plane, a sphere, and a cylinder are drawn with triangles calculated from sampling the ranges of $u$ and $v$ at set intervals. Here, we call the number of samples taken in each range the *resolution* and the step in between samples is the *delta*. Each parametric geometry will take the start and stop values for the ranges $u$ and $v$ as well as their respective resolutions. It will also take a surface function that defines the shape of the surface. When initialized, the geometry will use its function to calculate all the points along its surface by executing the function with every pair of sample values in the ranges $u$ and $v$.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `geometry` folder, create a new file called `parametric_geometries.py`.  
<input type="checkbox" class="checkbox inline"> Open `parametric_geometries.py` for editing and add the following code:  

```python
# geometry.parametric_geometries.py
from numpy import linspace

from geometry import Geometry

class ParametricGeometry(Geometry):
    """A geometric surface rendered with the given function for parameters u and v."""
    def __init__(self, u_start, u_stop, u_resolution,
                       v_start, v_stop, v_resolution, surface_function):
        super().__init__()
        
        # generate a matrix of point vertices for all (u,v) values in range
        point_matrix = []
        for u in linspace(u_start, u_stop, u_resolution + 1):
            matrix_row = []
            for v in linspace(v_start, v_stop, v_resolution + 1):
                matrix_row.append(surface_function(u,v))
            point_matrix.append(matrix_row)
```

The `__init__` class accepts the minimum and maximum values for each range $u$ and $v$, defined by `u_start`, `u_stop`, `v_start`, and `v_stop`. In Python, we can store functions in variables and pass them around like any other value. This means we can define the parameter `surface_function` as a Python function that calculates vertex coordinates from $u$ and $v$ values. Then we can call it like any other function with `surface_function(u,v)`.  

This class imports the function [`linspace`](https://numpy.org/doc/stable/reference/generated/numpy.linspace.html) from NumPy which gives us the range of sample values for $u$ and $v$ with their given resolutions. We use a nested `for` loop to run through every pair of $(u,v)$ sample values. These values are passed as parameters to the surface function and the result is stored in a 2D matrix of coordinate data that corresponds with each pair of $(u,v)$ values.

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
                P1 = point_matrix[n + 0][m + 0]
                P2 = point_matrix[n + 1][m + 0]
                P3 = point_matrix[n + 1][m + 1]
                P4 = point_matrix[n + 0][m + 1]
                position_data += [P1,P2,P3, P1,P3,P4]
                color_data += [C1,C2,C3, C4,C5,C6]

        self.set_attribute("vertexPosition", position_data, "vec3")
        self.set_attribute("vertexColor", color_data, "vec3")
        self.count_vertices()
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

This part of the `__init__` method defines vertices for each rectangular segment of the surface as a pair of triangles. We do this in a way similar to the [`RectangleGeometry`](/software-engineering-lab/notes/geometry_and_material/#rectangles) class from the previous lesson. This time, the vertex data for each rectangle is stored in the `point_matrix` in the order that they will be drawn on.  

Now we have a strong foundation for creating parametric geometries. Every new geometry we make from this point on will just require us to define a surface function and tweak the parameters as necessary.

## Planes

A plane is the simplest parametric object as it uses the parametric function that directly maps $u$ and $v$ values to $x$ and $y$ coordinates: $S(u,v) = (u,v,0)$. When rendered, it looks just like a `RectangleGeometry` instance, except it is divided into a number of smaller rectangles as defined by the resolutions of $u$ and $v$. 

Let's create a `PlaneGeometry` class that renders a plane with its center at the origin. Here, we will refer to range $u$ as the width and range $v$ as the height for clarity.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `parametric_geometries.py` file, add the following code after the `ParametricGeometry` class:  

```python
class PlaneGeometry(ParametricGeometry):
    """A 2D plane divided into segments."""
    def __init__(self, width=1, height=1, width_segments=8, height_segments=8):
        
        # The surface function S(u,v) = (u,v,0)
        surface_function = lambda u,v: (u, v, 0)
        
        super().__init__( -width/2, width/2, width_segments, 
                          -height/2, height/2, height_segments, 
                          surface_function)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

When we define a plane, we can set its width and height as well as its segmentation along each dimension. We then define the surface function using a special Python function called `lambda`. Lambda functions are small anonymous functions that only evaluate a single expression. Here, our function `lambda u,v: [u, v, 0]` takes values from the two parameters `u` and `v` then returns a list containing those values followed by a `0`. In this way we can easily represent the parametric function $S(u,v)=(u,v,0)$.

With our surface function defined, then we just call the `__init__` method on the superclass `ParametricGeometry` with the range of width values for $u$, the range of height values for $v$, and the surface function itself.

# Ellipsoids

Rounded shapes such as spheres are essentially made up of several circles of different sizes (called *cross-sections*). At the center of the sphere, the radius of the circular cross-section is equal to the radius of the sphere. At the top and bottom of the sphere, the radius of the cross-section is $0$. Every cross-section in between has a radius somewhere in between. Here we can use a parametric function $S(u,v)=(x,y,z)$ where $u$ is the range of $t$ values for the parametic equations of the cross-sections and $v$ is the range of radians for a half-circle as if it were drawn along the $y$-axis from the bottom of the sphere to the top.

For cross-sections that are perpendicular to the $y$-axis with radius $r$, we can define $z=r\cdot\cos(u)$ and $x=r\cdot\sin(u)$ where $0\le u\le 2\pi$. The radius of these cross-sections $r$ relate directly to the value of the cross-section's $y$-coordinate. For a sphere with radius $1$, the cross-section radius $r=1$ when $y=0$ and $r=0$ when $y=1$ or $y=-1$. If we consider these values as the result of a parametric function for $v$ then we can express them as $y=cos(v)$ and $r=sin(v)$ where $-\frac{\pi}{2} \le v \le \frac{\pi}{2}$.

| $v$              | $r$ | $y$ |
| ---------------- | --- | --- |
| $-\frac{\pi}{2}$ | 0   | -1  |
| 0                | 1   | 0   |
| $\frac{\pi}{2}$  | 0   | 1   |

Putting this all together then we see our parametric function for a sphere is:

$$S(u,v) = \left( \sin(u)\cdot\cos(v), \sin(v), \cos(u)\cdot\cos(v) \right) \\
\text{where } 0\le u\le 2\pi \text{, and } -\frac{\pi}{2} \le v \le \frac{\pi}{2}$$

Given that this function gives us the coordinates for a perfect sphere of radius $1$, we can create an ellipsoid by simply applying the three dimensions of width, height, and depth, to the $x$, $y$, and $z$ coordinates. Now let's write an `EllipsoidGeometry` class with its center at $(0,0,0)$ based on everything above.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `parametric_geometries.py` file, add the following import to the top of the file:  

```python
from math import sin, cos, pi
```

<input type="checkbox" class="checkbox inline"> Scroll to the end of the `parametric_geometries.py` file and add the following code after the `PlaneGeometry` class:  

```python
class EllipsoidGeometry(ParametricGeometry):
    """A unit sphere stretched by the given factors of width, height, and depth."""
    def __init__(self, width=1, height=1, depth=1, 
                       radial_segments=32, height_segments=16):

        # calculates points on the surface
        surface_function = lambda u,v: (
            width/2 * sin(u) * cos(v),
            height/2 * sin(v),
            depth/2 * cos(u) * cos(v)
        )

        super().__init__(0, 2*pi, radial_segments, 
                         -pi/2, pi/2, height_segments, surface_function)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Since the width, height, and depth dimensions span across the entire object, we only apply half of their values to the coordinates of a unit sphere to get the final coordinates of the ellipsoid centered at the origin.

When we call the superclass `__init__` method, we give our values for $u$ and $v$ for the ranges previously mentioned. Here we can think of the `radial_segments` value as the number of triangles that each cross-section will be divided into, and the `height_segments` value is the total number of cross-sections.

## Spheres

When we want to create a perfect sphere, it is useful to have a simpler interface than the one for an ellipsoid since the width, height, and depth of a sphere are all the same value.  

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `parametric_geometries.py` file, add the following code after the `EllipsoidGeometry` class:  

```python
class SphereGeometry(EllipsoidGeometry):
    """A perfect sphere with the given radius."""
    def __init__(self, radius=1, radial_segments=32, height_segments=16):

        super().__init__(2*radius, 2*radius, 2*radius, 
                         radial_segments, height_segments)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

Now when we use a `SphereGeometry` instance, we only need to give it a radius instead of three values for the width, height, and depth.

# Cylindrical Geometries

A cylinder is not as complicated as a sphere because all of its cross-sections have the same radius. So we can once again adopt the equations $z=r\cdot\cos(u)$ and $x=r\cdot\sin(u)$ where $0\le u\le 2\pi$. As for the $y$-coordinate, it will depend on the height $h$ of the cylinder and fall in the range of $-\frac{h}{2} \le y \le \frac{h}{2}$ for a cylinder centered at the origin. We can express the $y$-coordinates with the parameter $v$ as,

$$y=h\cdot \left( v - \frac{1}{2} \right) \text{, where } 0 \le v \le 1$$

Now, if we consider that the top of a cylindrical geometry can have a different radius than the bottom, then we can imagine a wider range of geometries such as cones and pyramids. For a cone standing upright, the radius of the top cross-section is the smallest at $0$ and the radius of the bottom cross-section is the widest. Given that our $v$ parameter expresses the $y$-coordinates from a range of values between $0$ and $1$, we can also use it to express the cross-section radius $r$. If we have a bottom radius $s$, then $r=(1-v)\cdot s$ since $v=0$ at the bottom, $v=1$ at the top, and all the points in between are on a straight line. 

What if we also have a non-zero radius at the top of the cylindrical object? In that case, with a top radius $t$ and a bottom radius $s$, then the cross-section radius $r$ can be expressed as $r=v\cdot t + (1-v)\cdot s$ where $0 \le v \le 1$.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `parametric_geometries.py` folder, add the following import statements just before the `ParametricGeometry` class.  

```python
from geometry.basic_geometries import PolygonGeometry
from core.matrix import Matrix
```

<input type="checkbox" class="checkbox inline"> Scroll to the end of the `parametric_geometries.py` file and add the following code after the `SphereGeometry` class:  

```python
class CylindricalGeometry(ParametricGeometry):
    """A cylindrical object with the given top and bottom radiuses."""
    def __init__(self, top_radius=1, bottom_radius=1, height=1,
                       radial_segments=32, height_segments=4, 
                       top_closed=True, bottom_closed=True):
        
        # calculates points on the surface
        surface_function = lambda u,v: (
            (v * top_radius + (1-v) * bottom_radius) * sin(u),
            height * (v - 0.5),
            (v * top_radius + (1-v) * bottom_radius) * cos(u)
        )

        super().__init__(0, 2*pi, radial_segments, 
                         0, 1, height_segments, surface_function)
```

The `CylindricalGeometry` class has two new parameters: `top_closed` and `bottom_closed`. As it is now, this geometry object will only render the rounded surface of the cylindrical object's sides. In order to render the top and bottom surfaces, we can use our `PolygonGeometry` class that we wrote earlier with a couple alterations.

First, we need the ability to transform the vertices of the geometry object. We already did the work of creating transformation matrices with our `Matrix` class, so ideally we should be able to apply those matrices to a `Geometry` object as well. We also need a way to merge the attribute data of two different geometry objects so they can be combined into one. Without this merge functionality, the top and bottom surfaces of cylinders would need to be separate mesh objects handled by the application, which is just more trouble for the programmer. 

These are generic features that should not depend on the type of geometry, so let's add them to the `Geometry` class.

<input type="checkbox" class="checkbox inline"> In the `geometry` folder, open `__init__.py` for editing.  
<input type="checkbox" class="checkbox inline"> Scroll down to the bottom of the file and add the following method to the `Geometry` class:  

```python
    def apply_matrix(self, matrix, variable_name="vertexPosition"):
        """Transform the data in an attribute using the given matrix."""
        if variable_name not in self._attributes.keys():
            raise Exception(f"Unable to apply matrix to unknown attribute: {variable_name}")

        old_position_data = self._attributes[variable_name].data
        new_position_data = []

        for old_pos in old_position_data:
            # copy the data and add a homogeneous fourth coordinate
            new_pos = old_pos + (1,)

            # apply the matrix
            new_pos = matrix @ new_pos

            # remove the homogeneous coordinate and append to the new data
            new_pos = new_pos[:3]
            new_position_data.append(new_pos)

        self.setAttribute(variable_name, new_position_data)
```

Remember that applying matrix transformations in 3D requires a fourth dimensional coordinate called the *homogeneous coordinate*. Since `Geometry` instances do not store vertex data in 4D, we need to add one before applying the matrix. In the code `old_pos + (1,)` we use a comma to indicate that we are creating a tuple with a single value. If there is no comma, then it will be treated simply as the value `1` instead of a tuple.

<input type="checkbox" class="checkbox inline"> Next add the following method to the `Geometry` class:  

```python
    def merge(self, other_geometry):
        """Merge data from attributes of other geometries into this object.
           Both geometries must share attributes with the same names."""
        for variable_name, attribute in self._attributes.items():
            attribute.data += other_geometry.attributes[variable_name].data
            self.set_attribute(variable_name, attribute.data)

        self.count_vertices()
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `merge` method is pretty straightforward. It loops through each variable in the `self._attributes` dictionary, gets data for the variable from the other geometry object, and then adds that data to its own variable before finally updating its vertex count.

Now we can complete the `CylindricalGeometry` class to give it top and bottom surfaces.

<input type="checkbox" class="checkbox inline"> Open the `parametric_geometries.py` file again and add the following code to the end of the `__init__` method of the `CylindricalGeometry` class:  

```python
        # add polygons to the top and bottom if requested
        if top_closed:
            top_geometry = PolygonGeometry(radial_segments, top_radius)
            rotation = Matrix.make_rotation_y(-pi/2) @ Matrix.make_rotation_x(-pi/2)
            transform = Matrix.make_translation(0, height/2, 0) @ rotation
            top_geometry.apply_matrix(transform)
            self.merge(top_geometry)

        if bottom_closed:
            bottom_geometry = PolygonGeometry(radial_segments, bottom_radius)
            rotation = Matrix.make_rotation_y(-pi/2) @ Matrix.make_rotation_x(pi/2)
            transform = Matrix.make_translation(0, -height/2, 0) @ rotation
            bottom_geometry.apply_matrix(transform)
            self.merge(bottom_geometry)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

We handle the top and bottom surfaces separately for maximum flexibility. For each one, we create a polygon with the same number of sides as the cylindrical geometry. Then we rotate it and translate it into position. Finally, we merge its data with this cylindrical geometry object.

## Cylinders

Now that all of the hard parts are complete, cylinders are really easy to make. The `CylinderGeometry` class restricts the interface to the `CylindricalGeometry` superclass to comply with the fact that a cylinder's radius is the same everywhere along its height.

:heavy_check_mark: ***Try it!***  
<input type="checkbox" class="checkbox inline"> Inside the `parametric_geometries.py` file, add the following code after the `CylindricalGeometry` class:  

```python
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
<input type="checkbox" class="checkbox inline"> Inside the `parametric_geometries.py` file, add the following code after the `CylinderGeometry` class:  

```python
class ConeGeometry(CylindricalGeometry):
    """A cylindrical object that comes to a point at the top."""
    def __init__(self, radius=1, height=1, radial_segments=32,
                       height_segments=4, closed=True):

        super().__init__(0, radius, height, radial_segments, 
                         height_segments, False, closed)
```

<input type="checkbox" class="checkbox inline"> Make sure there are no errors and save the file.  

The `ConeGeometry` class is also useful for creating pyramids. Simply make a new instance with 3 or 4 `radial_segments` and the "cone" will actually be a pyramid!