# Converting a particle system to use Yeppp.
Sometimes when writing C# code, you come across a problem that could use a little more performance. Particle systems are a great example of this. A particle is generally represented using a structure with simple properties, such as position, colour, age etc, and there can be hundreds of thousands, or even millions, of these structures in memory at the same time, and all of them need to be updated every frame. The types of updates you can do are specific to your games needs, but you'll almost certainly have some sort of movement type update for example. The intuitive way to process all these updates is to simply loop over each particle and change it's position based on some velocity, something like

	while(count-- > 0)
	{
	  	particle->X += particle->VelocityX * elapsedTime;
  		particle->Y += particle->VelocityY * elapsedTime;
  		particle++;
	}

I'm using a pointer to a particle here for speed, and integrating by time to make it frame rate independent. Nothing too complicated or processor intensive but try doing it a million times per frame and it starts to look a whole lot more expensive. By frame I of course mean the portion of your 16ms dedicated to particle effects, so maybe 3-4ms! .Net just isn't up to this. You might say just run it across multiple cores, and that would help but there will be multiple modifiers like this one (attractors, repulsors, colour modifiers etc) already running concurrently, so it is multi-threaded, just at a higher level. 

Enter Yeppp!. From their website  
>Yeppp! is a high-performance SIMD-optimized mathematical library for x86, ARM, and MIPS processors on Windows, Android, Mac OS X, and GNU/Linux systems.

Sounds impressive. They even put an exclamation mark in the name so you'll know it's good! :) I've been dying to try it out and given that I'm working on the particle system for our upcoming game [Honourbound](digitalfurnacegames.com) right now, it seemed the perfect opportunity.

## Mercury particle system
Back when we made our first game, P-3 Biotic, we used an open source particle system called Mercury. For Honourbound, we were planning on a very snazzy, custom GPU based system, but I started looking at the new version of Mercury recently and it was really simple and nicely written (with TDD style tests and everything! So sadly rare in game development it seems), and with a very smart memory management scheme, it was updating 1 million particles in just 10 milliseconds. Two things that jumped out at me though were
* It used an array of structures approach to storing the particles. Bad for cache coherency, which I'll get to below
* It was using the simple loop based system from above to apply modifier updates
By changing to a data-oriented structure of arrays approach  and replacing some of the modifiers with Yeppp based versions, the system can now update 1 million particles in just 1 millisecond!!! Exclamation marks everywhere! Note that this was on a six core 980x. YMMV.

The change to a structure of arrays approach was driven less by a desire to make the system more cache efficient, and more by the fact that Yeppp operates on arrays of values, so having things laid out this way was important, but it's a nice added bonus:) The old particle class looked like this:

    [StructLayout(LayoutKind.Sequential, Pack = 1)]
    public unsafe struct Particle
    {
        public float Inception;
        public float Age;
        public fixed float Position[2];
        public fixed float Velocity[2];
        public fixed float Colour[3];
        public float Opacity;
        public float Scale;
        public float Rotation;
        public float Mass;

        static public readonly int SizeInBytes = Marshal.SizeOf(typeof(Particle));
    }
    
while the new one looks like this:
ADD THE CODE HERE
Why is this more cache efficient? Simply because memory controllers fetch multiple bytes of memory succeeding the one you asked for in a single fetch operation, and put it all in the cache, so if all the Xs are laid out contiguously in memory, the next few Xs will be in cache memory when we need them. If we process all the X values first, then all then Y values, and so on, we'll be making good use of the cache. Now, in .Net land, this presupposes a few conditions:
1. Arrays are actually laid out contiguously in memory. I'm reasonably sure this is the case but I'd have to do more research to be sure.
2. The garbage collector won't fragment array memory during compaction. I have no idea if this is true or not.
3. Memory won't be moved around in the middle of an update. This probably isn't true, although it may be possible to circumvent by using Marshal.AllocHGlobal, which allocates memory from outside the .Net process. In your face Garbage Collector!
Another concern is thread interference. All the threads share the same cache, so when one thread gets some execution time, what does it do to the cache? Do threads store their own copies of the cache and do a flush and replace when they become active? I should find that out.  
With these changes to the particle class, the new movement modifier looks like this:
CODE




Initially array of structures approach.
Really cool efficient use of pointers in .Net
Take one of the modifiers (Move is nice and simple) and show the conversion to Yeppp
How to copy this to device memory and set up streams so that it can be read from.
