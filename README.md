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
Also if provided class is abstract [pure virtual] we need tell it that class is noncopyable, Boost.Python tries to register a converter for handling wrapped functions which handle function returning values of class type. 
Naturally, this has to be able to copy construct the returned C++ class object into storage that can be managed by a Python object. 
Since this is an abstract class, that would fail. 

```cpp
class_<Abstract, boost::noncopyable>("Abstract", no_init)
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

Boost.Python extension classes support single and multiple-inheritance in Python, just like regular Python classes. 
We can arbitrarily mix built-in Python classes with extension classes in a derived class' tuple of bases. 
Whenever a Boost.Python extension class is among the bases for a new class in Python, the result is an extension class.
In module definition we use template argument *bases<...>* to pass information about inheritance to *class_*.

Example :
```cpp
struct Base { virtual ~Base(); };
struct Derived : Base {};

void b(Base*);
void d(Derived*);
Base* factory() { return new Derived; }

BOOST_PYTHON_MODULE(inheritance) {
    class_<Base>("Base")
    /*...*/
    ;
    class_<Derived, bases<Base> >("Derived")
    /*...*/
    ;
    def("b", b);
    def("d", d);
    // Tell Python to take ownership of factory's result
    def("factory", factory, return_value_policy<manage_new_object>());
}
```

#### Virtual functions

Bosst.Python classes can behave polymorphically through virtual functions.
Consider *Base* class extending previous example.

```cpp
struct Base {
    virtual ~Base() {}
    virtual int f() = 0;
};
```

One of the goals of Boost.Python is to be minimally intrusive on an existing C++ design. 
In principle, it should be possible to expose the interface for a 3rd party library without changing it. 
It is not ideal to add anything to our class Base. 
Yet, when we have a virtual function that's going to be overridden in Python and called polymorphically from C++, we'll need to add some scaffoldings to make things work properly. 
What we'll do is write a class wrapper that derives from Base that will unintrusively hook into the virtual functions so that a Python override may be called: 

```cpp
struct BaseWrap : Base, wrapper<Base> {
    int f() {
        return this->get_override("f")();
    }
};
```

Notice too that in addition to inheriting from Base, we also multiply inherited *wrapper<Base>*. 
The wrapper template makes the job of wrapping classes that are meant to overridden in Python, easier.
We signal that our class is pure virtual using *pure_virtual*.

```cpp
class_<BaseWrap, boost::noncopyable>("Base")
    .def("f", pure_virtual(&Base::f))
;
```

But what if *f()* was not pure virtual?

```cpp
struct Base {
    virtual ~Base() {}
    virtual int f() { return 0; }
};
```

We need to construct our wrapper to handle default implementation.

```cpp
struct BaseWrap : Base, wrapper<Base>
{
    int f()
    {
        if (override f = this->get_override("f")) {
            return f(); 
		} else { 
        	return Base::f();
		}
    }

    int default_f() { return this->Base::f(); }
};
```

Take note that we expose both *&Base::f* and *&BaseWrap::default_f*. 
Boost.Python needs to keep track of 
	1. the dispatch function f
	2. the forwarding function to its default implementation default_f.
There's a special def function for this purpose. 

```cpp
class_<BaseWrap, boost::noncopyable>("Base")
    .def("f", &Base::f, &BaseWrap::default_f)
;
```

In Python, the results would be as expected: 

```python
>>> base = Base()
>>> class Derived(Base):
...     def f(self):
...         return 42
...
>>> derived = Derived()
>>> base.f()
0
>>> derived.f()
42
```

### Call policy

*CallPolicy* allows boost.python to deal with raw references and pointers. 
Different policies specifies different strategies of managing object ownership. 

#### return_internal_reference

Builds a Python object around a pointer to the C++ result object (which must have a *class_<>* wrapper somewhere), and applies some lifetime management to keep the "self" object alive as long as the Python result is alive. NULL pointer returning as None. 

#### return_value_policy<reference_existing_object>

naïve (and dangerous) approach

When the wrapped function is called, the value referenced by its return value is not copied.
A new Python object is created which contains an unowned U* pointer to the referent of the wrapped function's return value, and no attempt is made to ensure that the lifetime of the referent is at least as long as that of the corresponding Python object.

This class is used in the implementation of return_internal_reference. Also NULL pointer returning as None. 

#### return_value_policy<manage_new_object>

Can be used to wrap C++ functions returning a pointer to an object allocated with a *new-expression* and expecting the caller to take responsibility for deleting that C++ object from heap. 
Boost.Python will do it as part of Python object destruction. 

```cpp
T* factory() { return new T(); }

class_<T>("T");

def("Tfactory", factory, return_value_policy<manage_new_object>() );
```

#### with_custodian_and_ward<M,N>

Keeps N-th argument as long as M-th is alive.

Use of template parameters M,N:

    1 - 1st argument (self for method calls)
    2 - 2nd argument (1st for method calls)
    ... 

For example, container operation append usualy uses with_custodian_and_ward<1,2> which means keep argument alive while container itself is alive.

#### with_custodian_and_ward_postcall<M,N>

ties lifetimes of the arguments and results

M,N same as before but also you can use 0 - result 

#### Memory & smart pointers

Since Python handles memory allocation and garbage collection automatically, the concept of a "pointer" is not meaningful in Python. 
However, many C++ APIs expose either raw pointers or shared pointers.

##### Raw pointers

The lifetime of C++ objects created by *new* A can be handled by Python's garbage collection by using the *manage_new_object* return policy: 

```cpp
struct A {
    static A*   create () { return new A; }
    std::string hello  () { return "Hello, is there anybody in there?"; }
};

BOOST_PYTHON_MODULE(pointer) {
    class_<A>("A",no_init)
        .def("create",&A::create, return_value_policy<manage_new_object>())
        .staticmethod("create")
        .def("hello",&A::hello)
	;
}
```

#### Smart pointers

The usage of smart pointers (e.g. boost::shared_ptr<T>) is another common way to give away ownership of objects in C++. 
These kinds of smart pointer are automatically handled if you declare their existence when declaring the class to boost::python. 
This is done by including the holding type as a template parameter to class_<>, like in the following example: 

```cpp
#include <string>
#include <boost/shared_ptr.hpp>
#include <boost/python.hpp>

using namespace boost;
using namespace std;
using namespace boost::python;

struct A {
    static shared_ptr<A> create () { return shared_ptr<A>(new A); }
    std::string   hello  () { return "Just nod if you can hear me!"; }
};

BOOST_PYTHON_MODULE(shared_ptr)
{
    class_<A, shared_ptr<A> >("A",init<>())
        .def("create",&A::create )
        .staticmethod("create")
        .def("hello",&A::hello)
    ;
}
```

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
    .def(self -= int())	        // __isub__
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

## Retrurning lists & tuples

Boost.python supports python types tuple and lists.

- *boost::python::tuple* can be created using constructor or *make_tuple(args)* call. All C++ types are supported, we can return our own classed if there was a *class_<>* wrapper creted.
- *boost::python::list* can hold elements of any type. Lists support common operations like similarly to tuples all C++ types are supported.
	- lists support operations like : *append, extend, insert, count, pop, remove*

More details about lists and tuples can be found in [Reference Manual for Boost.Python](https://www.boost.org/doc/libs/1_66_0/libs/python/doc/html/reference/)

## Build

Standard way to build boost.python program is to use bjam, used also to build boost.python itself.
Below are examples of using SCons and CMake build systems to build example aplication.

### SCons

```python
BOOST_VERSION = 'boost.cvs'
BOOST = '/usr/local/src/' + BOOST_VERSION
BOOSTLIBPATH = BOOST+'/stage/lib'
env = Environment (LIBPATH=['./',BOOSTLIBPATH], CPPPATH=[BOOST, '/usr/include/python'],  
                   RPATH=['./',BOOSTLIBPATH])
env.SharedLibrary (target='uvector', source='uvector.cc', SHLIBPREFIX='', LIBS=[BOOST_PYTHON_LIB])
```

### CMake

```cmake
cmake_minimum_required(VERSION 3.5)

SET(ENV{BOOST_ROOT} "/path/to/my/boost") # set if find_pacgae fails with default paths

find_package(Boost COMPONENTS python REQUIRED)
find_package(PythonLibs 3.6 REQUIRED)

INCLUDE_DIRECTORIES(${Boost_INCLUDE_DIRS})
INCLUDE_DIRECTORIES(${PYTHON_INCLUDE_DIRS})

ADD_LIBRARY(MyLibrary SHARED MyLibraryInterface.cpp)
TARGET_LINK_LIBRARIES(MyLibrary "Boost::python" ${PYTHON_LIBRARIES})
```

##### Sources & further reading

1. [Reference Manual for Boost.Python](https://www.boost.org/doc/libs/1_66_0/libs/python/doc/html/reference/)
2. [*Building Hybrid Systems with Boost.Python*](https://www.boost.org/doc/libs/1_66_0/libs/python/doc/html/article.html)
3. [Boost.Python sources](https://github.com/boostorg/python)
4. [Detailed CallPolicy](https://www.boost.org/doc/libs/1_66_0/libs/python/doc/html/reference/function_invocation_and_creation/models_of_callpolicies.html)
