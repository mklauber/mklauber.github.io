---

layout: post
title: "Control your UI with State Machines"
date:   2015-10-15 13:14:27
tags: hoplite game-dev
author: mklauber
---
Managing the various screens and menus of your UI can be greatly simplified by using a Finite State Machine(FSM) to control the flow of the UI.  FSMs are easy to model and work very well for modeling the flow of a user's action throughout a game.  Furthermore, they compartmentalize code specific to each screen, making it simpler to reason about changes to any specific screen.
<!--more-->

Let's start with a simple example.  You're going to write a game of asteroids.  Players can start a new game, load an old game, save the current game, and view high scores.  (the mechanics of scoring a game of asteroids is immaterial).  Immediately, looking at the actions we just described, we can identify several independent interfaces.  And if two or more things are independent, we want their code to be independent.  So let's start out by writing skeletons for each independent interface.


{% highlight python linenos %}
def title_screen(screen, input):
    while True:

        # Paint the screen
        screen.clear()
        screen.put_text("start new game", 0,0)
        screen.put_text("load game", 10,0)
        screen.put_text("view high scores", 20,0)

        # Handle input
        for key in input: # input is a queue of events from the UI library.
            if key == "S":
                # Code here for running a game
            elif key == "L":
                # Code here for loading a game
            elif key = "H":
                # Code here for viewing high scores
            elif key = "Q":
                # Code here to quit the game

def game_screen(game, screen, input):
    while True:
        screen.clear()
        game.progress()
        screen.render(game)
        for key in input:
            game.handle(key)
        if game.hasWon():   # Go to the next_level
            # Go to the next_level
        if game.hasLost():
            # Go to the title_screen, or the high score screen

def highscore_screen(screen, input):
    while True:
        screen.clear()
        for i, score in enumerate(global.high_scores):
            screen.put_text(score, 10*i, 0)
        for key in input:
            if key == "Q":
                # return to title screen

if __name__ == '__main__':
     title_screen(screen, input)

{% endhighlight %}

Ok, there are some quick, basic descriptions of how to display different screens, and respond to input.  I want to point out something that we've immediately gained from separating these screen functions out.  The input handlers and rendering code for each function doesn't need to know about each other.  Keys that close the game on one screen can instead be back buttons on other screens.  

Now, let's look at how to move from one screen to another.  A first instinct might be to just call the code for the new screen from within the old screen.  

{% highlight python linenos %}
def highscore_screen(screen, input):
    while True:
        screen.clear()
        for i, score in enumerate(global.high_scores):
            screen.put_text(score, 10*i, 0)
        for key in input:
            if key == "Q":
                title_screen(screen, input)

{% endhighlight %}

Can you see the memory leak in here?  Every time you switch screens, you end up adding another layer to the stack.  The solution is to unwind the stack before calling the new screen.  To do that, we need to return not the execution of the screen, but a reference to the screen itself.  We also need to change `__main__` to call each new screen.

{% highlight python linenos %}
def title_screen(screen, input):
    while True:

        # Paint the screen
        screen.clear()
        screen.put_text("start new game", 0,0)
        screen.put_text("load game", 10,0)
        screen.put_text("view high scores", 20,0)

        # Handle input
        for key in input: # input is a queue of events from the UI library.
            if key == "S":
                game = new_game()
                c
            elif key == "L":
                game = load_game()
                game = new_game()
            elif key = "H":
                return highscore_screen
            elif key = "Q":
                return

def game_screen(game):
    def screen(screen, input):
        while True:
            screen.clear()
            game.progress()
            screen.render(game)
            for key in input:
                game.handle(key)
            if game.hasWon():   # Go to the next_level
                game = new_game()
                return game_screen(game)
            if game.hasLost():
                if score > global.high_score:
                    global.high_score = score
                    return highscore_screen
                else:
                    return title_screen
    return screen

def highscore_screen(screen, input):
    while True:
        screen.clear()
        for i, score in enumerate(global.high_scores):
            screen.put_text(score, 10*i, 0)
        for key in input:
            if key == "Q":
                # return to title screen

if __name__ == '__main__':
     cur_scr = title_screen
     while cur_scr != None:
        cur_scr = cur_scr(screen, input)

{% endhighlight %}

Do you see what we did there?  Instead of calling the new screen from the old screen, we return a reference to the screen.  Then in the `while` loop in `__main__`, we call each new function returned from the old one.  This makes it very easy to move directly from one interface to another without cluttering the stack or having to worry about where we will return to.  And if we ever want to exit the game, we just return `None` and the interface game loop will end.  

> One thing to highlight in this code is the changes we needed to make to the game_screen method.  Specifically, on line 23, we change game_screen from a function that runs the screen directly into a closure that encloses a game object and returns a function that runs that game as a screen.  This was just a convenience for us, as a way of providing the game object while not changing the signature of the function.  
>
> We could just as easily use classes, passing game to the `GameScreen` constructor, and then return the constructed instance.  We'd then change the `while` loop to invoke a method on each screen object, and that method would be what actually runs the interface.

{% highlight python linenos %}
class GameScreen(object):
    def __init__(self, game):
        self.game = game

    def execute(screen, input):
        while True:
            screen.clear()
            game.progress()
            screen.render(game)
            for key in input:
                game.handle(key)
            if game.hasWon():   # Go to the next_level
                game = new_game()
                return GameScreen(game)
            if game.hasLost():
                if score > global.high_score:
                    global.high_score = score
                    return HighscoreScreen()
                else:
                    return TitleScreen()

if __name__ == '__main__':
    cur_scr = TitleScreen()
    while cur_scr != None:
        cur_scr.execute(screen, input)
{% endhighlight %}

One additional fact worth pointing out is that by switching to this method of invoking screens, we've actually not lost the flexibility to directly call one screen from within another.  For example, consider if we want to be able to view the high scores during gameplay.  If we just return a reference to the `highscore_screen` when we leave the highscore screen, we'll go to the title screen.  Which means we've lost our place in the game.  

{% highlight python linenos %}

def game_screen(game):
    def screen(screen, input):
        while True:
            screen.clear()
            game.progress()
            screen.render(game)
            for key in input:
                if key == "H":
                    highscore_screen(screen, input)
                game.handle(key)
            if game.hasWon():   # Go to the next_level
                game = new_game()
                return game_screen(game)
            if game.hasLost():
                if score > global.high_score:
                    global.high_score = score
                    return highscore_screen
                else:
                    return title_screen


    return screen

{% endhighlight %}

Now, on line 8, if we press "H" during gameplay, we invoke `highscore_screen` right there in the middle of `game_screen`, and use the stack to return to exactly the same place in the game we were before the "H" key was pressed.  The game can then continue, and we've managed to track exactly where we came from before we went to the highscore screen.

Thus, we've got a few great tools
