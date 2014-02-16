---
layout: post
category : other
tagline: "Particles"
tags : [graphics, particles, directx, opengl, c#]
---
{% include JB/setup %}


# Converting a particle system to use Yeppp.
Sometimes when writing C# code, you come across performance problems that just can't be solved in straight C#. Particle systems are a great example of this. A particle is generally represented using a structure with simple properties, such as position, colour, age etc, and there can be hundreds of thousands, or even millions, of these structures in memory at the same time, and all of them need to be updated every frame. The types of updates you can do are specific to your games needs, but you'll almost certainly have some sort of movement type update for example. The intuitive way to process all these updates is to simply loop over each particle and change it's position based on some velocity, something like

	while(count-- > 0)
	{
	  	particle->X += particle->VelocityX * elapsedTime;
  		particle->Y += particle->VelocityY * elapsedTime;
  		particle++;
	}

I'm using a pointer to a particle here for speed, and integrating by time to make it frame rate independent. Nothing too complicated or processor intensive but try doing it a million times per frame and it starts to look a whole lot more expensive. By frame I of course mean the portion of your 16ms dedicated to particle effects, so maybe 3-4ms! .Net just isn't up to this. You might say just run it across multiple cores, and that would help but there will be multiple modifiers like this one (attractors, repulsors, colour modifiers etc) already running concurrently, so it is multi-threaded, just at a higher level. 

Enter Yeppp!. From their website  
>Yeppp! is a high-performance SIMD-optimized mathematical library for x86, ARM, and MIPS processors on Windows, Android, Mac OS X, and GNU/Linux systems.

Sounds impressive. They even put an exclamation mark in the name so you know it's good! :) I've been dying to try it out and given that I'm working on the particle system for our upcoming game [Honourbound](digitalfurnacegames.com) right now, it seemed the perfect opportunity.

## Mercury particle system
Back when we made our first game, P-3 Biotic, we used an open source particle system called Mercury. For Honourbound, we were planning on a very snazzy, custom GPU based system, and that might still happen, but I started looking at the new version of Mercury recently and it was really simple and nicely written (with TDD style tests and everything! So sadly rare in game development it seems), and with a very smart memory management scheme, it was updating around 1 million particles in just 10 milliseconds. Two things that jumped out at me though were
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

    [StructLayout(LayoutKind.Sequential)]
    public struct Particle
    {
		public float[] X;
		public float[] Y;
		public float[] VX;
		public float[] VY;
		public float[] Inception;
		public float[] Age;
		public float[] R;
		public float[] G;
		public float[] B;
		public float[] Opacity;
		public float[] Scale;
		public float[] Rotation;
		public float[] Mass;

		static public readonly int SizeInBytes = 36;
	}
Why is this more cache efficient? Simply because memory controllers fetch multiple bytes of memory succeeding the one you asked for in a single fetch operation, and put it all in the cache, so if all the Xs are laid out contiguously in memory, the next few Xs will be in cache memory when we need them. If we process all the X values first, then all then Y values, and so on, we'll be making good use of the cache. Now, in .Net land, this presupposes a few conditions:
1. Arrays are actually laid out contiguously in memory. I'm reasonably sure this is the case but I'd have to do more research to be sure.
2. The garbage collector won't fragment array memory during compaction. I have no idea if this is true or not.
3. Memory won't be moved around in the middle of an update. This probably isn't true, although it may be possible to circumvent by using Marshal.AllocHGlobal, which allocates memory from outside the .Net process. In your face Garbage Collector!
Another concern is thread interference. All the threads share the same cache, so when one thread gets some execution time, what does it do to the cache? Do threads store their own copies of the cache and do a flush and replace when they become active? I should find that out.  

With these changes to the particle class, the new movement modifier looks like this:  

    var velXIntegral = new float[YepppConstants.BufferLength];
    var velYIntegral = new float[YepppConstants.BufferLength];

	var i = 0;
	while (i < count)
	{
		var blockLength = Math.Min(YepppConstants.BufferLength, count - i);
		Yeppp.Core.Multiply_V32fS32f_V32f(particle.VX, i, elapsedSeconds, velXIntegral, 0, blockLength);
	    Yeppp.Core.Multiply_V32fS32f_V32f(particle.VY, i, elapsedSeconds, velYIntegral, 0, blockLength);

		Yeppp.Core.Add_IV32fV32f_IV32f(particle.X, i, velXIntegral, 0, blockLength);
		Yeppp.Core.Add_IV32fV32f_IV32f(particle.Y, i, velYIntegral, 0, blockLength);
			
		i += YepppConstants.BufferLength;
	}
This code processes blocks of particles, YepppConstants.BufferLength particles at a time. This sample has YepppConstants.BufferLength set to 8192, but that was based on trial and error profiling. I expect that the optimal block size will be different on different hardware. Yeppp supports a fairly basic set of primitive operations right now (add, subtract, multiply, dot product, sum, sum of squares, min and max) and some more complex operation (exp, log etc), but this is largely all that's needed for a particle system. There are a few things missing that would make a big difference (sum of squares between two different arrays, sqrt - this would make a big difference to performance if it used the SSE reciprocal square root instructions!) so hopefully they'll be added soon.

## Rendering
The difficulty with keeping particles in this format is getting them onto the graphics card in an efficient way. What's good for cache coherency on the CPU is the exact polar opposite of what's good on the GPU. Fun stuff. I've basically ignored that for now though:) Having said that, this doesn't seem to have effected rendering efficiency in any real way. I'm using memcpy to transfer each component into a locked vertex buffer contiguously. I'm then using one stream for each component, with an offset between each stream equal to the size in bytes of each components' array. The vertex declaration looks like this:  

    var vertexElements = new[]
    {
    	new VertexElement(0, 0,     DeclarationType.Float1, DeclarationMethod.Default, DeclarationUsage.Color, 0),					// Age
		new VertexElement(1, 0,     DeclarationType.Float1, DeclarationMethod.Default, DeclarationUsage.Position, 0),				// X
		new VertexElement(2, 0,		DeclarationType.Float1, DeclarationMethod.Default, DeclarationUsage.Position, 1),				// Y
		new VertexElement(3, 0,		DeclarationType.Float1, DeclarationMethod.Default, DeclarationUsage.Color, 1),					// R
		new VertexElement(4, 0,		DeclarationType.Float1, DeclarationMethod.Default, DeclarationUsage.Color, 2),					// R
		new VertexElement(5, 0,		DeclarationType.Float1, DeclarationMethod.Default, DeclarationUsage.Color, 3),					// R
		new VertexElement(6, 0,		DeclarationType.Float1, DeclarationMethod.Default, DeclarationUsage.PointSize, 0),				// Size
		new VertexElement(7, 0,		DeclarationType.Float1, DeclarationMethod.Default, DeclarationUsage.TextureCoordinate, 0),		// Rotation
        VertexElement.VertexDeclarationEnd
    };  
The test app was written against DirectX9 but the engine we use for Honourbound works on OpenGL, so we'll need to find a similar way to do the same there. Should be doable though.
