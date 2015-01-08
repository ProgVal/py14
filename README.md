# The worlds poorest JIT compiler for Python

This is a little experiment that shows how far you can go with the C++ 14 `auto` return type and templates.  Just add the `@cpp` to your Python function and on the next call the following will happen:

1. Python function is transpiled into C++ 14 template.
2. Types of the arguments the function is being called with are traced.
3. C++ will be compiled for the traced types
4. Python bindings are generated with the help of [boost python](http://www.boost.org/doc/libs/1_57_0/libs/python/doc/index.html)
5. On subsequent calls the C++ version of the function is called

## How it works

Consider the worlds poorest fibonacci implementation.

```python
import sys
from poorjit import cpp

@cpp
def fib(n):
    if n == 1:
        return 1
    elif n == 0:
        return 0
    else:
        return fib(n-1) + fib(n-2)

if __name__ == "__main__":
    n = int(sys.argv[1])
    print(fib(n))
```

Generating the C++ code for a Python function does not require knowing
about types -  as long as you generate a template.

The above `fib` function can be represented as the following template.

```c++
template <typename T1>
auto fib(T1 n) {
    if (n == 1) {
        return 1;
    } else if (n == 0) {
        return 0;
    } else {
        return fib(n-1) + fib(n-2);
    }
}
```

The only time we have to know the types is when we are compiling and binding to the Python code.
The `@cpp` decorator therefore records the types the function is called with and
generates the suitable bindings.

```c++
BOOST_PYTHON_MODULE(fib_extern) {
    boost::python::def("fib", fib<int>);
}
```

## Trying it out

Requirements:

- [boost python 1.55](http://www.boost.org/doc/libs/1_57_0/libs/python/doc/index.html)
- clang 3.5
- clang-format 3.5

It will be difficult to get this to work on your machine.
I therefore recommend using Docker.

Build the provided dockerfile.

```
sudo docker build -t poorjit .
```

And then run the sample `fib.py`.

```
docker run -v $(pwd):/root poorjit python fib.py 42
```

## In the future...

It should be possible to compile the C++ function for all the types
the function is called with. Consider the following `add` method.

```python
def add(a, b):
    return a + b

if __name__ == "__main__":
    print(add(10, 3))
    print(add("Hello", "World"))
```

On the first call to `add` we compile the method for `int` and on the second call
a version for `int` and `str`.

```
BOOST_PYTHON_MODULE(fib_extern) {
    boost::python::def("fib_int", fib<int>);
    boost::python::def("fib_str", fib<std::string>);
}
```

The decorator then chooses the right C++ method to call for you.
