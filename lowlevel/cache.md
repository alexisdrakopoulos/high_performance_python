# CPython stdlib ep1: How does Python Cache work?

As part of our low-level CPython series exploring the standard library we

## Python Cache

The [functools](https://docs.python.org/3/library/functools.html) standard library comes with some interesting [caching](https://en.wikipedia.org/wiki/Cache_replacement_policies#Least_recently_used_(LRU)) functionality in the form of decorators that may be applied to functions & methods. Here we see an example of caching a function `add` which returns the sum of two integers `x` and `y`.
```python
def add(x: int, y: int) -> int:
    return x + y

from functools import cache

@cache
def cached_add(x: int, y: int) -> int:
    return x + y
```
When `add(x, y)` is called the numbers are loaded and placed onto the stack followed by a binary add that pops them off and returns their sum to the top of the stack:
```
  2           0 LOAD_FAST                0 (x)
              2 LOAD_FAST                1 (y)
              4 BINARY_ADD
              6 RETURN_VALUE
```
However when `cached_add(x, y)` is called something more interesting happens. Simply dissasembling it doesn't tell us much:
```
Disassembly of __wrapped__:
  3           0 LOAD_FAST                0 (x)
              2 LOAD_FAST                1 (y)
              4 BINARY_ADD
              6 RETURN_VALUE

Disassembly of cache_parameters:
520           0 LOAD_DEREF               0 (maxsize)
              2 LOAD_DEREF               1 (typed)
              4 LOAD_CONST               1 (('maxsize', 'typed'))
              6 BUILD_CONST_KEY_MAP      2
              8 RETURN_VALUE
```
We can see our original function in `__wrapped__` but we also now have some strange opcodes in `cache_parameters` as well as even stranger arguments? what is `typed`!? Let's find out!

## LRU Cache Implementation

lru_cache is implemented in Python [here](https://github.com/python/cpython/blob/6cbb57f62d345d7a5d6aeb1b3b5d37a845344d5e/Lib/functools.py#L479) and, with the noisy code removed, is quite simple:
```python
_CacheInfo = namedtuple("CacheInfo", ["hits", "misses"])

def lru_cache():

    def decorating_function(user_function):
        wrapper = _lru_cache_wrapper(user_function, _CacheInfo)
        return update_wrapper(wrapper, user_function)

    return decorating_function
```
We first call `_lru_cache_wrapper` with the user function supplied as well as a constant named tuple, we then make a call to `update_wrapper` passing our wrapper as well as the original user function!

**Interesting factoid alert! The following will result in two caches!**
```python
cached_add(x=1, y=2)
cached_add(y=2, x=1)
cached_add.cache_info()
>>> CacheInfo(hits=0, misses=2, maxsize=128, currsize=2)
```
This is due to the fact that sorting of keyword args was switched off to improve speed.

