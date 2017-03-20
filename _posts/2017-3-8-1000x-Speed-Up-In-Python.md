---
layout: post
title: "How I Got 1000x Speed Up In Python"
categories: [python, optimization]
tags: [python, cython, C, pypi, pip]
image:
  feature: Python1000x.png
  teaser: Python1000x.png
---

# Synopsis
I detail my journey of optimizing a component of my python program and how it resulted in a speed up factor of 1000. I explore how to build and maintain a Python-C extension across multiple versions of python and within package managers like Pypi.

---

# Background
Python is great, development time is fast, errors are usually sensible, and most importantly there are great package managers with tens of thousands of supported modules. These are just some of the reasons why it’s my weapon of choice when rapid development is required. 
However sometimes you’ll require a specific component that needs to run extremely fast, which is not something pyhon does particularly well. When building [khet-online](http://khet-online.com), an online version of a chess-like game, I had such a problem. 
The AI for the game had poor performance because the native python could only explore ~1000 moves per second, whereas most chess AI’s require upwards of one million per second. 

---

# Path to optimization
The first step to any optimization is to first profile. In my case I used Pycharm's [default profiler](https://docs.python.org/2/library/profile.html). <br/><br/>
From there I improved my initial implementation by using better data structures IE numpy byte arrays over python dictionaries. I also got significant gains by using python 3.x over the python 2.x I had been using previously. <br/><br/>
All this effort equated to about a 2.6x speed up or about ~2500 moves per second. This is still much too slow to be useful, especially when I had no intension on developing a quality heuristic, which may have made the low move exploration workable.
<br/><br/>
The specific component I wanted to optimize was basically composed of simple for loops and lots of branching. <br/><br/>
In fact the code was composed of ~77 if conditions per evaluation of move. 
Of course I removed as many conditions as possible when, but the nature of generating valid board moves means one must actually validate if a configuration is valid, and unlike checkers, Khet has a lot of rules…. <br/><br/>
A sample logic snippet sits below: <br/><br/>
```python
# Code snippet describing how a game mechanic interacts with a specific piece
if piece == _pyramid:
	if light_direction == _up:
		if orientation == _down:
			return _left
		elif orientation == _right:
			return _right
		else:
			return None
	elif light_direction == _down:
		if orientation == _left:
			return _left
		elif orientation == _up:
			return _right
		else:
			return None
	elif light_direction == _left:
		if orientation == _right:
			return _down
		elif orientation == _up:
			return _up
		else:
			return None
	elif light_direction == _right:
		if orientation == _down:
			return _down
		elif orientation == _left:
			return _up
		else:
			return None
# ... 
```


I had a few options on how to optimize my python
1. [LLVM compiler](http://numba.pydata.org/)
2. [Better interpreter (Iron Python)](http://ironpython.net/)
3. [Python Extension (“C” binding)](https://docs.python.org/2/extending/extending.html)

Option (2) was out because I was deploying on cloud environments like elastic bean stalk and google app engine (GAE) where I would have no control of the interpreter being used.
<br/><br/>
In the end I chose option (3) because everyone that went through with the implementation saw 100x+ improvement in speed. 
It's quite possible that the LLVM compiler might have been an easier route and was certainly worth a bit more exploration.

---

# Building a python module, in C?
So my research indicated I would see something along the lines of a 100x speed up if I just reimplemented the exact same code in C. Geared with nothing but [docs.python.org](https://docs.python.org/2/extending/extending.html) I got to work reimplementing all that I had written in python to also run in C. 
<br/><br/>
The down side of this method really started to show as it took a lot of effort to reimplement all the logic I so carefully crafted in python. If you follow the similar path you’ll realize just how much python handles for you. Keeping a counter for every runtime allocated array is not so much fun. Also unlike the nice exceptions in python, C just gives you the infamous segfault or worse yet the silent but deadly memory leak. 
<br/><br/>
You also have to consider the additional logic required to setup the interface between C and your module.
<br/>
I kept my setup as simple as possible exposing only one function that took four parameters:
1.	Color of moving player 
2.	The board state (in my case a list of integers)
3.  The minimum depth of the possible move search tree
4.	How many moves should be explored 

---

## Setting up a python extension
A python extension makes a compiled C function callable from python. 
<br/><br/>
To do this one can use [setup tools](http://setuptools.readthedocs.io/en/latest/setuptools.html), which will basically compile the provided C code and make it usable from python. 

<br/>
<div style="text-align:center">
<img src="/images/PythonExtensionScheme.png" alt="Python-SetupTools-C"/>
</div>
The setup.py will look akin to the following:
```python
from setuptools import setup, Extension
# module name = khetsearch
# 
test_module = Extension('demo',
                        sources=['demo.c'],
                        include_dirs=['<path to code>'])

# add C extension module to be built (technically defined as a dependency)
setup (name = 'YourPackage',
       version = '1.0',
       description = 'This is a demo package',
       ext_modules = [test_module])
```

Our C file defines all the functions that needs to be exposed to python:
```c
#include <Python.h>

static char demo_docs[] =
    "Pass character array as board representation and number of iteration(int)\n";

static PyMethodDef demo_funcs[] = {
    {"search", (PyCFunction)search,
     METH_VARARGS, demo_docs},
    {NULL}
};

// init<your module name>
void initdemo(void)
{
    Py_InitModule3("demo", demo_funcs, "Extension module example!");
}
```

Here the keyword "demo" is looked for as we defined that as the module name...


The C interface for a exposed function ends up looking like the following:
```c
/***
* Python-C entry point (provided method)
*/
static PyObject * search(PyObject* self, PyObject *args){
    // defining variables
    PyObject *l;
    const char * board;
    struct MoveResults mr;
    struct MoveRating r;
    char actualBoard[80];
    PyObject * listObj;
    // ...
	
    // Get parameters from python
    // b -> boolean (moving player)
    // O -> python object (in this case a list of ints)
    // i -> integer (min depth of search tree / # of iterations)
    if (!PyArg_ParseTuple(args, "bOii", &color, &listObj, &minDepth, &iterations)){
        return NULL;
    }

    // Board has 80 squares. Cast ints to chars
    for(i=0;i<80;i++){
        tmp = (char)PyInt_AsLong(PyList_GetItem(listObj, i));
        actualBoard[i] = tmp;
    }

    // Do the search and return move ratings
    mr = getMoveRatings(color, actualBoard,minDepth,iterations);

    // Allocate a new python list for the return value
    l = PyList_New(mr.numMoves*2);// {Move, Move Rating}

    // Copy values into python array
    i=0;
    while (i<mr.numMoves){
        r = mr.moves[i];
        PyList_SET_ITEM(l, i*2, PyInt_FromLong((long)r.moveValue));
        PyList_SET_ITEM(l, i*2 + 1, PyFloat_FromDouble(r.moveRating));
        i++;
    }

    // Memory Clean Up
    free(mr.moves);
	
	// Return python list
    return l;
}
```

Once the setup is run (**python setup.py**) the code can be run by:
```python
import demo
demo.search(numeric_color, numeric_board, min_depth, max_evaluations)
```

This methodology should work for adding your module to pypi. However be wary that didn't systems may use different compilers, I went the most cautious approach and implemented everything to be compatible with C89.
<br/><br/>
[A general tutorial for submitting a package to pypi](http://peterdowns.com/posts/first-time-with-pypi.html).

---

# Supporting multiple versions of python
Now you have running code but you want to support multiple versions of python of course. 
This ends up being tricky because between python 2 and 3 they actually changed the C api, especially around operations of initializing an application
<br/><br/>
For me the quickest hack was the following:
1. Map python 2 api functions to their python 3 equivalents
2. Conditionally define the init function based on the python version

```c
/// Python 3 uses different api, map the python 3 functions to the python 2 names ///
#if PY_MAJOR_VERSION >= 3
  #define PyIntObject                  PyLongObject
  #define PyInt_Type                   PyLong_Type
  #define PyInt_Check(op)              PyLong_Check(op)
  #define PyInt_CheckExact(op)         PyLong_CheckExact(op)
  #define PyInt_FromString             PyLong_FromString
  #define PyInt_FromUnicode            PyLong_FromUnicode
  #define PyInt_FromLong               PyLong_FromLong
  #define PyInt_FromSize_t             PyLong_FromSize_t
  #define PyInt_FromSsize_t            PyLong_FromSsize_t
  #define PyInt_AsLong                 PyLong_AsLong
  #define PyInt_AS_LONG                PyLong_AS_LONG
  #define PyInt_AsSsize_t              PyLong_AsSsize_t
  #define PyInt_AsUnsignedLongMask     PyLong_AsUnsignedLongMask
  #define PyInt_AsUnsignedLongLongMask PyLong_AsUnsignedLongLongMask
#endif

/// Python 3 uses different initialization structures... ///
#if PY_MAJOR_VERSION >= 3
	// Python 3 init function
    static struct PyModuleDef demoModule = {
       PyModuleDef_HEAD_INIT,
       "demo",   /* name of module */
       demo_docs, /* module documentation, may be NULL */
       -1,       /* size of per-interpreter state of the module,
                    or -1 if the module keeps state in global variables. */
       demo_funcs
    };

    PyMODINIT_FUNC PyInit_demo(void)
    {
        return PyModule_Create(&demoModule);
    }


#else
	// python 2 init function
	void initdemo(void)
	{
		Py_InitModule3("demo", demo_funcs, "Extension module example!");
	}

#endif
```

With tools like [CI Travis](https://travis-ci.org/) (I’m a big fan ) you can quickly test if your module works across multiple version of python. 

---

# Results
C performs branching and looping at rates significantly faster than that of native python. 
In my case I saw a speed up of ~1000x, the new bottle neck became related to memory instead of processing speed.
<br/>
<div style="text-align:center">
<img src="/images/SearchTree.gif" alt="example search tree"/>
</div>
<br/>
Even though my C implementation was probably around ~10x more space efficient in storing move information I still ran into memory problems because of the sheer magnitude of additional moves that could be explored.
<br/><br/>
So there you go, if you optimize the runtime speed enough you might run into space issues...
