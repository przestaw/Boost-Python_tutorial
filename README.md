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
	def("square", square, "Returns a square of given number");
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
A good practise is to give document each function with a docstring.

```cpp
template <class Fn>
void def(char const* name, Fn fn);

template <class Fn, class A1>
void def(char const* name, Fn fn, A1 const&);

template <class Fn, class A1, class A2>
void def(char const* name, Fn fn, A1 const&, A2 const&);

template <class Fn, class A1, class A2, class A3>
void def(char const* name, Fn fn, A1 const&, A2 const&, A3 const&);
```

Functions from *def* family takes 2 to 5 arguments.
- *name* - function name in python convention
- *fn* - function pointer
- a1-a3 - docstring, keywords or policies in any possible order

Docstring is a brief documentation of a function, similar to a docstring in a first line of a python function.
Keywords object holds a sequence of arguments names, and whose type encodes the number of keywords specified. The keyword-expression may contain default values for some or all of the keywords it holds.

An example of keywords usage:

```cpp
#include <boost/python/def.hpp>
using namespace boost::python;

int foo(double x, double y, double z=0.0, double w=1.0);

BOOST_PYTHON_MODULE(xxx)
{
   def("foo", foo
            , ( arg("x"), "y", arg("z")=0.0, arg("w")=1.0 ) 
            );
}
```

Another example with our square function:

```cpp
// expose function myNamespace::square with name "squareInt"
// first argument is function named
// second argument is function pointer
// third (optional) argument is documentation delivered in module
// fourth (optional) argument is argument name
def("squareInt", &myNamespace::square, "Returns a square of given number", arg("number"));
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
def("squareFloat", square<double>, "Returns a square of given number");
// expose function square for integer
def("squareInt", square<int>, "Returns a square of given number");
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
Syntax and behaviour in similar way to function definition with the diffrence chaining of *def* calls to define multiple methods.

```cpp
#include <boost/python.hpp>
using namespace boost::python;

class class_ : public object {
	...
    // defining methods
    template <class F>
    class_& def(char const* name, F f);
    template <class Fn, class A1>
    class_& def(char const* name, Fn fn, A1 const&);
    template <class Fn, class A1, class A2>
    class_& def(char const* name, Fn fn, A1 const&, A2 const&);
    template <class Fn, class A1, class A2, class A3>
    class_& def(char const* name, Fn fn, A1 const&, A2 const&, A3 const&);

    // declaring method as static
    class_& staticmethod(char const* name);
    ...
};
```

Simple example of defining methods

```cpp
#include <boost/python.hpp>
using namespace boost::python;

// Simple class
class hello {
    public:
        hello(const std::string& country) { this->country = country; }
        std::string greet() const { return "Hello from " + country; }
    private:
        std::string country;
};

// A function taking a hello object as an argument.
std::string invite(const hello& w) {
    return w.greet() + "! Please come soon!";
}

BOOST_PYTHON_MODULE(hello) {
    class_<hello>("hello", init<std::string>())
        .def("greet", &hello::greet)  // Add a regular member function.
        .def("invite", invite)  // Add invite() as a regular function to the module.
    ;

    def("invite", invite); // invite() can also be made a member of module!!!
}
```

Then we can use our methods and functions in python like so:

```python
>>> from getting_started2 import *
>>> hi = hello('Poland')
>>> hi.greet()
'Hello from Poland'
>>> invite(hi)
'Hello from Poland! Please come soon!'
>>> hi.invite()
'Hello from Poland! Please come soon!'
```

#### Data members and properties

Data members may also be exposed to Python so that they can be accessed as attributes of the corresponding Python class. Each data member that we wish to be exposed may be read-only or read-write. Members and static members can be exposed using *def_readonly* and *def_readwrite* methods.

```cpp
class class_ : public object {
	...
	// exposing data members
	template <class D>
	class_& def_readonly(char const* name, D T::*pm);
	template <class D>
	class_& def_readwrite(char const* name, D T::*pm);
	// exposing static data members
	template <class D>
	class_& def_readonly(char const* name, D const& d);
	template <class D>
	class_& def_readwrite(char const* name, D& d);
	...
};
```

Consider example - point with members x and y, counting instances.

```cpp
#include <boost/python.hpp>
using namespace boost::python;

struct Point {
	Point(int x, int y) : x(x), y(y) { ++count; }
	~Point() { --count; )
	int x;
	int y;
	static size_t count;
};

Point::count = 0;

BOOST_PYTHON_MODULE(hello) {
	class_<Point>(init<int, int>(args("x", "y"), "Constructor for class point")
		.def_readwrite("x", &Point::x)
		.def_readwrite("y", &Point::y)
		.def_readonly("pointsCount", &Point:count)
  	;
}
```

In C++, classes with public data members are usually frowned upon. 
Well designed classes that take advantage of encapsulation hide the class data members. 
The only way to access the class data is through access (getter/setter) functions. 
Access functions expose class properties. 
However, in Python attribute access is fine, it does not neccessarily break encapsulation to let users handle attributes directly, because the attributes can just be a different syntax for a method call.

```cpp
class class_ : public object {
	...
  	// property creation
  	template <class Get>
  	void add_property(char const* name, Get const& fget, char const* doc=0);
  	template <class Get, class Set>
  	void add_property(
	char const* name, Get const& fget, Set const& fset, char const* doc=0);
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
		.add_property("x", &Point::getX, &Point::setX, "Property X")
		.add_property("readOnlyY", &Point::getY, "Read only property Y")
		.add_property("y", &Point::getY, &Point::setY, "Property Y")
		.add_static_property("pointsCount", &Point:getCount)
  	;
}
```

#### Inheritance

## TODO

#### Virtual functions

## TODO

#### Copy policy & smart pointers

## TODO

## Python operators and special methods

C++ has a lot of well defined operators and allows operator overloading for new types.
Boost.Python takes advantage of this and makes it easy to wrap C++ operator-powered classes.

Consider a file position class FilePos and a set of operators that take on FilePos instances:

```cpp
class FilePos { /*...*/ };

FilePos     operator+(FilePos, int);
FilePos     operator+(int, FilePos);
int         operator-(FilePos, FilePos);
FilePos     operator-(FilePos, int);
FilePos&    operator+=(FilePos&, int);
bool        operator<(FilePos, FilePos);
```

The class and the various operators can be mapped to Python rather easily and intuitively.
The code snippet above is very clear and needs almost no explanation at all. 
It is virtually the same as the operators signatures. 
Just take note that self refers to FilePos object. 

```cpp
class_<FilePos>("FilePos")
    .def(self + int())          // __add__
    .def(int() + self)          // __radd__
    .def(self - self)           // __sub__
    .def(self - int())          // __sub__
    .def(self += int())         // __iadd__
	.def(self -= int())			// __isub__
    .def(self < self);          // __lt__
```

Python has a few Special Methods - like for example *str()*. 
Boost.Python supports all of the standard special method names supported by real Python class instances.
A similar set of intuitive interfaces can also be used to wrap C++ functions that correspond to these Python special functions. 

Example:

```cpp
class Rational
{ public: operator double() const; };

Rational pow(Rational, Rational);
Rational abs(Rational);
ostream& operator<<(ostream&,Rational);

BOOST_PYTHON_MODULE(hello) {
	class_<Rational>("Rational")
    	.def(float_(self))                  // __float__
    	.def(pow(self, other<Rational>))    // __pow__
    	.def(abs(self))                     // __abs__
    	.def(str(self))                     // __str__
    ;
}
```

Please note that *operator<<* is used by the method defined by def(str(self))

## References, copies and limitations

## TODO

### Call policy

## TODO

###### Sources

1. [Reference Manual for Boost.Python](https://www.boost.org/doc/libs/1_64_0/libs/python/doc/html/reference/)
2. [*Building Hybrid Systems with Boost.Python*](https://www.boost.org/doc/libs/1_64_0/libs/python/doc/html/article.html)
3. [Boost.Python sources](https://github.com/boostorg/python)
