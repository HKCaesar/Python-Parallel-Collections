###Python Parallel Collections
####Implementations of dict and list which support parallel map/reduce style operations

####Who said Python was not setup for multicore computing? 
In this package you'll find very simple parallel implementations of list, string and dict and a list-like generator. The parallelism uses the [Python 2.7 backport](http://pythonhosted.org/futures/#processpoolexecutor-example) of the [concurrent.futures](http://docs.python.org/dev/library/concurrent.futures.html) package. If you can define your problem in terms of map/reduce/filter/flatten operations, it will run on several parallel Python processes on your machine, taking advantage of multiple cores. 
Otherwise these datastructures are equivalent to the non-parallel ones found in the standard library.

Please note that although the below examples are written in interactive style, due to the nature of multiple processes they will not 
actually work in the interactive interpreter. 

####Getting Started
```python
pip install python-parallel-collections
pip install futures
from parallel.parallel_collections import ParallelList, ParallelDict, ParallelString, ParellelGen
```

####Examples

```python
>>> def double(i):
...     return i*2
... 
>>> list_of_list =  ParallelList([[1,2,3],[4,5,6]])
>>> flat_list = list_of_list.flatten()
[1, 2, 3, 4, 5, 6]
>>> list_of_list
[[1, 2, 3], [4, 5, 6]]
>>> flat_list.map(double)
[2, 4, 6, 8, 10, 12]
>>> list_of_list.flatmap(double)
[2, 4, 6, 8, 10, 12]
```

As you see every method call returns a new collection, instead of changing the current one.
The exception is the foreach method, which is equivalent to map but instead of returning a new collection it operates directly on the 
current one and returns `None`.  
```python
>>> flat_list
[1, 2, 3, 4, 5, 6]
>>> flat_list.foreach(double)
None
>>> flat_list
[2, 4, 6, 8, 10, 12]
```

Since every operation (except foreach) returns a collection, these can be chained.
```python
>>> list_of_list =  ParallelList([[1,2,3],[4,5,6]])
>>> list_of_list.flatmap(double).map(str)
['2', '4', '6', '8', '10', '12']
```

####When to use the ParallelGen class?
The ParallelGen class should be used in the same cases that you would normally use a generator: to avoid the evaluation of an intermittent datastructure. With the parallel generator, you can chain map/filter/reduce calls without evaluating the entire datastructure on every operation, just like you would when building data processing pipelines using a chain of generator functions. Each element in the datastructure will be processed one by one. The below example illustrates this. Note each operation on the parallel list results in the entire list being evaluated before the next operation, while the generator allows every element go through each step before sending the next one in. 
Also note the the generator will not result in anything happening unless you actually do something to evaluate, such as the list comprehension does in the below example. (a call to len for example would also result in the evaluation of the generator)

```python
>>> def _print(item):
...     print item 
...     return item
... 
>>> def double(item):
...     return item * 2 
... 
>>> plist = ParallelList(range(5))
>>> [i for i in plist.map(double).map(_print).map(double).map(_print)]
0
2
4
6
8
0
4
8
12
16
>>> pgen = ParallelGen(range(5))
>>> [i for i in pgen.map(double).map(_print).map(double).map(_print)]
0
0
2
4
4
8
6
12
8
16
```

####Regarding lambdas and closures
Sadly lambdas, closures and partial functions cannot be passed around multiple processes, so every function that you pass to the collection methods needs to be defined using the def statement. If you want the operation to carry extra state, use a class with a `__call__` method defined.
```python
>>> class multiply(object):
...     def __init__(self, factor):
...         self.factor = factor
...     def __call__(self, item):
...         return item * self.factor
... 
>>> multiply(2)(3)
6
>>> list_of_list =  ParallelList([[1,2,3],[4,5,6]])
>>> list_of_list.flatmap(multiply(2))
[2, 4, 6, 8, 10, 12]
```

###Quick examples of map, reduce and filter

####Map and FlatMap

Functions passed to the map method of a list will be passed every element in the list and should return a single element. For a dict, the function will receive a tuple (key, values) for every key in the dict, and should equally return a two element sequence. Flatmap will first flatten the sequence then apply map to it.
 
```python
>>> def double(item):
...    return item * 2
...
>>> list_of_list =  ParallelList([[1,2,3],[4,5,6]])
>>> list_of_list.flatmap(double).map(str)
['2', '4', '6', '8', '10', '12']
>>> def double_dict(item):
...     k,v = item
...     try:
...         return [k, [i *2 for i in v]]
...     except TypeError:
...         return [k, v * 2]
... 
>>> d = ParallelDict(zip(range(2), [[[1,2],[3,4]],[3,4]]))
>>> d
{0: [[1, 2], [3, 4]], 1: [3, 4]}
>>> flat_mapped = d.flatmap(double_dict)
>>> flat_mapped
{0: [2, 4, 6, 8], 1: [6, 8]}

>>> def to_upper(item):
...     return item.upper() 
... 
>>> p = ParallelString('qwerty')
>>> mapped = p.map(to_upper)
'QWERTY'
```

####Reduce
Reduce accepts an optional initializer, which will be passed as the first argument to every call to the function passed as reducer
```python
>>> def group_letters(all, letter):
...     all[letter].append(letter)
...     return all
... 
>>> p = ParallelList(['a', 'a', 'b'])
>>> reduced = p.reduce(group_letters, defaultdict(list))
{'a': ['a', 'a'], 'b': ['b']}
>>> p = ParallelString('aab')
>>> p.reduce(group_letters, defaultdict(list))
{'a': ['a', 'a'], 'b': ['b']}
```

####Filter
The Filter method should be passed a predicate, which means a function that will return True or False and will be called once for every element in the list and for every (key, values) in a dict.
```python
>>> def is_digit(item):
...     return item.isdigit()
...
>>> p = ParallelList(['a','2','3'])
>>> pred = is_digit
>>> filtered = p.filter(pred)
>>> filtered
['2', '3']

>>> def is_digit_dict(item):
...    return item[1].isdigit()
...
>>> p = ParallelDict(zip(range(3), ['a','2', '3',]))
{0: 'a', 1: '2', 2: '3'}
>>> pred = is_digit_dict
>>> filtered = p.filter(pred)
>>> filtered
{1: '2', 2: '3'}
>>> p = ParallelString('a23')
>>> p.filter(is_digit)
'23'
```