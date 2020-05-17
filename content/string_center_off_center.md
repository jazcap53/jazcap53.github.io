title: Python's Eccentric `str.center()`
date: May 17, 2020
author: Andy Jarcho

## A bit of odd behavior

There's a small piece of Python that's made me curious: the behavior of str.center().

Python 3.8's [official doc for that method](https://docs.python.org/3/library/stdtypes.html#str.center) reads, in full:  
>str.center(width[, fillchar])  
Return centered in a string of length width. Padding is done using the specified fillchar (default is an ASCII space). The original string is returned if width is less than or equal to len(s).

OK. But what happens when len(s) is odd and width is even, or vice versa? Then, it's not possible to exactly center the string in the output field.

When you call, e.g.,  
```
'hello'.center(10, '^')
```
,  
Python has to choose between '^^^hello^^' and '^^hello^^^'. In this case, Python chooses '^^hello^^^'. 
The shorter segment of padding appears before the input string; the longer padding comes after.

But, when you call  
```
'heck'.center(9, '^')
```
,  
the output is '^^^heck^^', with the *longer* piece of padding going first.

PyPy, cython, and jython all exhibit the same behavior.

That's always struck me as odd, so I looked into Python's source code for the implementation.

After much hunting (thank you [ag, the Silver Searcher](https://github.com/ggreer/the_silver_searcher)), 
I found the relevant code in 'cpython/Objects/unicodeobject.c'.

It looks like so:
```C
static PyObject *
unicode_center_impl(PyObject *self, Py_ssize_t width, Py_UCS4 fillchar)
/*[clinic end generated code: output=420c8859effc7c0c input=b42b247eb26e6519]*/
{
    Py_ssize_t marg, left; 

    if (PyUnicode_READY(self) == -1) 
        return NULL; 

    if (PyUnicode_GET_LENGTH(self) >= width)
        return unicode_result_unchanged(self);

    marg = width - PyUnicode_GET_LENGTH(self);
    left = marg / 2 + (marg & width & 1);

    return pad(self, left, marg - left, fillchar);
}
```
These three lines show what's going on:
```C
    marg = width - PyUnicode_GET_LENGTH(self);
    left = marg / 2 + (marg & width & 1);

    return pad(self, left, marg - left, fillchar);
```
Let's refer to our input string as 'S'. `PyUnicode_GET_LENGTH(self)` 
is simply the length of S. So, if `width` (our field width) and len(S) have 
the same parity: 
* `marg` will be even, 
* `(marg & width & 1)` will be 0, 
* `left` will equal `marg - left`, 
* there will be the same amount of padding before and after S,  
and everybody goes home happy.

But if `width` and len(S) have **different** parity: 
* `marg` will be odd, 
* `left` will differ from `marg - left` by exactly 1,
  * if `width` is odd, Python adds 1 to `left`, and our extra fill character appears to the left of S,
  * if `width` is even, Python adds 0 to `left`, and our extra fill character appears to the *right* of S.

Why is this behavior undocumented? The answer lies in 
[Section 7.2.5](https://devguide.python.org/documenting/#economy-of-expression) 
of the Python Developer's Guide: "Economy of Expression". Its first sentence reads:
>More documentation is not necessarily better documentation. 
>Err on the side of being succinct.

'Nuf said.