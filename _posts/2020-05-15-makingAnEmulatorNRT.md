---
layout: splash
title: "CHIP-8 in NovelRT"
date: 2020-05-15 07:00:00
excerpt: A dive into creating an emulator in the NovelRT engine.
header:
  overlay_image: https://user-images.githubusercontent.com/10300290/80096502-ab308280-8537-11ea-881c-32d1cf62463e.png
---

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
  
  In a few different CHIP-8 programs that I had found, it appears that they would store the display in a size 2048 array by doing the equivalent of:
  ```
  std::array<unsigned char, 2048> gfx;
  ```
  Meaning that since the display for CHIP-8 was 64x32, we would be storing the entire display in a one-dimension array for a 2D plane. Unwrapping this properly is where the fun part begins :D  
  
  The simplest way I could pull this off was to use a nested for-loop - one to control the x-axis, and one to control the y-axis. Then, if the "pixel" in the display array was "active" (think 1s and 0s), it would display white; if not, it would display black. Sounds simple enough, but I already had to use a workaround at step 1...
  
  <img src="https://cdn.discordapp.com/attachments/543898968942444573/703722544821305364/expected.PNG" />
  By default, without changing engine code the engine outputs a blue background - *awesomeeee...*
  
  So, creating one big rectangle to block it out will work for now:
  ```
  auto bkgdTransform = NovelRT::Transform(origin, 0, NovelRT::Maths::GeoVector2<float>(1920, 1080));
  auto bkgd = runner.getRenderer().lock()->createBasicFillRect(bkgdTransform, 3, NovelRT::Graphics::RGBAConfig(0,0,0,255));
  ```
  
  Next is to actually make the "pixels" so that we can draw what the CPU tells us to:
  ```
	//Setup gfx and input
	float screenH = 1080.0f;
	float screenW = 1920.0f;
	auto origin = NovelRT::Maths::GeoVector2<float>(screenW / 2, screenH / 2);
	
	//Get Pixel and Increment Dimensions
	auto pixelWidth = screenW / 64;			
	auto pixelHeight = screenH / 32;
	auto incrementX = 30.0f;			//X and Y work off of midpoints
	auto incrementY = 33.75f;
	...
	std::array<std::array<std::unique_ptr<NovelRT::Graphics::BasicFillRect>, 64>,32> pixels = 
		std::array<std::array<std::unique_ptr<NovelRT::Graphics::BasicFillRect>, 64>, 32>();

	//Create pixels in 2D array
	for (int y = 1; y <= 32; y++)
	{
		auto pixelsX = std::array<std::unique_ptr<NovelRT::Graphics::BasicFillRect>, 64>();
		auto pixelOrigin = NovelRT::Maths::GeoVector2<float>();
		if (y == 1)
		{
			pixelOrigin = NovelRT::Maths::GeoVector2<float>(incrementX / 2, (incrementY / 2));
		}
		else
		{
			pixelOrigin = NovelRT::Maths::GeoVector2<float>(incrementX / 2, incrementY);
		}
		for (int x = 0; x < 64; x++)
		{
			auto transform = NovelRT::Transform(pixelOrigin, 0, NovelRT::Maths::GeoVector2<float>(pixelWidth, pixelHeight));
			pixelsX[x] = render.lock()->createBasicFillRect(transform, 2, NovelRT::Graphics::RGBAConfig(255,255,255,0));
			incrementX += pixelWidth;
			//Shift the pixels into alignment with the screen
			if (x != 0)
			{
				pixelsX[x]->transform().position().setX(pixelsX[x]->transform().position().getX() - (pixelWidth / 2));
			}
			if (y != 1)
			{
				pixelsX[x]->transform().position().setY(pixelsX[x]->transform().position().getY() - (pixelHeight / 2));
			}
			pixelOrigin.setX(incrementX);
		}
		incrementX = 30.0f;
		incrementY += pixelHeight;
		auto point = y - 1;
		pixels[point] = std::move(pixelsX);
	}
  ```
  Here, you'll see that I...  
  1) Set the screen's width and height,  
  2) Create the pixel width and height,  
  3) Set up an amout to increment X and Y by, and  
  4) Go through the process of creating the transforms and rectangles (known as `BasicFillRect`s in NovelRT).
  
  Now, if you read through, you'll notice that I hardcode the screen size to 1920x1080. I *do* have the option of pulling the screen size directly from the display via NovelRT's `WindowingService`, however this is more of a misnomer - the 1920x1080 size is actually what NovelRT uses in world-space, which allows CHIP-8 to resize without having to change much of anything to do so.  
    
  Next, you'll see that I break up the for-loop a bit with multiple if-else statements. This is due to the fact that the transforms work off of midpoints when rendering, and the engine follows the top-left methodology when rendering, so if I were to specify a rectangle's origin at `incrementX` and `incrementY` (instead of halving them) everything would shift be shifted downwards and to the right.
  
  After massaging the code a bit so that the transforms initialized in their correct placements, and loading in one of the variants of Pong...
  
  <img src="https://cdn.discordapp.com/attachments/556617985209532458/701613232061939722/unknown.png" />
  *WE HAVE LI-*  
  <img src="https://cdn.discordapp.com/attachments/556617985209532458/701613362282365009/unknown.png" />  
  ...ftoff.  
    
  Welp, Rome wasn't built in a day and neither was this, so time for debugging. Thankfully, this was an error with how I wrote the instruction, not how I was trying to render it :)  
    
  Some more debugging, and...  
  <img src="https://user-images.githubusercontent.com/10300290/82118827-a8483c80-9747-11ea-9fba-530125868bb6.png" />  
  Woohoo! It's almost done!  
    
  Finally, it was time to do some neatening up and add the sound like I had expected to do. I'm more than thankful that most of this is just handled in `CPU.cpp`, as this is just a few moments of cleaning up output...  
  <br /><img src="https://user-images.githubusercontent.com/10300290/82119175-64a30200-974a-11ea-960f-6735fe58600a.png" alt="Pounding the shell with logs of each Opcode being executed" height="240" width="207" />	Debug vs. Release	<img src="https://user-images.githubusercontent.com/10300290/82119174-64a30200-974a-11ea-8fa1-882b75b26796.png" alt="Nice clean one-time output :D" height="240" width="400" />
	
  and making sure that when sound is initialized as such...
  
  ```
  _audio = runner->getAudioService();  
  _audio.lock()->initializeAudio();
  ```
  
 ..that it let's me pull in OpenAL-Soft and control it to my whim...
 
 ```
void CPU::generateBeep()
{  
	//Nobody said I needed to play by the rules.. :D
	alGenBuffers(1, &_buff);
		
	...
	short* samples;
	samples = new short[bufferSize];
	for (int i = 0; i < bufferSize; i++)
	{
		samples[i] = static_cast<short>(32760 * std::sin((2.0f*float(3.14159265359)*frequency)/sampleRate * i));
	}

	alBufferData(_buff, AL_FORMAT_MONO16, samples, bufferSize, static_cast<ALsizei>(sampleRate));
	ALuint source = 0;
	alGenSources(1, &source);
	alSourcei(source, AL_BUFFER, _buff);
 	_source = source;
}

void CPU::beep()
{
	alSourcePlay(_source);
}
 ```
  
  ...and that's it! Did a little more testing based on the opcodes with a test ROM (to make sure the instructions passed their test), and it was complete!
  
## After the journey was over...
  
So, after all of this, what did I take away from it?  
- Always use your resources when it comes to emulation.  
: Anything more than CHIP-8 would require more research and testing, and a completely different approach most likely.  

- Using NovelRT was pretty simple in this context!  
: Although maybe not the *best* choice due to the stigma of using game engines instead of something like SDL natively (overhead, performance, blah blah..) in this context it worked *almost* for everything I needed to do.
  
- There's a couple of PRs/suggestions I could offer to NovelRT.  
: Namely, changing the default background color would have helped avoid maybe... ~4 lines of code, but I can see it being an issue in other projects.  
: Also, seeing if there's any interest in sine-wave generation for the Audio Library would help, seeing as sometimes you just need a simple sound, not a complex WAV file being loaded in. (It helps that I'm the one responsible for this...)  
  
- And finally, it was a great feeling completing a project instead of bouncing back and forth!  
: For someone who seems to not be able to complete things as my mind wanders, I appreciate the fact that I stuck through it (in 9 days since I typically dedicated only 2/3 hours *at most* a day) and actually put a copy out there for people to try if they wanted.  
  
For those of you who stuck through this, thanks for reading! Here's a quick snippet of the emulator in action:  
<iframe width="560" height="315" src="https://www.youtube-nocookie.com/embed/cdlNpU32bA0" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>  
  
  And if you'd like to see the code, you can check it out [here].  
    
Until next time!

-- Cap'n Kenny



[here]: https://github.com/capnkenny/Novel-8
[touhou]: https://github.com/novelrt/touhou-novelrt
