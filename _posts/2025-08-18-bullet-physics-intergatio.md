## Bullet Physics intergation

One of most popular physics engines is Bullet Physics, great engine in my opinion, but it lacks of proper documentation
and Bullet wiki was be shutdown several years ago. So, I want to share my thought on intergation and using of Bullet physics engine.

---

### 1. Beginning

This port was written for Bullet 3.2.5 and can be downloaded from github page: https://github.com/bulletphysics/bullet3/releases/tag/3.25

For first of all we need to create several objects to have physics in the our engine. Bullet use composition over inheritance so 
we should pass several object instances for support dynamics world.

```cpp
// Somewhere in header:

btDefaultCollisionConfiguration* collisionConfiguration;
btCollisionDispatcher* dispatcher;
btBroadphaseInterface* broadphaseInterface;
btConstraintSolver* solver;
btDiscreteDynamicsWorld* dynamicsWorld;
```

Let's look at there's classes:

* btDefaultCollisionConfiguration is collision configuration class, contains Bullet configuration data.
* btCollisionDispatcher is collision detection dispatcher or Narrow Phase.
* btBroadphaseInterface is interface for Broad Phase. We will use btDbvtBroadphase which is broadphase implementation through dynamic AABB bounding volume trees.
* btConstraintSolver is interface for collision solver. We will use btSequentialImpulseConstraintSolver which is implementation of the Projected Gauss Seidel (iterative LCP) method.
* btDynamicsWorld is base class for dynamic and static physics world. We will use btDiscreteDynamicsWorld which is discrete world implementation.

```cpp
// Somewhere in implementation file:

collisionConfiguration = new btDefaultCollisionConfiguration();
dispatcher = new btCollisionDispatcher(collisionConfiguration);
broadphaseInterface = new btDbvtBroadphase();
solver = new btSequentialImpulseConstraintSolver();
dynamicsWorld = new btDiscreteDynamicsWorld(dispatcher, broadphaseInterface, solver, collisionConfiguration);
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

### 2. Introduction to physics world

