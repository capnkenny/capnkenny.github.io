---
layout: single
title: "CHIP-8 in NovelRT"
date: 2020-05-15 07:00:00
excerpt: A dive into creating an emulator in the NovelRT engine.
---
  
<img src="https://user-images.githubusercontent.com/10300290/80096502-ab308280-8537-11ea-881c-32d1cf62463e.png" />
  
*(Before reading, please note that at the time of writing, I am currently affiliated with the NovelRT project. Insert shameless plug here.)*
  
Recently, I found myself at a point in time where I felt that I wanted to do more than just one project. Normally, I tend to bounce from one to the other, but in this case I wanted to complete something before moving on.
  
*Enter CHIP-8 emulators.*  
  
For me, emulators have always been fascinating. As a child, the thought of making my PC act like a Sega Genesis or a SNES blew my mind - even up to the point that one of my first OpenGL programs was to draw the symbolic Gamecube "G" logo as close as possible to how the console portrayed it (looked nice but wasn't close). With that said, it had currently been around a year or so that I had last taken a stab at writing the basic of basic emulators - one for the "CHIP-8" console. I knew that I had attempted to do it before (in XNA/MonoGame with C#, with LibGDX in Java, etc.) but I had never tried to write one in C++, and I really wanted to see if I could write one that would be "complete" enough to actually play a few ROMs.
  
  With that said, typically you don't use game engines for this. The reasons being (at least from the last time I had read were that):
  1. You don't use nearly half the features of the engine, introducing bloat, and
  2. You induce more overhead than needed, which can typically degrade performance. (Especially with newer systems being emulated)
  
  After not heeding the two warnings, and deciding that recreating a CPU with 4KB of RAM and the least amount of instructions I could ask for would be more fun, I set out to look for what I would make it in. And in 5 minutes, the verdict was in...
  - ~~Unity~~ (No C++)
  - ~~Unreal Engine~~
  - ~~CryEngine~~
  - NovelRT
  Why not make it in the engine I'm working on!? Surely nothing can go wrong, right?
  
## Going into NovelRT
  
  Since I'm (at the time of this being written) a member of the NovelRT Group, I guess it would be expected that a shameless plug is inserted into this post (it won't happen every time, I promise).
  
  For those who don't know, *NovelRT* is a cross-platform 2D game engine that is designed with an emphasis on Visual Novels (*cough*Novel*cough*) that can be found [here](https://github.com/novelrt/NovelRT). It can, however, also be used for other uses as well - [touhou] games, platformers, or anything that is in 2D perspective. And/or a game (I guess it doesn't actually have to be?). Regardless, knowing how much work we've put into it so far, I wanted to make something a little bit different than what was intended, and see how it reacts when using it to simulate a CPU and display for CHIP-8.  
  
  So going forward, the plan of action was as such...
  - Implement the CPU and RAM
  - Get graphics working
  - Hook into input for key detection
  - Make a *BEEEEP!* sound via the engine*
  
  And on that note, I already knew I ran into a problem, hence the asterisk.  
  Being the person who designed the audio portion of NovelRT, I knew right off the bat that it wasn't designed to generate sound, it was only designed to load sound files (and poorly at that until I refactor it...) - I'll have to bypass it to make the sound when needed. Not terrible, but not what I wanted to start off with when I purposely wanted to ***use the engine***. Nonetheless, I'll carry on...
  
## Creating the Simulation
  
  I'll be bluntly honest here - there was no way I was just diving into emulation without a head start and some help.  
    
  Google became my best friend, and between that and r/EmuDev, I was reading about implementing opcodes, how the memory was to be mapped, etc. in no time.  
  Not using bitwise operations made me feel a bit lost for a while, however with implementing the instructions *a lot* of them would be similar, making it easier and easier to not rely on the tutorials as much. With that said, I didn't mind as much if I was using the concepts that I was learning, I just didn't want to copy the code one for one - copying and pasting isn't a way to learn, and sometimes tutorials make it inevitable >.< .
  
  Continuing on past the normal opcodes, there were a few of them that I was more concerned with - namely, the opcodes that pertained to:
  a) Drawing,
  b) Input, and
  c) the Sound Timer.
  
  The Sound Timer one wasn't relatively too hard - once we decrement the timer to 0, we make a beep noise, and we reset the timer accordingly.  
  For Input, it was somewhat simple as well - for one opcode, if there's input, then react. The other instruction - if there's no input, stop everything until we get it. But from there on, I needed a way to grab input. Thankfully, NovelRT provided KeyCodes that I could map to, and since they would be polled automatically by it's Interaction Service, I'd just ask to get the current state of the key as such:
  ```
  _input.lock()->getKeyState(NovelRT::Input::KeyCode::One);
  ```
  As long as this was updated after a cycle, we would be able to accurately get the key presses and allow the CPU to respond accordingly. Drawing, on the other hand, probably couldn't have been more difficult due to the way I handled it.
  
## Drawing the Simulation

  To start, I'm going to fully admit that I was only trying to get what I could done between the times of 6AM - 8AM every weekday just so that I could finish up before work started, and I'd have time to eat and shower. In retrospect, I shouldn't rush the design process, nor should I assume that what works in a tutorial will work for me.
  
  
  
  
  [touhou]: https://github.com/novelrt/touhou-novelrt
