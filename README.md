<p align="center"> 
  <img src="https://i.imgur.com/7XvtEWf.png" alt="alt logo">
</p>

[![PayPal](https://drive.google.com/uc?id=1OQrtNBVJehNVxgPf6T6yX1wIysz1ElLR)](https://www.paypal.me/nxrighthere) [![Bountysource](https://drive.google.com/uc?id=19QRobscL8Ir2RL489IbVjcw3fULfWS_Q)](https://salt.bountysource.com/checkout/amount?team=nxrighthere) [![Discord](https://discordapp.com/api/guilds/515987760281288707/embed.png)](https://discord.gg/ceaWXVw)

This is an improved version of [smmalloc](https://github.com/SergeyMakeev/smmalloc) a [fast and efficient](https://github.com/SergeyMakeev/smmalloc#features) memory allocator designed to handle many small allocations/deallocations in heavy multi-threaded scenarios. The allocator created for using in applications where the performance is critical such as video games.

Using smmalloc allocator in the .NET environment helps to minimize GC pressure for allocating buffers and avoid using lock-based pools in multi-threaded systems. Modern .NET features such as [`Span<T>`](https://msdn.microsoft.com/en-us/magazine/mt814808.aspx) greatly works in tandem with smmalloc and allows conveniently manage data in native memory blocks.

Usage
--------
##### Create a new smmalloc instance:
```c#
// 8 buckets, 16 MB each, 128 bytes maximum allocation size
SmmallocInstance smmalloc = new SmmallocInstance(8, 16 * 1024 * 1024);
```

##### Destroy the smmalloc instance and free allocated memory:
```c#
smmalloc.Dispose();
```

##### Create thread cache for a current thread:
```c#
// 4 KB of thread cache for each bucket, hot warmup
smmalloc.CreateThreadCache(4 * 1024, CacheWarmupOptions.Hot);
```

##### Destroy thread cache for a current thread:
```c#
smmalloc.DestroyThreadCache();
```

##### Allocate memory block:
```c#
// 64 bytes of a memory block
IntPtr pointer = smmalloc.Malloc(64);
```

##### Release memory block:
```c#
smmalloc.Free(pointer);
```

##### Write data to memory block:
```c#
// Using Marshal
byte data = 0;

for (int i = 0; i < smmalloc.Size(pointer); i++) {
	Marshal.WriteByte(pointer, i, data++);
}

// Using Span
Span<byte> buffer;

unsafe {
	buffer = new Span<byte>((byte*)pointer, smmalloc.Size(pointer));
}

byte data = 0;

for (int i = 0; i < buffer.Length; i++) {
	buffer[i] = data++;
}
```

##### Read data from memory block:
```c#
// Using Marshal
int sum = 0;

for (int i = 0; i < smmalloc.Size(pointer); i++) {
	sum += Marshal.ReadByte(pointer, i);
}

// Using Span
int sum = 0;

foreach (var value in buffer) {
	sum += value;
}
```

##### Hardware accelerated operations:
```c#
// Xor using Vector and Span
if (Vector.IsHardwareAccelerated) {
	Span<Vector<byte>> bufferVector = MemoryMarshal.Cast<byte, Vector<byte>>(buffer);
	Span<Vector<byte>> xorVector = MemoryMarshal.Cast<byte, Vector<byte>>(xor);

	for (int i = 0; i < bufferVector.Length; i++) {
		bufferVector[i] ^= xorVector[i];
	}
}
```

##### Copy data using memory block:
```c#
// Using Marshal
byte[] data = new byte[64];

// Copy from native memory
Marshal.Copy(pointer, data, 0, 64);

// Copy to native memory
Marshal.Copy(data, 0, pointer, 64);

// Using Buffer
unsafe {
	// Copy from native memory
	fixed (byte* destination = &data[0]) {
		Buffer.MemoryCopy((byte*)pointer, destination, 64, 64);
	}

	// Copy to native memory
	fixed (byte* source = &data[0]) {
		Buffer.MemoryCopy(source, (byte*)pointer, 64, 64);
	}
}
```

##### Custom data structures:
```c#
struct Entity {
	public uint id;
	public byte health;
	public byte state;
}

int entitySize = Marshal.SizeOf(typeof(Entity));
int entityCount = 10;

IntPtr pointer = smmalloc.Malloc(entitySize * entityCount);

Span<Entity> entities;

unsafe {
	entities = new Span<Entity>((byte*)pointer, entityCount);
}

uint id = 1;

for (int i = 0; i < entities.Length; i++) {
	entities[i].id = id++;
	entities[i].health = (byte)(new Random().Next(1, 100));
	entities[i].state = (byte)(new Random().Next(1, 255));
}

smmalloc.Free(pointer);
```

API reference
--------
### Enumerations
#### CacheWarmupOptions
Definitions of warmup options for `CreateThreadCache()` function:

`CacheWarmupOptions.Cold` warmup not performed for cache elements.

`CacheWarmupOptions.Warm` warmup performed for half of the cache elements.

`CacheWarmupOptions.Hot` warmup performed for all cache elements.

### Classes
A single low-level disposable class is used to work with smmalloc. 

#### SmmallocInstance

Contains a managed pointer to the smmalloc instance.

##### Constructors
`SmmallocInstance(uint bucketsCount, int bucketSize)` creates allocator instance with a memory pool. Size of memory blocks in each bucket increases with a count of buckets. The bucket size parameter sets an initial size of a pooled memory in bytes.

##### Methods
`SmmallocInstance.Dispose()` destroys the smmalloc instance and frees allocated memory.

`SmmallocInstance.CreateThreadCache(int cacheSize, CacheWarmupOptions warmupOption)` creates thread cache for fast memory allocations within a thread. The warmup option sets pre-allocation degree of cache elements.

`SmmallocInstance.DestroyThreadCache()` destroys the thread cache.

`SmmallocInstance.Malloc(int bytesCount, int alignment)` allocates aligned memory block. Allocation size depends on buckets count multiplied by 16, so the minimum allocation size is 16 bytes. Maximum allocation size using two buckets in a smmalloc instance will be 32 bytes, for three buckets 48 bytes, for four 64 bytes, and so on. The alignment parameter is optional. Returns pointer to a memory block.

`SmmallocInstance.Free(IntPtr memory)` frees memory block.

`SmmallocInstance.Realloc(IntPtr memory, int bytesCount, int alignment)` reallocates memory block. The alignment parameter is optional. Returns pointer to a reallocated memory block.

`SmmallocInstance.Size(IntPtr memory)` gets usable memory size. Returns size in bytes.

`SmmallocInstance.Bucket(IntPtr memory)` gets bucket index of a memory block. Returns placement index.
