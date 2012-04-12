PyAutoC
=======


Introduction
------------

PyAutoC automatically wraps native C functions and structs so that they can be called from the Python/C API.

* Don't fancy the idea of hand wrapping every function and struct in your codebase?
* Don't like the look of the monster that is SWIG?
* Want a way for developers to register extra functionality at run-time?

PyAutoC is here to help.


Basic Function Usage
-------------------- 
    
```c
#include "Python.h"
#include "PyAutoC.h"

static float add_numbers(int first, float second) {
  return first + second;
}

int main(int argc, char **argv) {
  
  Py_Initialize();

  PyAutoConvert_RegisterPrimatives();
  PyAutoFunction_RegisterArgs2(add_numbers, float, int, float);

  PyObject* args = Py_BuildValue("(if)", 5, 6.13);
  PyObject* result = PyAutoFunction_Call(add_numbers, args);
  PyObject_Print(result, stdout, 0);
  
  Py_DECREF(result);
  Py_DECREF(args);
  Py_Finalize();
  
  return 0;
  
}
```
	
PyAutoC will call "add\_numbers" with values converted from their Python counterparts. It will then convert the return value back into a Python Object.

	
Basic Struct Usage
------------------ 

```c
#include "Python.h"
#include "PyAutoC.h"

typedef struct {
  float x, y, z;
} vector3;

int main(int argc, char **argv) {
	
	Py_Initialize();
	
	PyAutoConvert_RegisterPrimatives();
	
	PyAutoStruct_Register(vector3);
	PyAutoStruct_RegisterMember(vector3, x, float);
	PyAutoStruct_RegisterMember(vector3, y, float);
	PyAutoStruct_RegisterMember(vector3, z, float);

	vector3 position = {1.0f, 2.11f, 3.16f};

	PyObject* y = PyAutoStruct_Get(vector3, &position, y);
	PyObject_Print(y, stdout, 0);
	
	Py_DECREF(y);
	Py_Finalize();
	
	return 0;
}
```
	
Structs work similarly to their functional counterparts. They can be accessed at run-time and do automatic conversion of types.

Extended Usage 1
----------------

You can use PyAutoC to very quickly and easily create Python C modules for a bunch of functions such as might be done via SWIG or similar.

```c
#include "Python.h"
#include "PyAutoC.h"

static float add_numbers(int first, float second) {
  return first + second;
}

static PyObject* call(PyObject* unused, PyObject* args) {
	PyObject* func = PyTuple_GetItem(args, 0);
	PyObject* fargs = PyTuple_GetSlice(args, 1, PyTuple_Size(args));
	PyObject* ret = PyAutoFunction_CallByName(PyString_AsString(func), fargs);
	Py_DECREF(fargs);
	return ret;
}

static PyMethodDef method_table[] = {
	{"call", call, METH_VARARGS, ""},
	{NULL, NULL, 0, NULL},
};

PyMODINIT_FUNC initpyautoc_demo(void) {

	PyAutoConvert_RegisterPrimatives();
  
	PyAutoFunction_RegisterArgs2(add_numbers, float, int, float);	
	
  Py_InitModule("pyautoc_demo", method_table);
}
```

Then in Python...

```python
import pyautoc_demo
print pyautoc_demo.call("add_numbers", 5, 6.13);
```

Once you have this basic interface it is easy to intergrate more complicated and transparent APIs with some Python magic. For example...

Extended Usage 2
----------------

Because PyAutoC lets you register and access new Python functions at run time it is perfect for automatically wrapping existing C Structs as Python classes. For example by overriding "\_\_getattr\_\_" and "\_\_setattr\_\_" of a class we can easily make a Python object that behaves as if it were a C struct.

```python
import my_class

class my_python_class(object):
	def __getattr__(self, attr): return my_class.get_attr(self, attr)
	def __setattr__(self, attr, val): return my_class.set_attr(self, attr, val)
```
		
Then in the C API you can write something that looks like this.
    
```c
typedef struct {
  char* name;
  float value;
} my_class;

static PyObject* my_class_get_attr(PyObject* unused, PyObject* args) {
	PyObject* self = PyTuple_GetItem(args, 0);
	PyObject* attr = PyTuple_GetItem(args, 1);
	
	my_class* cstruct = get_instance_ptr(self);
	return PyAutoStruct_GetMember(my_class, cstruct, PyString_AsString(attr));
}

static PyObject* my_class_set_attr(PyObject* unused, PyObject* args) {
	PyObject* self = PyTuple_GetItem(args, 0);
	PyObject* attr = PyTuple_GetItem(args, 1);
	PyObject* val = PyTuple_GetItem(args, 2);
	
	my_class* cstruct = get_instance_ptr(self);
	PyAutoStruct_SetMember(my_class, cstruct, PyString_AsString(attr), val);
	Py_RETURN_NONE;
}

static PyMethodDef method_table[] = {
	{"get_attr", my_class_get_attr, METH_VARARGS, ""},
	{"set_attr", my_class_set_attr, METH_VARARGS, ""},
	{NULL, NULL, 0, NULL},
};

PyAutoStruct_Register(my_class);
PyAutoStruct_RegisterMember(my_class, name, char*);
PyAutoStruct_RegisterMember(my_class, value, float);

Py_InitModule("my_class", method_table);
```

The above python object will then act as if it were a "my\_class" struct and on attribute accesses to "name" or "value" will return appropriate python objects. A lot less work than writing a bunch of getters and setters!

The "get\_instance\_ptr" function is left for you to implement and there are lots of options. The idea is that somehow the python instance should tell you how to get a pointer to the actual struct instance in C which it represents. One option is to store C pointers in the python instance using something like "PyCObject\_FromVoidPtr". An alternative I like to just store a string in the python instance which uniquely identifies it. Once you have this in C it is possible to just look this string up in a dictionary or similar to find the actual pointer.

The above technique can also be easily extended so that as well as members, the class is able to look up and execute methods too!

If you use "PyAutoStruct\_GetMember\_TypeId" or "PyAutoStruct\_SetMember\_TypeId" you can even extend the above to work for arbritary structs/classes. For this to work you need to somehow get a PyAutoType value. This can be found by feeding a string into "PyAutoType\_Register".

The "PyAutoType\_Register" function is a simple function which gives a unique identifier to each new string it encounters. This means that if you give it a string of a previously registered data type it will return a matching Id. One trick I like it to use is to feed into it the ".\_\_class\_\_.\_\_name\_\_" property of a python instance. This means that I can create a new python class with overwritten "\_\_getattr\_\_" and "\_\_setattr\_\_" it will automatically act like the corrisponding C struct with the same name.

New Argument Types
------------------

To call functions or access struct members which have non-primitive types it is possible to register your own conversion functions.

```c
typedef struct {
	int x, y;
} pair;

static PyObject* convert_from_pair(void* data) {
	pair p = *(pair*)data;
	return Py_BuildValue("(ii)", p.x, p.y);
}

void convert_to_pair(PyObject* pyobj, void* out) {
	pair* p = (pair*)out;
	p->x = PyInt_AsLong(PyTuple_GetItem(pyobj, 0));
	p->y = PyInt_AsLong(PyTuple_GetItem(pyobj, 1));
}

PyAutoConvert_Register(pair, convert_from_pair, convert_to_pair);
```

Now it is possible to call any functions with 'pair' as an argument or return type and PyAutoC will handle any conversions automatically. You can also use the registered functions directly in your code by using PyAutoConvert.

```c
pair p = {1, 2};
PyObject* pypair = PyAutoConvert_From(pair, &p);  
```
	
Issues
------

* Using PyAutoC for functions creates a small memory and performance overhead. This is because it duplicates much of the process involved in managing the stack such as copying stack data. Because most of the logic happens at run-time it also uses a lot of function pointers. These cannot be optimised and inlined very easily so processes such as converting lots of Python data to C data can be slower than if the process is declared statically. Still, if you are wrapping with a scripting language like python then perhaps it is less of a concern.

* The function registration macros are a little verbose though this is unavoidable. Always remember that the argument count must be specified in the name and also if the function returns void.

```c
PyAutoFunction_RegisterArgs2(add_numbers, float, int, float);
PyAutoFunction_RegisterVoidArgs3(add_numbers_message, void, char*, int, float);
```
	
* There is hard coded limit to many aspects such as number of conversion functions which can be registered or maximum number of arguments in registered functions. This is something I plan to change in consecutive versions but for now the limits can be increased with some simple edits to the source and recompiling.
