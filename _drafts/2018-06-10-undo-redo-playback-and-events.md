---
layout: post
title: "Undo, Redo, and Turn Based Game Engines"
date:   2018-06-10 16:41:27
tags: patterns game-dev hoplite
excerpt: "Adding Undo functionality to your game after the fact is nigh impossible.  But starting with it is so
simple that once you do you'll wonder how you ever managed to write code without doing so."
---

<p class="aside" markdown="1">This is the "Command" pattern.  It's a Gang Of Four pattern, but explaining it with an
example is much easier to understand than trying to grok the original documentation. </p>

So if you ever, ever, look into implementing undo and redo for any purpose, you'll run into the `Command` pattern.  The
pattern is presented as rather complex, but in reality it's quite simple to explain.  There's two pieces to this pattern.
First, there's a `state` object that describes the current state of game world.  Second, there's a bunch of "Command"
objects that describe changes to be executed against the state.

Each time the user takes and action, the system generates a `Command` and then executes it against the `State`.  It then
stores the command in a "undo" stack.  To undo an action, the system pops the last `Command` off the "undo" stack,
executes the `rollback` method of the `Command` against the `State`, and stores the action on a "redo" stack.  To redo
an action, it's just the reverse.  Pop the next action off the "redo" stack, `execute` it against the current state, and
put it back on the "undo" stack.

Let's go ahead and create a few example objects.  Let's start with a example state for a simple battle game.  The state
will be made up of a dictionary of game elements.  Each element will be of the form `ID:Health`.  In this case, we're not
even going to use a complex data structure, since our state is so simple.

<p class="aside" markdown="1">It can be quite useful to implement state using json safe data types.  Doing so makes it
simple to dump the current state to the console or file.  A side effect of it being simple to dump the current state to
a file is you're halfway to having saving and loading games implemented.</p>

{% highlight python linenos %}
{
	"Hero": {"health": 3},
	"Enemy-1": {"health": 1},
	"Enemy-2": {"health": 1},
	"Enemy-3": {"health": 1}
}

{% endhighlight %}

Now let's create an Attack Command. The important thing is that the Command remembers the change it's responsible for
making.  So in python, the simplest way to implement this is as an object.  The `__init__` method records the parameters
of the change, and then the `execute` method accepts a state object and performs the modification requested to the
provided state.  The rollback method uses the same parameters to make the opposite change to the provided state.

<p class="aside" markdown="1">Occasionally, in order to implement undo, you'll need to record something about the state
prior to modifying it during the execute method.  A prime example of this is when units die.  When you run the
`execute` method on the `Die` action, you'll want to record the stats of the unit your removing.  Then during the
`rollback` method, you can use that info to place it back on the battlefield.</p>
{% highlight python linenos %}
class Attack(object):
	def __init__(self, target, damage):
		self.target = target
		self.damage = damage

	def execute(self, state):
		state[self.target]["health"] -= self.damage

	def rollback(self, state):
		state[self.target]["health"] += self.damage

{% endhighlight %}

The most important aspect of implementing the `Command` pattern is that commands should be ____________.  This means
that if you take a Command `C`, call it's `execute` method against a State `S` getting State `S'`, and then calling the
it's `rollback` method against State `S'` should give you `S` again.  This means that your commands will result in the
same final state no matter how many times the user clicks undo and redo.

Here we should describe the changes to state as the Attack action is executed and rolled back.


Within Turn Based Game Engines, there's a couple gotchas that you need to watch out for.  The biggest one is randomness.
Many games include a element of randomness when determining if attacks hit or for calculating damage numbers.  If you
intend to include this randomness in your game, calculate it prior to the `execute` method.  If it's included during the
`execute` method, undoing and redoing an attack could cause it to do more or less damage that it did initially.

In the next article, I'll describe my implementation of the core game engine, which thanks to this design, needs to know
almost nothing about the actual design of the game.
