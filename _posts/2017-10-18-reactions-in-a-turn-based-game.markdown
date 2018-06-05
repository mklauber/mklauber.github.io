---
layout: post
title: "Implementing Reactions in Hoplite"
date:   2015-10-13 16:41:27
tags: patterns game-dev hoplite
---
In most turn based games, each actor gets to take a single action on their turn.  So the game loop is pretty simple.

{% highlight python linenos %}
class Engine(object):
    def next_turn(self):
        actor = self.get_next_actor()
        action = actor.get_action()
        action.execute(self.state)
        return action
{% endhighlight %}

So each turn, we get the next actor, ask them what action they want to take, and then have the action change the state.  
Pretty simple, pretty straight forward.  We'd wrap that in some kind of `while not won or loss: engine.next_turn()` and
call it good.  And our history would look like this:

{% highlight json linenos %}
[
  <Warrior Move (0,1)>,
  <Archer Move (2,1)>,
  <Hero Move (1,1)>
]
{% endhighlight %}

However, Hoplite is an unusual turn based game.  The Hero does not have an actual `Attack` actions.  All his attacks are
triggered by movements.  The attacks are **reactions** to the `Move` action.  And a lot of the game is focused around
positioning the Hero and his enemies so that the Hero can eliminate multiple targets in a single turn.  It makes little
sense then, to consider each turn as a single action, but rather as a collection of actions.  In my game state, a turn
looks like this:

{% highlight json linenos %}
[
  [<Warrior Moves (0,1)>],
  [<Archer Moves (2,1)]>,
  [<Hero Moves (1,1)>, <Hero Slashes (0,1)>, <Hero Lunges (2,1)>...]
]
{% endhighlight %}

Now the important thing to notice here, is that the user is only responsible for generating the first action.  Every
other action is a reaction.  So our engine needs to calculate all the reactions.  So our engine starts to get more
complicated.

{% highlight python linenos %}
class Engine(object):
    def next_turn(self):
        actor = self.get_next_actor()
        action = actor.get_action()
        reactions = calcuate_reactions(action)
        turn = [action] + reactions
        for action in turn:
            action.execute(self.state)
        return turn
{% endhighlight %}

We get the action same as before, and we though some mystical process, calcualte the reactions the action triggers.
Then we execute all those actions to update the state, and then we return the turn.But there's still a problem.  A
reaction may trigger a addtional subsequent reaction.  Our game engine, when calculating turns, needs to also calculate
any triggered reactions.  So we end up with something like this:

{% highlight python linenos %}
class Engine(object):
    def next_turn(self):
        actor = self.get_next_actor()
        turn = []
        actions = [actor.get_action()]
        while len(actions) > 0:
            action = actions.pop(0)
            reactions = calcuate_reactions(action)
            actions.append(reactions)
            action.execute(self.state)
            turn.append(action)
        return turn
{% endhighlight %}

Now this is pretty close to what we want.  We pop with the first action, and check if it triggers any reactions.  If
it does, we add those to the list of actions, and then update state and add the action to the turn.  Then we check the
next action and continue until there are no more actions left.  But there is one thing wrong with this.  When an action
triggers reactions, you want those reactions to happen immediately after the action that triggered them, delaying any
further other actions to later.  This is because from a player perspective, it's easiest to follow the activity if
actions that are proximate in cause are proximate in time.  Therefore, we need to change line 9 above to insert the
reactions immediately after the action.

{% highlight python linenos %}
class Engine(object):
    def next_turn(self):
        actor = self.get_next_actor()
        turn = []
        actions = [actor.get_action()]
        while len(actions) > 0:
            action = actions.pop(0)
            reactions = calcuate_reactions(action)
            actions.insert(0, reactions)
            action.execute(self.state)
            turn.append(action)
        return turn
{% endhighlight %}

And now we're done.  The engine calculates turns correctly, checking if each action triggers reactions, and if so, inserts
them after the triggering action.  This process continues until all reactions are complete, and we have a full turn.
