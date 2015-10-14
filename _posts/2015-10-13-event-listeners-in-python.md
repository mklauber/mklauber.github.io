---
layout: post
title: "An Event System in one line of Python"
date:   2015-10-13 16:41:27
categories: patterns game-dev
---

<p class="aside" markdown="1">This pattern is called "Observer".  It's very old, I'm just pleased with how simple it is to implement using python's builtin structures.  You can learn more about it [here](http://gameprogrammingpatterns.com/observer.html).</p>

Just wanted to jot down here something that I found very clever.  Using a couple datastructures from the python standard libary, and a little known fact about functions in python, it's possible to build an event system in a single line of code.

Python has an incredible standard library, with a lot of the basic datatypes you'd use built in.  In addtion to basic types like ints, strings, floats, and decimals it also include a wide variety of optimized data structures.  Lists, dicts(hashes), queues, and sets.  

It also has a convienence datatype known as a `defaultdict`.  A `defaultdict` when created, takes a function that returns the default value if you try to access a key before it is assigned, it will assign the result of that function to the key and then return it.  An example may help.

{% highlight python linenos %}
>>> from collections import defaultdict
>>> d = defaultdict(lambda: 10)
>>> d[0]
10
>>> d[1] + 1
11
>>> d[2] = 14
>>> d[2]
14
>>> 
{% endhighlight %}

As you can see in that example, if we try to access a key before it's assigned, we'll get back the result of calling the function passed to the constructor of `defaultdict`.  If we assign to a key, that value is updated, and will be returned in future instead.  

Another fact to know about python is that complex data structures such as lists and sets are stored by reference.  This means that if you call a function on a set that modifies that set, anywhere that set is referenced is also updated.

{% highlight python linenos %}
>>> a = set()
>>> a.add(1)
>>> a
set([1])
>>> b= a
>>> b.discard(1)
>>> a
set([])
>>>
{% endhighlight %}

with this, we can achieve the following.

{% highlight python linenos %}
>>> d = defaultdict(set)
>>> d['key'].add('value')
>>> d
defaultdict(<type 'set'>, {'key': set(['value'])})
>>> d['key']
set(['value'])
{% endhighlight %}

<p class="aside">
I promise this all ties into an event system soon.
</p>

Now we've got a data structure that allows you to add arbitrary values to arbitrary keys, without having to check if the key exists, or if the value is already present.  In python dictionary keys and the values in sets have to be hashable.  This has a specific meaning.  It is a integer number that is guarenteed to a) differ when two objects differ, and b) be identical when two objects are identical.  Now, not all objects are hashable.  The general rule is that if an object contains other objects, they aren't hashable.  So lists, sets, and dicts are all not hashable and thus can't be keys in dicts or stored in sets.  This is because these objects are stored by reference, and thus two lists containing identical objects could still have different references.  



But the important fact about python that makes all this work is that a fuction is hashable.  functions are first class objects, you can pass them around, pass a function to another function (like we did for creating a defaultdict), or anything else you can do with any other object.  And since they're hashable, we can store them in sets.  Which leads to this.

{% highlight python linenos %}
>>> listeners = defaultdict(set)
>>> def handle_test(arg):
...  print arg
... 
>>> listeners['test'].add(handle_test)
>>> for listener in listeners['test']:
...  listener('Hi')
... 
Hi
>>> listeners['test'].add(handle_test)
>>> for listener in listeners['test']:
...  listener('Hi')
... 
Hi
>>> for listener in listeners['bad']:
...  listener('Hi')
... 
>>> listeners['test'].discard(handle_test)
>>> 
>>> for listener in listeners['bad']:
...  listener('Hi')
... 
>>> 
{% endhighlight %}

That's a single data structure where you just add the listener for an event.  If there aren't any listeners yet, it'll create the set for you.  If you've already added this listener, it will ensure it's only present once.  And since python's `set.discard` method does nothing if the argument isn't in the set, you can remove handlers as many times as you want too.  

<p class="aside">
As a warning, be careful having events trigger other events, it's quite easy to infinitely recurse if event 'A' triggers function 'a' which emits event 'B' which triggers function 'b' which emits event 'A' and so on.
</p>

Now, emitting a message does require two lines and a for..in loop, but honestly, that doesn't bother me that much.  It's possible to subclass `defaultdict` to add a `emit` method, but honestly, I find that tends to clutter up the stack trace if your code emits a lot of events, especially if events trigger other events.  




