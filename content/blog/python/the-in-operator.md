---
title: "The in Operator"
date: 2023-07-01T19:10:35+02:00
draft: false
---

# Introduction

If you use the `in` operator in Python, testing whether an element is 
present in a collection is almost as easy as asking a question.

```shell
>>> 'one'  in ['one', 'two', 'three']
True
>>> 'four'  in ('one', 'two', 'three')
False
>>> 'five' in {'one', 'two', 'three'}
False
>>> 'three' in {'one': 1, 'two': 2, 'three': 3}
True
```

From the examples above, we can see that the `in` operator works with various
collection types: lists, tuples, sets, and dictionaries. What about performance?
If you frequently had to test membership of items against a collection of
million elements, which data structure would you choose and why?

# Memory Footprint and Performance

## Lists and Tuples

To create and store elements in a list, Python has to allocate some memory for
the list itself, and memory for the pointers to the elements it stores. To
prevent memory allocation on each `list.append`, Python allocates some extra
memory to keep appends fast (amortized `O(1)`). So while there may be a single
element in a list, there is actually room for 4. Once those 4 places are
occupied, Python will perform a reallocation making room for another 4. You can
test this out for yourself by calling `list.__sizeof__()` after each append.

```python
l = []
print(l.__sizeof__())  # allocation for the list itself

for i in range(5):
    l.append(i)  # only when the fifth item is appended will the list change size
    print(len(l), l.__sizeof__())
```

Tuples on the other hand, have a fixed size. That means, there is no
reallocation and no other layers of indirection. The elements of tuples are
stored directly in a struct making them very lean, memory wise.

Finding an element in a tuple or a list potentially requires traversing the
whole collection and comparing each element in the collection against the
object which membership you are testing. The following code snippet illustrates
that:

```python
def contains(collection, target):
    for item in collection:
        if target == item:
            return True
    return False
```

If the `target` object is not present in the collection, the whole collection
will be traversed before `False` is returned. The complexity of such an
operation is `O(n)`. For a list twice as long, membership testing will take
twice as long. If membership testing happens often on large lists or tuples,
Python will spend quite some time traversing those lists and tuples, over and
over again.

## Sets and Dictionaries

What about sets and dictionaries? To understand those data structures, we
first need to understand the basics of hash maps.

### Custom Hash Map

Not being satisfied with having to traverse the whole collection to find the
element of interest, we wonder is there a way we could simply find the index
of the element we are looking for. So, if we have a list `l = ['one', 'two',
'three']`, and we want to know whether `'two'` is in our list, we need a
function that will map the element `'two'` to an index of 1 because `l[1]
== 'two'`. The hash function does precisely that. Let's try to implement our
own hash map so that we can qualitatively understand the memory footprint and
performance.

The hash map stores key-value pairs in buckets. The underlying data
structure those key-value pairs are stored in is an array. For our Python
equivalent, we will use a list. Those buckets are again stored in arrays, so
we can use a list again to store them.

```python
def naive_hash(value: str):
    # Maps every string starting with 'a' or 'A' to 0, 'b' or 'B' to 1, etc.
    if value[0] not in 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ':
        raise ValueError('naive_hash supports only ascii letters')
    return ord(value[0].upper()) - 65


class KeyValue:
    def __init__(self, key, value):
        self.key = key
        self.value = value

    def __repr__(self):
        cls = self.__class__.__name__
        return f'<{cls}(key={self.key}, value={self.value})>'


class Bucket:
    def __init__(self):
        # Bucket holds a list of key-value pairs.
        self._keys_values: list[KeyValue] = []

    def put(self, target: KeyValue):
        # Check whether this key is already present. If it is replace its value.
        for key_value in self._keys_values:
            if key_value.key == target.key:
                key_value.value = target.value
                return
        # This key is not present, just append the key-value pair.
        self._keys_values.append(target)

    def get(self, key: str) -> KeyValue:
        # If there are many key-value pairs in a bucket, we have to check 
        # each one for the key we are searching for.
        for key_value in self._keys_values:
            if key_value.key == key:
                return key_value
        raise KeyError(f'no key "{key}"')

    def delete(self, key: str):
        for key_value in self._keys_values:
            if key_value.key == key:
                self._keys_values.remove(key_value)
                return
        raise KeyError(f'no key "{key}"')

    def contains(self, key) -> bool:
        for key_value in self._keys_values:
            if key_value.key == key:
                return True
        return False

    def __bool__(self):
        return bool(self._keys_values)

    def __repr__(self):
        cls = self.__class__.__name__
        return f'<{cls} {repr(self._keys_values)}>'


class HashMap:
    def __init__(self):
        # HashMap holds a list of buckets.
        # We initialize the list with empty buckets to avoid memory 
        # reallocation. There are 26 ascii letters, so we create 26 buckets.
        self._buckets: list[Bucket] = [Bucket() for _ in range(26)]

    def put(self, key_value: KeyValue):
        # We calculate the index, i.e. the corresponding bucket for this key.
        hash_value = naive_hash(key_value.key)
        self._buckets[hash_value].put(key_value)

    def get(self, key: str):
        hash_value = naive_hash(key)
        return self._buckets[hash_value].get(key)

    def delete(self, key: str):
        hash_value = naive_hash(key)
        self._buckets[hash_value].delete(key)

    def contains(self, key) -> bool:
        hash_value = naive_hash(key)
        return self._buckets[hash_value].contains(key)

    def __repr__(self):
        cls = self.__class__.__name__
        buckets = ', '.join(
            f'{idx}: {bucket}' for idx, bucket in enumerate(self._buckets)
            if bucket
        )
        if buckets:
            return f'<{cls} {buckets}>'
        return f'<{cls}>'
```

There is a lot to digest in the code sample above, so let's repeat the most 
important steps and points. The main idea of the hash map is to find the 
corresponding bucket for a given key. We do this by using our `naive_hash` 
function which produces a bucket index for any string containing ascii letters. 
There is one subtle point to be made regarding our naive hash function. It 
produces the same hash value, i.e. the same bucket index for all strings 
starting with the same letter. This is called a hash collision, and when 
that happens, the corresponding bucket stores multiple key-value pairs. 
Let's see an example.

```shell
>>> hm = HashMap()
>>> hm
<Hashmap>  # our hash map is empty
>>> hm.put(KeyValue('one', 1)) 
>>> hm
<HashMap 14: <Bucket [<KeyValue(key=one, value=1)>]>>  # bucket 14 got an item
>>> hm.put(KeyValue('two', 2))
>>> hm
<HashMap 14: <Bucket [<KeyValue(key=one, value=1)>]>, 19: <Bucket [<KeyValue(key=two, value=2)>]>>
>>> hm.put(KeyValue('three', 3))  # hash collision
>>> hm
<HashMap 14: <Bucket [<KeyValue(key=one, value=1)>]>, 19: <Bucket [<KeyValue(key=two, value=2)>, <KeyValue(key=three, value=3)>]>>
>>> naive_hash('one'), naive_hash('two'), naive_hash('three')
(14, 19, 19)
```

From the above example we can see that `'one'` goes to bucket 14, while 
`'two'` and `'three'` produce a hash collision and end up in the same bucket,
bucket 19.

### Hash Map Performance

Because of the underlying hash function, finding the appropriate bucket is a 
single operation, which means retrieving elements from the hash map is very
fast. In fact, it is `O(1)`, i.e. independent of the number of elements in 
the hash map unless collisions are so frequent elements end up in the same 
bucket. That's why production ready hash maps reallocate memory, keep track 
of bucket occupancy, and redistribute elements in the bucket to keep 
retrieving elements from the hash map `O(1)`. To prevent hash collisions 
from happening too frequently, Python keeps approximately one third of the 
underlying array unoccupied. This increases the memory footprint, but it gives 
a huge performance gain.

Although sets and dictionaries have a memory overhead, they are data structures
optimized for fast membership checking, and you should use them with
the `in` operator whenever possible.

# What is a Collection

Recalling our example from the introduction, you might believe that 
affecting performance with regard to the `in` operator amounts to changing 
braces; from square and round to curly. Actually, that's not far from the 
truth. The question is, why does it work? And, maybe more importantly, how 
can you define your own objects that support membership checking via the 
`in` operator?

This works because lists, tuples, dictionaries, and sets are all instances 
of `collections.abc.Collection` class. That is an abstract base class used 
to define a protocol an instance of that class should implement. If you 
check the documentation, you will see that `Collection` inherits from 
`Sized`, `Iterable`, and `Container`. Inheriting from `Sized` means that you 
can ask how many elements are there in the collection by implementing 
`__len__`. Inheriting from `Iterable` means that the collection can be 
iterated over using a for loop, by implementing `__iter__`. And finally, 
inheriting from `Container` means that you can test for membership using the 
`in` operator by implementing the `__contains__` special method. This is 
what all collections have in common, and this is what makes Python easy to 
use. Let's explore this through some code samples.

```python
class ContainerTest:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __contains__(self, item):
        return item in (self.x, self.y)
```

```shell
>>> ct = ContainerTest(1, 2)
>>> 1 in ct
True
>>> 2 in ct
True
>>> 3 in ct
False
```

In the example above, Python will actually call the `__contains__` special 
method like this.

```python
ct.__contains__(1)
```

That's fairly straightforward, but what is perhaps less known is that Python 
will try to support the `in` operator even if `__contains__` is not 
implemented. First, Python will try iterating over the collection if that 
object supports iteration via `__iter__`. Lastly, it will try membership 
testing using the legacy `__getitem__` protocol. Let's take a look at a few 
code examples.

```python
class IterableTest:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __iter__(self):
        yield from (self.x, self.y)
```

This class supports iteration, and that is enough for membership testing 
using the `in` operator. That's the reason why the `in` operator works with 
generators as well.

```shell
>>> it = IterableTest(1, 2)
>>> 1 in it
True
>>> 2 in it
True
>>> 3 in it
False
```

Finally, if neither `__contains__` nor `__iter__` is available, Python will 
try the legacy `__getitem__` protocol for membership testing (and iteration).

```python
class GetItemTest:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __getitem__(self, item):
        if item == 0:
            return self.x
        elif item == 1:
            return self.y
        else:
            raise IndexError
```

The `__getitem__` protocol is used to support the indexing operation, such 
as you would apply on a list.

```shell
>>> l = [1, 2, 3]
>>> l[0]  # l.__getitem__(0)
1
```

Let's verify that `__getitem__` supports membership testing.

```shell
>>> gt = GetItemTest(1, 2)
>>> 1 in gt
True
>>> 2 in gt
True
>>> 3 in gt
False
```

# Conclusion

We have seen that the `in` operator is compatible with many data structures. 
Python will try to support that operator first by checking whether the 
`__contains__` special method is implemented. If it is not, it will try 
iterating through the collection by using `__iter__`, or `__getitem__` if 
the former is not available. If elements of a collection are hashable, the 
`in` operator will be most performant with sets and dictionaries.
