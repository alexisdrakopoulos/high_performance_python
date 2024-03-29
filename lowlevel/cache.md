# CPython stdlib ep1: How does Functools Cache work?

As part of our low-level CPython series exploring the standard library we will explore functools caching functionality! Specifically [LRU cache](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)).

***Note: Code from the CPython library shown in this (and other) posts is usually stripped down to its essentials for ease of communication but always referenced! In this case we removed some args and thus some annoying branches to focus on the core implementation.***

## Python Cache

The [functools](https://docs.python.org/3/library/functools.html) standard library comes with some interesting caching functionality in the form of decorators that may be applied to functions & methods. Here we see an example of caching a function `add` which returns the sum of two integers `x` and `y`.
```python
def add(x: int, y: int) -> int:
    return x + y

from functools import lru_cache

@lru_cache
def cached_add(x: int, y: int) -> int:
    return x + y
```
In subsequent calls with the same arguments the function will not evaluate but rather retrieve a cached value:
```python
# we have never called it before!
add_cached(x=1, y=2)
cached_add.cache_info()
# >>> CacheInfo(hits=0, misses=1, maxsize=128, currsize=1)
add_cached(y=2, x=1)
cached_add.cache_info()
# >>> CacheInfo(hits=0, misses=2, maxsize=128, currsize=2)

# now let's call one from before
add_cached(x=1, y=2)
cached_add.cache_info()
# >>> CacheInfo(hits=1, misses=2, maxsize=128, currsize=2) # note the size stays the same!
```

## LRU Cache Implementation

lru_cache is implemented in Python [here](https://github.com/python/cpython/blob/6cbb57f62d345d7a5d6aeb1b3b5d37a845344d5e/Lib/functools.py#L479) and, with the noisy code removed, is quite simple:
```python
def lru_cache(user_function):

    def decorating_function(user_function):
        wrapper = _lru_cache_wrapper(user_function)
        return update_wrapper(wrapper, user_function)

    return decorating_function
```
We first call `_lru_cache_wrapper` with the user function supplied followed by a call to `update_wrapper` passing our wrapper as well as the original user function!

The [`_lru_cache_wrapper`](https://github.com/python/cpython/blob/6cbb57f62d345d7a5d6aeb1b3b5d37a845344d5e/Lib/functools.py#L525) implementation might seem complex at first glance, however its mostly just boilerplate with a couple of pieces of logic:
```python
def _lru_cache_wrapper(user_function):
    # Constants shared by all lru cache instances:
    sentinel = object()  # unique object used to signal cache misses
    make_key = _make_key  # build a key from the function arguments

    cache = {}
    cache_get = cache.get  # bound method to lookup a key or return None

    def wrapper(*args, **kwds):
        # Simple caching without ordering or size limit
        key = make_key(args, kwds)
        result = cache_get(key, sentinel)
        if result is not sentinel:
            return result
        result = user_function(*args, **kwds)
        cache[key] = result
        return result

    return wrapper
```
The core datastructure here is a dictionary called `cache` which is written to with `cache_get`, which once we look at it is just the standard [`.get()`](https://docs.python.org/3/library/stdtypes.html#dict.get) method defined for dictionaries in Python. However let's take a step back and break `_lru_cache_wrapper` down further.

If we cleanup the code and comment it we get:
```python
def _lru_cache_wrapper_cleaned(user_function):

    sentinel = object()
    cache = {}

    def wrapper(*args, **kwargs):
        # get a hash key
        key = _make_key(args, kwargs)

        # check if we already have this key!
        potential_result = cache.get(key, sentinel)

        # if we didn't hit the sentinel we return our cache hit!
        if potential_result is not sentinel:
            return potential_result

        # if we are here we sadly didn't hit our cache and need to call the function
        result = user_function(*args, **kwargs)

        # we can now cache it using the key we made above
        cache[key] = result

        return result
```
the only trick here is using the [sentinel](https://en.wikipedia.org/wiki/Sentinel_value) value `object()`; recall that Python dictionaries `.get()` method takes two arguments, the first being the key to find in the dictionary, with the second being the default value to return if that key is not found! Here we return a sentinel which we use to determine whether we have cached this key before, and if not we add it to our cache.

***NOTE: `_lru_cache_wrapper` is a lot more complex if a max size is set and involves some threading!***

### [Hashing](https://en.wikipedia.org/wiki/Hash_function)

We still haven't discussed one key piece! The `_make_key` function that takes in the args and kwargs of our user function and returns a hashed key. If you are a little bored the tl;dr is that it uses the Python [`hash` function](https://docs.python.org/3/library/functions.html#hash), which just calls `__hash__` and therefore has a pretty boring [implementation](https://github.com/python/cpython/blob/27b989403356ccdd47545a93aeab8434e9c69f21/Objects/object.c#L765):
```c
Py_hash_t
PyObject_Hash(PyObject *v)
{
    PyTypeObject *tp = Py_TYPE(v);
    if (tp->tp_hash != NULL)
        return (*tp->tp_hash)(v);

    /* object can't be hashed */
    return PyObject_HashNotImplemented(v);
}
```

This still leaves one question, what exactly are we hashing? Well the arguments of course! If we look at the implementation of `_make_key`:
```python
def _make_key(
    args,
    kwds,
    kwd_mark=(object(),),
    fasttypes={int, str},
    type=type,
    len=len,
):
    key = args
    if kwds:
        key += kwd_mark
        for item in kwds.items():
            key += item
    elif len(key) == 1 and type(key[0]) in fasttypes:
        return key[0]
    return _HashedSeq(key)
```
we see our `args` and `kwargs` (named `kwds` here) as well as some other default arguments that we don't touch. Note that `args` are in the form of a tuple of argument values eg `f(5, 10) -> (5, 10)` and `kwargs` are dictionaries `f(x=5, y=10) -> {"x": 5, "y": 10}`.

First we handle the special case in which no `kwargs` passed and only a single integer or string `args` is passed. If that is true we simply return its value; `_make_key((5,), {}) -> 5` or `_make_key(("Hello!,"), {}) -> 'Hello'`.

In all other cases we build a key by doing `key = args + object() + kwarg keys + values repeatedly`. Therefore something like `_make_key(args=(5, 10), kwds={"x": 5, "y": 10})` would have `key = (5, 10, <object object at 0x1062b52f0>, 'x', 5, 'y', 10)`. We then return `_HashedSeq(key)` which internally calls the builtin `hash`.

Ignoring the special case the function can essentially be reduced to:
```python
def simple_key(args, kwargs):
    return _HashedSeq(args + (object(),) + tuple([item for item in kwargs.items()]))
```

***Interesting factoid alert! The following will result in two caches!***
```python
cached_add(x=1, y=2)
cached_add(y=2, x=1)
cached_add.cache_info()
# >>> CacheInfo(hits=0, misses=2, maxsize=128, currsize=2)
```
***This is due to the fact that sorting of keyword args was switched off to improve speed.***

At this point you might wonder why does `_HashedSeq(key)` exist? To reduce the amount of hashing we have to do! If you recall the innards of the `_lru_cache_wrapper` from before, we have a small inner check that looks something like:
```python
    # get a hash key
    key = _make_key(args, kwargs)

    # check if we already have this key!
    potential_result = cache.get(key, sentinel)

    # if we didn't hit the sentinel we return our cache hit!
    if potential_result is not sentinel:
        return potential_result

    # if we are here we sadly didn't hit our cache and need to call the function
    result = user_function(*args, **kwargs)

    # we can now cache it using the key we made above
    cache[key] = result
```
So where do we perform hashing? Well at `cache.get` of course, as well as in `cache[key]`. Twice for one key! What `_HashedSeq` does is calculate the hash once upon initialization and then store it in the `__hash__` method. This is then applied to all the tuples returned from `_make_key`:
```python
class _HashedSeq(list):
    __slots__ = "hashvalue"

    def __init__(self, tup, hash=hash):
        self[:] = tup
        self.hashvalue = hash(tup)

    def __hash__(self):
        return self.hashvalue
```

### `update_wrapper`

We left `update_wrapper` until the very end due to it not being particularly interesting. It exists solely to return a wrapper that looks like the original function! That's the `__wrapper__` we saw at the very start. 

The code is fairly short too:
```python
WRAPPER_ASSIGNMENTS = (
    "__module__",
    "__name__",
    "__qualname__",
    "__doc__",
    "__annotations__",
)
WRAPPER_UPDATES = ("__dict__",)

def update_wrapper(
    wrapper, wrapped, assigned=WRAPPER_ASSIGNMENTS, updated=WRAPPER_UPDATES
):
    for attr in assigned:
        try:
            value = getattr(wrapped, attr)
        except AttributeError:
            pass
        else:
            setattr(wrapper, attr, value)
    for attr in updated:
        getattr(wrapper, attr).update(getattr(wrapped, attr, {}))
    # Issue #17482: set __wrapped__ last so we don't inadvertently copy it
    # from the wrapped function when updating __dict__
    wrapper.__wrapped__ = wrapped
    # Return the wrapper so this can be used as a decorator via partial()
    return wrapper
```
Here the wrapper is the function we want to make look like our wrapped (user function) func. We do this by transplanting relevent attributes outlined above using `setattr`, and by finally updating the `__dict__`.

Just to make it clear on what journey our `user_function` takes let's look at it with only calls and returns. We first pass it to `lru_cache` which passes it to `decorating_function`. `decorating_function` then passes it to `_lru_cache_wrapper` and gets back a `wrapper`:
```python
def lru_cache(user_function):
    def decorating_function(user_function):
        wrapper = _lru_cache_wrapper(user_function)
        return update_wrapper(wrapper, user_function)
    return decorating_function
```
where `wrapper`, without the caching, is just a function that calls the wrapper and returns its result (hence being identical to the function in operation):
```python
def _lru_cache_wrapper(user_function):
    def wrapper(*args, **kwargs):
        # do caching stuff here
        result = user_function(*args, **kwargs)
        return result
    return wrapper
```

## Conclusion

Python caching is actually quite simple, and even implemented purely in Python by leveraging the strong performance of the Python dictionary and the builtin hashing functions.

I hope this was interesting enough that you made it to the end, and I hope to write many more such posts!