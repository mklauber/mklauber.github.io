---
layout: post
title: "Separating the Engine and UI"
date:   2018-06-08 16:41:27
tags: patterns game-dev hoplite
---

When I started to create my game, I knew I wanted a few basic things.  I wanted the game engine to be completely
separate from whatever UI I wrapped around it.  I wanted to be able to switch the game between a graphical or
text based UI.  I also decided that I wanted the UI to drive the engine, passing in Actions for actors who required
input.

<p class="aside" markdown="1">I decided to have the UI drive the engine based on Bob Nystrom's post
<a href="http://journal.stuffwithstuff.com/2014/07/15/a-turn-based-game-loop/">A Turn-Based Game Loop</a>.  If you're
interested in roguelike and turn based game design, I cannot recommend his blog, or his book
<a href="http://gameprogrammingpatterns.com/">Game Programming Patterns</a> highly enough.</p>

So after I'd finished up the design of my game engine, I knew I needed a UI wrapped around it.  I'd written the engine
with the idea that the UI would drive the engine, passing in actions for actors who required input, and drawing the
state after each action.  With this in mind, my engine was designed to announce every action taken during the game.
Ignoring user input for the moment, I ended up with a first round where my UI looked like this.

{% highlight python linenos %}
class Engine(object):
    def __init__(self):
        self.state = State()    # Ignore the internals of how I manage state here
        self.listeners = set()

    def announce(self, action):
        for listener in self.listeners:
            listener(self.state, action)

    def execute(self, action):
        pass                    # Ignore how I modify state here

    def progress():
        actor = self.get_next_actor()   # Get an actor
        action = actor.get_action()     # Get the actors action
        self.execute(action)            # Update the state
        self.announce(action)           # Announce the action to anyone interested


class UI(object):
    def __init__(self, engine):
        self.engine = engine
        self.engine.listeners.add(self.handle_events)

    def render(self, state, action):
        self.screen.clear()             # This is a vast oversimplification of how I draw things, But
        for element in state:           # that's fine for now.  I just wanted to show that at the start,
            self.screen.draw(element)   # I'd clear the screen, and then add each element from the state


    def run_game(self):
        while self.engine.is_finished == False:
            self.engine.progress()

{% endhighlight %}

So if we look at the flow of control in the above example, we can see that the UI in it's `__init__` method adds it's
`handle_events` method as a listener for Engine state changes.  Then `run_game` tells the engine to progress.  Each time
the engine progresses, it get's an actor, get's the actor's action, updates the state, and then announces that action to
the UI.  The UI object's `render` method then takes control back and renders the new state, before returning to the
engine, which finishes up and returns to `run_game`.  Run game repeats the process until the engine reports that the game
is over, and then the UI exits.  

The nice thing about this is that nothing in the engine cares about the UI.  In fact, you can run the engine entirely
without a UI, taking the final loop in `run_game` and running it manually.  This brings with it the nice benefit that you
can implement multiple UIs for the game and easily interchange them.  To start of with, I wrote a terminal based UI using
curses.  This was nice because it was relatively easy to setup, and quickly got me into a code -> run -> test -> code loop.
I could test things quickly, and it allowed me to make quick progress on the game engine.  However, terminal graphics and
interfaces don't suit the game as well as mouse/touch based graphics, so implementing the engine in this way allows me to
switch to implementing the UI in pyGame or similar without having to refactor the engine.  
