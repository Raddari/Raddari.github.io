---
layout: post
title: "Reversing Minesweeper: The Beginning"
author: Harvey Wyatt
categories: reversing minesweeper
---
As a kid, I loved clicking random boxes in minesweeper until I inevitably lost. As an adult I like to think I'm a little better at the game 😳

![Minesweeper win](/assets/images/minesweeper/win.png)
*Oh yeah 😎*

In all seriousness I love minesweeper and I think I like reverse engineering. I've never really done it before (other than using cheat engine to cheat on Roblox back in the day... shhh) so I thought why not give it a crack on the O.G. Windows XP minesweeper. My hope is that I can eventually rewrite it in C++. Should be easy enough, right?

Unfortunately I already made some progress before deciding I should write about it. Oops. Anyway, using Ghidra and Cheat Engine I've identified the function that executes when the game ends and cleaned it up with some enums and renaming.

Right now, it looks a little something like this:
```c
void game_over(int reason) {
  g_game_state = GAME_STATE_ENDED;
  g_smiley_state = (reason != 0) + 2;
  draw_smiley(g_smiley_state);
  FUN_01002f80((reason != 0) * '\x04' + 10);
  if ((reason != 0) && (DAT_01005194 != 0)) {
    FUN_0100346a(-DAT_01005194);
  }
  play_sound(SOUND_TYPE_BOMB - (reason != 0));
  _DAT_01005000 = 0x10;
  if ((reason != 0) && ((ushort)DAT_010056a0 != 3)) {
    if (g_timer < (int)(&DAT_010056cc)[(ushort)DAT_010056a0]) {
      (&DAT_010056cc)[(ushort)DAT_010056a0] = g_timer;
      FUN_01001b81();
      FUN_01001baa();
    }
  }
  return;
}
```
(I'm using a `g_` prefix to name global variables that Ghidra usually shows as `DAT_########`)

Pretty nasty stuff 🤮 Let's break it down:
```c
g_game_state = GAME_STATE_ENDED;
g_smiley_state = (reason != 0) + 2;
draw_smiley(g_smiley_state);
```
Game state is pretty self-explanatory. `g_smiley_state` and `draw_smiley` refer to the icon at the top of the screen which displays various emotions based on how terribly you play. How cute.

At first, `(reason != 0)` had me stumped (especially before I worked out what *reason* actually was in the decompiled output 😳). However, there is a line of code later which clarifies what is going on here. For now just know that the last two lines set the icon to either a sunglasses face when you win or a very unhappy x-eyed face when you click a bomb.

```c
FUN_01002f80((reason != 0) * '\x04' + 10);
if ((reason != 0) && (DAT_01005194 != 0)) {
  FUN_0100346a(-DAT_01005194);
}
```
I honestly have no idea what is going on here yet. Next.

```c
play_sound(SOUND_TYPE_BOMB - (reason != 0));
```
Interesting. When `(reason != 0)` evaluates to `false`, the bomb sound effect is played. We can then infer that `reason == 0` indicates a loss. Going back to `g_smiley_state = (reason != 0) + 2` we can assume `g_smiley_state` is `2` on a loss, and `3` on a win. Progress!

Using the above info, I created a nice little enum to contain what we know about the smiley states so far:

![Smiles all around :D](/assets/images/minesweeper/smiley-state.png)

And Ghidra is so cool that it changed the line to `g_smiley_state = (reason != 0) + SMILEY_STATE_LOSER` automatically. Love that 😍

```c
_DAT_01005000 = 0x10;
```
No idea lol

```c
if ((reason != 0) && ((ushort)DAT_010056a0 != 3)) {
  if (g_timer < (int)(&DAT_010056cc)[(ushort)DAT_010056a0]) {
    (&DAT_010056cc)[(ushort)DAT_010056a0] = g_timer;
    FUN_01001b81();
    FUN_01001baa();
  }
}
```
We know that `(reason != 0)` indicates a win. The next line then checks if the timer is lower than a value in some array. That leads me to believe that this piece of code updates the leaderboard whenever you get a best time. Currently the leaderboard looks like this:

![The fastest sweepers](/assets/images/minesweeper/leaderboard.png)
*Not only do I have to beat minesweeper, but I have to do it in less than 999 seconds...*

So I DEMOLISHED a round of **beginner**, and then...

![Enter leaderboard name](/assets/images/minesweeper/enter-name.png)
*I'm not very creative 😳*

![Updated leader](/assets/images/minesweeper/updated-leader.png)
*w00t!*

Now I open up cheat engine and DEMOLISH a round after breaking on the comparison, which looks a little like this:

![Minesweeper assembly](/assets/images/minesweeper/assembly.png)
*I should really read up on assembly a ~~bit~~ lot more*

Cheat engine helpfully shows the dereferenced values in the comment column which confirms my suspicions: `DAT_010056cc` is an array containing the best times for each difficulty. Since the difficulty is explicitly checked to `!= 3`, I'm going to assume that `beginner == 0`, `intermediate == 1`, `expert == 2` and `custom == 3` seeing as there is no custom leaderboard.

Knowing that, we can clean up the decompiled output a bit with some enums and renaming:
```c
void game_over(int reason) {
  g_game_state = GAME_STATE_ENDED;
  g_smiley_state = (reason != 0) + SMILEY_STATE_LOSER;
  draw_smiley(g_smiley_state);
  FUN_01002f80((reason != 0) * '\x04' + 10);
  if ((reason != 0) && (DAT_01005194 != 0)) {
    FUN_0100346a(-DAT_01005194);
  }
  play_sound(SOUND_TYPE_BOMB - (reason != 0));
  _DAT_01005000 = 0x10;
  if ((reason != 0) && (g_game_difficulty != GAME_DIFFICULTY_CUSTOM)) {
    if (g_timer < g_leaderboard[g_game_difficulty]) {
      g_leaderboard[g_game_difficulty] = g_timer;
      FUN_01001b81();
      FUN_01001baa();
    }
  }
  return;
}
```

Already looking much better! Next step is to figure out what `FUN_01001b81` and `FUN_01001baa` are doing.
```c
void FUN_01001b81(void) {
  DialogBoxParamW(g_app_handle,(LPCWSTR)0x258,g_window,FUN_0100181f,0);
  _DAT_0100515c = 1;
  return;
}
```
Looks like a dialog popup, most likely where you enter your name for the leaderboard. Looking at the microsoft docs, there's a callback function `FUN_0100181f` which may reveal some more info.

```c
void FUN_01001baa(void) {
  DialogBoxParamW(g_app_handle,(LPCWSTR)0x2bc,g_window,FUN_010016fa,0);
  return;
}
```
Similar the other function above, I suspect diving into the callback function may help decipher what this is doing.

That's all I got for now, unfortunately. I have way too much uni work to do, and nowhere near enough time to do it 😳
I can't say when I'll get around to part 2, but stay tuned!

\- raddari