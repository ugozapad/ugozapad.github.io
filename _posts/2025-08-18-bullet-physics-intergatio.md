## Bullet Physics intergation

One of most popular physics engines is Bullet Physics, great engine in my opinion, but it lacks of proper documentation
and Bullet wiki was be shutdown several years ago. So, I want to share my thought on intergation and using of Bullet physics engine.

---

### 1. Beginning

For first of all we need to create several objects to have physics in the our engine. Bullet use composition over inheritance so 
we should pass several object instances for support dynamics world.

```cpp

// Somewhere in header:

btDefaultCollisionConfiguration* collisionConfiguration;
btCollisionDispatcher* dispatcher;
btBroadphaseInterface* overlappingPairCache;
btSequentialImpulseConstraintSolver* solver;
btDiscreteDynamicsWorld* dynamicsWorld;

// Somewhere in implementation file:

collisionConfiguration = new btDefaultCollisionConfiguration();
dispatcher = new btCollisionDispatcher(collisionConfiguration);
overlappingPairCache = new btDbvtBroadphase();
solver = new btSequentialImpulseConstraintSolver();
dynamicsWorld = new btDiscreteDynamicsWorld(dispatcher, overlappingPairCache, solver, collisionConfiguration);

```

also you can override default Bullet allocator(which is malloc), but you should do this before any Bullet object creation:


```cpp

// For non-aligned allocator:

void* BulletAlloc(size_t size)
{
	// malloc is example
	return malloc(size);
}

void BulletFree(void* ptr)
{
	// free is example
	free(ptr);
}

btAlignedAllocSetCustom(BulletAlloc, BulletFree);

// For aligned allocator:

void* BulletAlignAlloc(size_t size, int alignment)
{
	// _aligned_malloc is example
	return _aligned_malloc(size, alignment);
}

void BulletAlignFree(void* ptr)
{
	// _aligned_free is example
	_aligned_free(ptr);
}

btAlignedAllocSetCustomAligned(BulletAlignAlloc, BulletAlignFree);

```