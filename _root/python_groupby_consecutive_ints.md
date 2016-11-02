# Python groupby and Detecting Consecutive Integers

I recently had a need to detect consecutive integers in a longer list so that I would be collapse the list into something consumable as a Slurm job array requests.  Basically:

    1 3 5 6 7 8 12

to

    1 3 5-8 12

A quick Google search pointed over to
[this](http://stackoverflow.com/a/2361991/1653263) answer on stackoverflow
pointing to the [Python
docs](https://docs.python.org/2.6/library/itertools.html#examples).

```
>>> from itertools import groupby
>>> from operator import itemgetter
>>> data = [ 1, 4,5,6, 10, 15,16,17,18, 22, 25,26,27,28]
>>> for k, g in groupby(enumerate(data), lambda (i, x): i-x):
...     print map(itemgetter(1), g)
```

or, for [python3](#note_for_python3):

```
>>> from itertools import groupby
>>> from operator import itemgetter
>>> data = [ 1, 4,5,6, 10, 15,16,17,18, 22, 25,26,27,28]
>>> for k, g in groupby(enumerate(data), lambda x: x[1] - x[0]):
...     print(list(map(itemgetter(1), g)))
```

One of the comments was:

>  I was about to upvote this answer because of how clever it is.
>  Unfortunately, it's too clever for me to upvote without it having an
>  explanation of what the code is doing/how it works. – mgilson Nov 24 '15 at
>  16:52 

Rather than just dumping this into my tool, I thought I'd try to work out exactly why this whole bundle of gunk works.

It begins with the list- all integers.  For `groupby` to work properly these
need to be sorted.

First there's the [`groupby`](https://docs.python.org/3/library/itertools.html#itertools.groupby) function:

> Make an iterator that returns consecutive keys and groups from the iterable.
> The key is a function computing a key value for each element. If not
> specified or is None, key defaults to an identity function and returns the
> element unchanged. Generally, the iterable needs to already be sorted on the
> same key function.

To make the list "group-able", the `enumerate` function is applied to the list,
creating a list of tuples:

```
In [47]: list(enumerate(data))
Out[47]: 
[(0, 1),
 (1, 4),
 (2, 5),
 (3, 6),
 (4, 10),
 (5, 15),
 ...
```

The reason for creating an enumerate will become clear shortly.

We have an iterable to group- now we need to have a function to group the
elements of this list.  In this case, we've got a lambda function that will,
given a list or tuple, return the difference between the first and second
elements.  Elements of the list which return the same value for this difference
will be grouped together and returned as a list-within-a-list.

I struggled with this for some time.  I could not figure out for the life of me
_how_ that difference would indicate that numbers in the list were consecutive.
So, I walked it through by hand:

```
>>> e = enumerate(data)
>>> for t in list(e):
       print("{}: {}".format(t[1]-t[0],t))
       ....:     
1: (0, 1)
3: (1, 4)
3: (2, 5)
3: (3, 6)
6: (4, 10)
10: (5, 15)
10: (6, 16)
10: (7, 17)
10: (8, 18)
13: (9, 22)
15: (10, 25)
15: (11, 26)
15: (12, 27)
15: (13, 28)
```

So here is where the author of that example was quite clever.  If a sequence of
numbers is consecutive over some range in the list, the difference between any
number in that sub-sequence and its position in the list will remain constant
as the index and value will be increasing by the same amount (+1):

```
10: (5, 15)  # 15 - 5 = 10
10: (6, 16)  # 16 - 6 = 10
10: (7, 17)  # 17 - 7 = 10
10: (8, 18)  # 18 - 8 = 10
```

Once that penny dropped, it became quite obvious (hashtag head-desk).  All that
remains at that point is to extract the grouped lists.  The
`list(map(itemgetter))` extracts all of the grouped values from the group
object and puts them into a list:

```
>>> for k, g in groupby(enumerate(data), lambda x : x[1]-x[0]):
        print(list(map(itemgetter(1), g)))
        ....:     
[1]
[4, 5, 6]
[10]
[15, 16, 17, 18]
[22]
[25, 26, 27, 28]
```

## <a name="note_for_python3">Changes for Python 3</a>

1.  I can't find the reference, but it seems that `map` and some other
    functions that returned list objects in python2 now return iterators,
    necessitating coercing the iterator returned by `map()` into a list for our
    purposes.
1.  `print` requires parenthesis!
1.  `lambda` has a different format
