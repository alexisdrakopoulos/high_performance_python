
# Why are loops 2x faster inside a function?

Okay so the title might have been a little clickbaity as it clearly depends on what your loops are doing, how many iterations they go through and the answer to the titles question is not even related to loops! If you're willing to forgive the title let's try to figure out what in the heck is going on.

## Profiling

Let's write one Python program where the loop is not contained within a function (and call it `out.py`):
```python
for i in range(100_000_000):
    i
```
and the same program within a function (and call it `in.py`):
```python
def f():
    for i in range(100_000_000):
        i

f()
```
We arbitrarily specified 100 million iterations, since the very clever author knows that this will result in single digit seconds of runtime. Let's run both our programs with the [time](https://en.wikipedia.org/wiki/Time_(Unix)) utility:
```bash
❯ time python3 out.py
python3 out.py  2.54s user 0.01s system 99% cpu 2.557 total
❯ time python3 in.py
python3 in.py  1.08s user 0.01s system 99% cpu 1.097 total
```
Huh? Why is the exact same for loop written outside of a function twice as slow?! (Spoiler: it's related to scoping!)

## Bytecode

If we [dissasemble](https://docs.python.org/3/library/dis.html) our code into bytecode for `out.py` we get:
```
  1           0 LOAD_NAME                0 (range)
              2 LOAD_CONST               0 (100000000)
              4 CALL_FUNCTION            1
              6 GET_ITER
        >>    8 FOR_ITER                 4 (to 18)
             10 STORE_NAME               1 (i)
             12 LOAD_NAME                1 (i)
             14 POP_TOP
             16 JUMP_ABSOLUTE            4 (to 8)
        >>   18 LOAD_CONST               1 (None)
             20 RETURN_VALUE
```
whereas for `in.py` we get:
```
  2           0 LOAD_GLOBAL              0 (range)
              2 LOAD_CONST               1 (100000000)
              4 CALL_FUNCTION            1
              6 GET_ITER
        >>    8 FOR_ITER                 4 (to 18)
             10 STORE_FAST               0 (i)

  3          12 LOAD_FAST                0 (i)
             14 POP_TOP
             16 JUMP_ABSOLUTE            4 (to 8)

  2     >>   18 LOAD_CONST               0 (None)
             20 RETURN_VALUE
```

Let's highlight the differences, with green being `in.py` (loop inside the function `f()`) and red being `out.py`:
```diff
+ 2           0 LOAD_GLOBAL              0 (range)
- 1           0 LOAD_NAME                0 (range)
              2 LOAD_CONST               1 (100000000)
              4 CALL_FUNCTION            1
              6 GET_ITER
        >>    8 FOR_ITER                 4 (to 18)
+            10 STORE_FAST               0 (i)
-            10 STORE_NAME               1 (i)

+ 3          12 LOAD_FAST                0 (i)
-            12 LOAD_NAME                1 (i)
             14 POP_TOP
             16 JUMP_ABSOLUTE            4 (to 8)

  2     >>   18 LOAD_CONST               0 (None)
             20 RETURN_VALUE
```
Only three instructions differ! `LOAD_GLOBAL` instead of `LOAD_NAME`, `STORE_FAST` instead of `STORE_NAME`, and `LOAD_FAST` instead of `LOAD_NAME`. To those with a keen eye there is one more difference, the argument indices to LOAD_CONST (and a few other instructions) being 1 instead of 0. This has no impact on runtime. We can also ignore any differences outside of the for loop and only focus on instructions between bytes 8 and 16:
```diff
        >>    8 FOR_ITER                 4 (to 18)
+            10 STORE_FAST               0 (i)
-            10 STORE_NAME               1 (i)

+ 3          12 LOAD_FAST                0 (i)
-            12 LOAD_NAME                1 (i)
             14 POP_TOP
             16 JUMP_ABSOLUTE            4 (to 8)
```
where we perform a `STORE` and a `LOAD` operation, followed by popping the top item, the newly stored `i`, off the stack (as we don't use it).

## `LOAD_FAST` vs `LOAD_NAME`


Navigating to the Python docs for [`LOAD_FAST`](https://docs.python.org/3.12/library/dis.html#opcode-LOAD_FAST) we are greeted with the following short explanation: 
`Pushes a reference to the local co_varnames[var_num] onto the stack.` Whereas with [`LOAD_NAME`](https://docs.python.org/3.12/library/dis.html#opcode-LOAD_NAME) we are told:
`Pushes the value associated with co_names[namei] onto the stack.`

Diving into the CPython source code we can find these in ceval.py: [`LOAD_NAME`](https://github.com/python/cpython/blob/8a285df806816805484fed36dce5fd5b77a215a6/Python/ceval.c#L2961) and [`LOAD_FAST`](https://github.com/python/cpython/blob/8a285df806816805484fed36dce5fd5b77a215a6/Python/ceval.c#L1811). Even with zero knowledge of [C](https://en.wikipedia.org/wiki/C_(programming_language)) one can quickly guess that `LOAD_FAST`, which has 5 lines of code, is faster than `LOAD_NAME` which has a hefty 60.

`LOAD_FAST` seems to call `GETLOCAL`, does a quick check that we actually got a value and then returns it:
```c
PyObject *value = GETLOCAL(oparg);
```
We can also look at `LOAD_NAME` with all the error checking/raises removed:
```c
TARGET(LOAD_NAME) {
    PyObject *name = GETITEM(names, oparg);
    PyObject *locals = LOCALS();
    PyObject *v;
    if (PyDict_CheckExact(locals)) {
        v = PyDict_GetItemWithError(locals, name);
        if (v != NULL) {
            Py_INCREF(v);
        }
    }
    else {
        v = PyObject_GetItem(locals, name);
    }
    if (v == NULL) {
        v = PyDict_GetItemWithError(GLOBALS(), name);
        if (v != NULL) {
            Py_INCREF(v);
        }
        else {
            if (PyDict_CheckExact(BUILTINS())) {
                v = PyDict_GetItemWithError(BUILTINS(), name);
                Py_INCREF(v);
            }
            else {
                v = PyObject_GetItem(BUILTINS(), name);
                }
            }
        }
    PUSH(v);
    DISPATCH();
}
```
Note that [`PyDict_GetItem`](https://github.com/python/cpython/blob/42fee931d055a3ef8ed31abe44603b9b2856e04d/Objects/dictobject.c#L1649) and [`PyDict_GetItemWithError`](https://github.com/python/cpython/blob/42fee931d055a3ef8ed31abe44603b9b2856e04d/Objects/dictobject.c#L1770) are very similar, with the former silencing errors for historical reasons and the latter raising them.

We can see that we call:
1. `name = GETITEM(names, oparg)` as well as `locals = LOCALS()`.
2. if `PyDict_CheckExact(locals)` we call `PyDict_GetItemWithError(locals, name)` which tries to find `name` in `locals`.
3. Otherwise we call `PyObject_GetItem(locals, name)`.
4. If at this point we still don't have a value, we call `PyDict_GetItemWithError` in the `GLOBALS` namespace.
5. If we still don't have a value, we call `PyDict_CheckExact` in `BUILTINS`.
6. We finally return the value

Rather than diving into these further, we can see that `LOAD_NAME` interfaces with various dictionaries to attempt to find your value, whereas `LOAD_FAST` has a single call to your `GET_LOCAL` with its argument value.

Compared to the global namespace which stores various pieces of data in dictionaries, functions store local variables in a fixed-sized array and are self-contained (unless the `global` keyword is used). This makes accesses much simpler since less things that can go wrong; no hashing or fancy implementations, only a simple pointer is required! The global namespace has to stay dynamic and handle the addition and removal of local variables at runtime, and therefore can't benefit from a fixed array.


### Credits

This post is largely based on a talk from [Anjana Vakil](https://vakila.github.io/) in EuroPython 2016 titled [Exploring Python Bytecode](https://www.youtube.com/watch?v=GNPKBICTF2w) as well as a StackOverflow [post](https://stackoverflow.com/questions/11241523/why-does-python-code-run-faster-in-a-function).