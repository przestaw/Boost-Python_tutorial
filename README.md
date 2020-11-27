# Boost.Python

## Requirements 

- python interpreter
- python development headers
- Boost.Python library

## Getting started

Start with a minimal viable example - a square function

1. Write our Cpp code

```cpp
#include <boost/python.hpp>

using namespace boost::python;

// first we need to define some function - in this case square
int square( int a ) {
	return a*a;
}

// second create module named "example"
BOOST_PYTHON_MODULE(example) {

	// lastly expose our function in module "example"
	def("square", &square, "Returns a square of given number");
}
```
	
2. Compile code as shared library - `*.so` on unix and `*.dll` on windows

3. The resulting shared library is now visible to Python. Here's a sample Python session:

```python
python3
>>> import example
>>> print(example.square(2))
4	
```
	
## Exposing Classes & Functions

Using Boost.Python we can expose both classes and functions.
Each of them can have separate documenting string given as last argument.

### Exposing functions

To expose a function we use a call to function *def*. 
We give a name to our function which may or may not correspond with name given in a Cpp source.
We must provide pointer to a function to bind it with a given name.
A good practise is to give document each function with a string at the end.

```cpp
// expose function myNamespace::square with name "squareInt"
// first argument is function named
// second argument is function pointer
// third (optional) argument is documentation delivered in module
def("squareInt", &myNamespace::square, "Returns a square of given number");
```
	
We may also expose specialisation of template function as python function.
Let's consider an example with function square.

```cpp
template<typename T>
T square( T val ) {
	return val*val;
}
```
	
We may expose it both for python integer and floating types, and give it diffrent names.

```cpp
// expose function square for floating point
def("squareFloat", &square<double>, "Returns a square of given number");
// expose function square for integer
def("squareInt", &square<int>, "Returns a square of given number");
```
	
### Exposing Classes

To expose a class we use *class_<Type>* objects providing methods for property, methods and member declarations.
Consider a basic example:

```cpp
#include <boost/python.hpp>
using namespace boost::python;

struct World {
	void set(std::string msg) { this->msg = msg; }
	std::string greet() { return msg; }
	std::string msg;
};

BOOST_PYTHON_MODULE(hello) {
	class_<World>("World")
		.def("greet", &World::greet)
		.def("set", &World::set)
  	;
}
```

Example python session:

```python
>>> import hello
>>> planet = hello.World()
>>> planet.set('bioweb')
>>> planet.greet()
'bioweb'
```

Each of *class_<Type>* methods returns reference to itself allowing for easy method chaining.
In example we used method *def* having identical syntax to function used for creation of function wrappers.

#### Constructors

To define constructos we use so called init-expressions.
Init expressions have syntax:

```cpp
template <T1 = unspecified,...Tn = unspecified>
struct init {
  	init(char const* doc = 0);
  	template <class Keywords>
	init(Keywords const& kw, char const* doc = 0);
  	template <class Keywords>
	init(char const* doc, Keywords const& kw);
};
```

Where Tn is n-th type of onstructor argument, and Keywords is name bindings for arguments.
To provide constructor for a class we pass init-expression to the constructor.
If class has more than one constructor we use modifier function *def*.

```cpp
template <class Init>
class_& def(Init init_expr);
```

If we do not want to expose any constructor at all, we may use *no_init*.

```cpp
class_<Abstract>("Abstract", no_init)
```

Let's see that with some examples.

- Example 1 - constructor added to class world
```cpp
#include <boost/python.hpp>
using namespace boost::python;

struct World {
	World(std::string msg): msg(msg) {} // added constructor
  	void set(std::string msg) { this->msg = msg; }
  	std::string greet() { return msg; }
  	std::string msg;
};

BOOST_PYTHON_MODULE(hello) {
	class_<World>("World", init<std::string>(args("msg"), "Constructor for class World"))
		.def("greet", &World::greet)
		.def("set", &World::set)
	;
}
```

- Example 2 - class with 3 constructors

```cpp
#include <boost/python.hpp>
using namespace boost::python;

struct Point {
	Point() : x(0), y(0) {}
	Point(int val) : x(val), y(val) {}
	Point(int x, int y) : x(x), y(y) {}
	int x;
	int y;
};

BOOST_PYTHON_MODULE(hello) {
	class_<Point>(init<>("Default constructor"))
		.def(init<int>(args("val"), "Constructor for diagonal points")
		.def(init<int, int>(args("x", "y"), "Constructor for class point")
		.def_readwrite("x", &Point::x)
		.def_readwrite("y", &Point::y)
  	;
}
```

#### Methods

Defining methods is achieved using *def* family methods and *staticmethod* function used to declare methods as static. 




#### Data members and properties


Properties could be for read only or for read&write.

```cpp
class class_ : public object {
	...
  	// property creation
  	template <class Get>
  	void add_property(char const* name, Get const& fget, char const* doc=0);
  	template <class Get, class Set>
  	void add_property(
	char const* name, Get const& fget, Set const& fset, char const* doc=0);
	...
};
```

Property could be also static

```cpp
class class_ : public object {
	...
  	// static property creation
  	template <class Get>
	void add_static_property(char const* name, Get const& fget);
	template <class Get, class Set>
	void add_static_property(char const* name, Get const& fget, Set const& fset);
	...
};
```

Lets see this in an example.

```cpp
#include <boost/python.hpp>
using namespace boost::python;

class Point {
public:
	Point(int x, int y) : x(x), y(y) { ++count; }
	~Point() { --count; )
	
	int getX() { return x; }
	int getY() { return y; }
	void setX(int val) { x = val; }
	void setY(int val) { y = val; }
	
	static int getCount() { return count; }
private:	
	int x;
	int y;
	static size_t count;
};

Point::count = 0;

BOOST_PYTHON_MODULE(hello) {
	class_<Point>(init<int, int>(args("x", "y"), "Constructor for class point")
		.add_property("readOnlyX", &Point::getX, "Read only property X")
		.add_property("X", &Point::getX, &Point::setX, "Property X")
		.add_property("readOnlyY", &Point::getY, "Read only property Y")
		.add_property("y", &Point::getY, &Point::setY, "Property Y")
		.add_static_property("pointsCount", &Point:getCount)
  	;
}
```

#### Inheritance



#### Virtual functions



## Python operators and special methods



## References, copies and limitations



### Call policy



