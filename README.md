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



#### Constructors



#### Methods



#### Data members and properties



#### Inheritance



#### Virtual functions



## Python operators and special methods



## References, copies and limitations



### Call policy



