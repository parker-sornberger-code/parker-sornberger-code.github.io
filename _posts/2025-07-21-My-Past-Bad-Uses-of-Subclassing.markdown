---
layout: post
title:  Questionable uses of inheritance 
date:   2025-07-21 14:13:00 -0400
categories: Python
---

Inheritance in object oriented programming prevents redundancy and make code more reusable. For new programmers, it's an attractive skill to try to develop. There are countless tutorials online on how a person new to this in Python can learn the skill. These tutorials usually involve creating something like a generic class for an animal, and then derived classes such as a cat or dog. They're cute ways to introduce how properties are transferred from parent to child. 

However, tutorials only go so far. Like all programming, people learn by doing. Unfortunately, the best tools for teaching with the approach of writing code and seeing what happens are making mistakes. More often than not, poorly written code will still run, without error. 

For instance, there is nothing wrong with this code in the sense that the Python interpreter will gladly run it. 

{% highlight Python %}
class Parent:
    def __init__(self, important_argument_1, important_argument_2):
        self.important_attr_2 = important_argument_1
        self.important_attr_2 = important_argument_2
    def parent_foo(self):
        print(self.important_attr_1, self.important_attr_2)

class Child(Parent):
    def __init__(self, unrelated_arg_1, unrelated_arg_2):
        self.reduntant_attr_1 = unrelated_arg_1
        self.reduntant_attr_2 = unrelated_arg_2
child = Child(1, 2)
{% endhighlight %}

Perhaps there are code linters which would identify that the Child class does not initialize the Parent class. Because Child inherits from Parent, Child can still access the function, `parent_foo`, which relies on Parent being initialized since it accesses its attributes. Furthermore, because Parent has not been initialized, Child does not have any of the attributes of the Parent class! 
{% highlight Python %}
{% endhighlight %}
Trying to call the `parent_foo` function from a Child object will give the following error:
{% highlight Python %}
child.parent_foo()
AttributeError: 'Child' object has no attribute 'important_attr_2'
{% endhighlight %}
A lazy way to fix this code would be to just give Child a variadic ("splat") argument where someone can just put in the arguments needed to initialize the Parent class. (Don't do this though, since someone has to know the necessary arguments for Parent). 
{% highlight Python %}
class Parent:
    def __init__(self, important_argument_1, important_argument_2):
        self.important_attr_1 = important_argument_1
        self.important_attr_2 = important_argument_2
    def parent_foo(self):
        print(self.important_attr_1, self.important_attr_2)

class Child(Parent):
    def __init__(self, unrelated_arg_1, unrelated_arg_2, *parent_args):
        super(Child, self).__init__(*parent_args)
        self.reduntant_attr_1 = unrelated_arg_1
        self.reduntant_attr_2 = unrelated_arg_2
        
child = Child(1, 2, "hello", "sunshine")
child.parent_foo() # --> "hello sunshine"
{% endhighlight %}

Now this code generates no errors, since the Parent class has been initialized by the Child class. 

In the bad example above, the code also shows another flaw of some object oriented code, that has been a theme for my previous bad code. Child does not extend Parent in a meaningful way. Of course, this is a dummy example, created to be bad. The point stands though, especially before the call to initalize parent via the super function was added in the revised code. Refactoring the code to remove Child as a subclass to Parent changes nothing. All the `parent_foo` function did was just print two attributes, which could just be added to Child to give that same functionality. 

Here is a more advanced example. Consider making an array class similar to the functionality of a NumPy array. A natural choice would be to subclass a container already in Python to provide some ease in the more mundane implementation details. Tuple would not fit since an array would require `__setitem__`. This could only be invoked via the idiom `super().__setitem__`, which cannot support item assignment because tuples are designed to be immutable. Thus, the best choice would be a list. 

The below code is a rough recreation of a NumPy-style array implemented as a subclass of a list.

{% highlight Python %}
class Array(list):
    def __init__(self, container):
        super(Array, self).__init__(container)
        self.dimensions = 1
        self.shape = (len(container), )
        self._check_shape()
    def _check_shape(self):
        shape = []
        item = self
        while hasattr(item, "__len__"):
            shape.append(len(item))
            item = item[0]
        self.shape = tuple(shape)
        self.dimensions = len(shape)
    def __getitem__(self, index):
        ret = self
        if isinstance(index, int):
            ret = super().__getitem__(index)
            if isinstance(ret, list):
                return Array(ret)
            return ret
        if isinstance(index, tuple):
            
            for i in index:
                ret = ret[i]
            if isinstance(ret, list):
                return Array(ret)
            return ret
        if isinstance(index, slice):
            ret = ...
            """
            You get the idea byt this point hopefully
            """
    def __iter__(self):
        """
        This is not my original __iter__ for this class
        I had a bunch of gnarly generator logic using chain from itertools
        Inlcuding that code in this fruitfly example would be a bit too much...
        But the point is, my __iter__ was very convoluted
        In the end, if the array was just a single, flat list, it would default to the list __iter__
        """
        return super().__iter__()
{% endhighlight %}
While again, this approach does work, I do not believe it had to be a subclass of list to work. I override nearly every method that makes a list a list, and while I do not show it here, in my original code, I do not even block access to append, extend, or even pop. NumPy arrays have a single size that cannot be increased. If I raised an error if a user tried to access append, extend, or pop, then I would have even less reason to subclass list. Furthermore, in the call to initialize the parent class, `container` is essentially copied again since `super(Array, self).__init__(container)` is equivalent in effect to `newlist = list(container)`. Thus initalizing the class -literally just creating an object- takes linear time!

Moreover, in this example, while I do not show it in my code snippet, believe me when I say that `__setitem__` was a nightmare. A key component of a NumPy array is its ability to mutate a parent object via actions on a view into that parent object. In place operations are very imporant to how people use NumPy arrays in real code. Because my class basically returned new Array objects everywhere, it was incredibly difficult to reflect a change into the original array. 

Finally, while it is not shown exactly in this code snippet because of how gnarly the original logic was, I viewed an N-dimensional array as a nested array of arrays. This is just not how it works for NumPy. To make something like a nested array dynamically in C, someone would have to create a myriad of pointer to pointer types for vairous cases. Something like

{% highlight C %}
typedef struct {
    float** arr;
}two_d_array;
typedef struct {
    float*** arr;
}three_d_array;
//...
typedef struct {
    float****** arr;
}high_d_array_lol;
{% endhighlight %}

To contrast this approach, here is the definition of the core data structure in the C API for NumPy, the PyArray.

{% highlight C %}
typedef struct PyArrayObject {
    PyObject_HEAD
    char *data; //THIS IS THE IMPORTANT PART
    int nd;
    npy_intp *dimensions;
    npy_intp *strides;
    PyObject *base;
    PyArray_Descr *descr;
    int flags;
    PyObject *weakreflist;
    /* version dependent private members */
} PyArrayObject;
{% endhighlight %}

The array data, the attribute which holds the contiguous numerica data is `char *data`. There is no mutation of an existing sequence or container type, and of course, why would there be? The point of NumPy would is to provide an alternative container that holds leaner objects than anything found in Python for high efficiency numerical computing. 