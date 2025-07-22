---
layout: post
title:  PyNumberMethods
date:   2025-07-21 10:51:00 -0400
categories: C-Extentions
---

In Python, it's almost trivial to create an object that overrides the "+" operator using the so-called dunder/magic methods. 
{% highlight Python %}
class MyCoolObject:
    def __add__(self, other):
        ...
    def __radd__(self, other):
        #if you're feeling fancy ;)
        ...
{% endhighlight %}
However, manually adding each addition magic method like the above can be tedious, and instead some people will implement these methods on the fly using an "operator fallback" type recipe. This method relies on type checking, and will ensure that for each time an operator is chosen, the correct functions will be used to handle it. 
{% highlight Python %}
class MyCoolFancyObject:
    def _generic_FancyObject_addition(a, b):
        ...
    def _operator_fallbacks(monomorphic_operator, fallback_operator):
        def forward(a, b):
            ...
        def reverse(b, a):
            ...
        return forward, reverse
    __add__, __radd__ = _operator_fallbacks(_generic_FancyObject_addition, operator.add)
{% endhighlight %}
The monomorphic operator handles the case when both objects have the same type. The fallback operator will be when the objects have mixed types. The forward and reverse functions in the fallback function do the job of checking the types of their arguments and determining if the monomorphic or fallback operator should be used for their supplied arguments. Despite the visual complexity of the above code, it still is largely boilerplate for many projects. Likewise, it is *very* pythonic. Look at those first class functions being passed around! Check out the nested functions! 

This functionality can still be implemented in C with an extension module. Unfortunately, the code is much less attractive. In the object model for the C api, a typical Python class looks something like the below:

{% highlight C %}
static PyTypeObject MyAwesomeCObject{
	.ob_base = PyVarObject_HEAD_INIT(NULL, 0) //macro written by wizards
	.tp_name = "PACKAGE_NAME.MODULE_OBJECT_NAME", //MyAwesomeCObject.MyAwesomeCObject
	.tp_doc = PyDoc_STR("The documentation for my awesome extension"),
	.tp_itemsize = 0, //dangerous
	.tp_flags = Py_TPFLAGS_DEFAULT,
	.tp_new = MyFraction_new, //create a new instance of a class like __new__ in Python 
	.tp_init = (initproc)MyAwesomeCObject_init, //same as __init__ 
	.tp_dealloc = (destructor)MyAwesomeCObject_dealloc, //magical function invoked by the garbage collector (SIMILAR BUT DISTINCT FROM __del__ in regular classes)
	.tp_members = MyAwesomeCObject_members, //the regular attributes of a class, but they can be C-types too
	.tp_methods = MyAwesomeCObject_methods, //the functions in a class that a person can call in Python
	.tp_getset = MyAwesomeCObject_getsetlist, //stuff like __get__ and being able to add functionality like with @property in Python
	.tp_as_number = &MyFraction_num_methods, // __add__, __radd__, ..., __matmul__ everything math
	.tp_repr = (reprfunc)MyFraction_repr, //repr(..) 
};
{% endhighlight %}

The darling little attribute "tp_as_number" holds a reference to a structure that holds methods that tell the Python interpreter how to treat a C object like a number. 

{% highlight C %}
static PyNumberMethods MyAwesomeCObject_number_methods = {
	.nb_add = (binaryfunc)MyAwesome_addition_function,
	.nb_subtract = (binaryfunc)MyAwesome_subtraction_function,
    ...
};
{% endhighlight %}

The above snippet shows the PyNymberMethods struct that has 2 functions in it to handle addition and subtraction. Both are cast to the type of "binaryfunc", which is because "+" and "-" are binary operators, as in they take 2 arguments. These two functions have to handle both monomorphic and fallback operators. The definition for "MyAwesome_addition_function" might look like the below:

{% highlight C %}
static PyObject* 
MyAwesome_addition_function(PyObject* a, PyObject* b){
};

{% endhighlight %}
The arguments to the function are both pointers to PyObject, references to some object that the compiler does not yet know the types of. 

Just like in the "fancy" Python code, the types of the arguments have to be checked to invoke the right functions. This is not the best way to handle this process, but I can at least say the below code will work.  

{% highlight C %}
static PyObject*
MyAwesome_Monomorphic_Addtion_Operator(MyAwesomeCObject* a, MyAwesomeCObject* b)
{
    ... //handle garbage collection of course
    return (PyObject*)WhateverReturnValue;
}
static PyObject*
MyAwesome_Fallback_Addtion_Operator(MyAwesomeCObject* a, PyObject* b)
{

    ... //handle garbage collection of course
    return (PyObject*)WhateverReturnValue;
}
static PyObject* 
MyAwesome_addition_function(PyObject* a, PyObject* b)
{
    if ( Py_IS_TYPE(a, &MyAwesomeCObjectType) && Py_IS_TYPE(b, &MyAwesomeCObjectType) ){
        MyAwesomeCObject* _a = (MyAwesomeCObject*)a; 
        MyAwesomeCObject* _b = (MyAwesomeCObject*)b;
        return MyAwesome_Monomorphic_Addtion_Operator(_a, _b);
    }
    else if ( Py_IS_TYPE(a, &MyAwesomeCObjectType) && Py_IS_TYPE(b, &SomeOtherTypeLol) ) {
        MyAwesomeCObject* _a = (MyAwesomeCObject*)a;
        return MyAwesome_Fallback_Addtion_Operator(_a, b);
    }
    else if (Py_IS_TYPE(a, &MyAwesomeCObjectType) && Py_IS_TYPE(b, &SomeOtherTypeLol) ) {
        MyAwesomeCObject* _b = (MyAwesomeCObject*)b
        return MyAwesome_Fallback_Addtion_Operator(_b, a);
    }
    //additional cases or just raise an error lol
};

{% endhighlight %}

The C API provides macros to check the "types" of structures, so this allows for functions to be dispatched accordingly. In my code example, which should not be taken as gospel, I had to also do a few C-style casts to get types to play nice with each other. 

I did this in the assumption that MyAwesomeCObject contains attributs not found in a generic PyObject. For these to be accessed, the compiler has to know that it is trying to pull a reference to those attributes from something guarenteed to have them. Since the type checking macros ensure which variables are of the "type" pointer to MyAwesomeCObject, I currently believe this type of cast could be "safe". Conversely, because the return types are of pointers to PyObject, I have to recast the pointer to MyAwesomeCObject back to a pointer to PyObject. This is so it can be returned to the Python interpreter. 


