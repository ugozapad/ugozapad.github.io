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
// For non-aligned allocator, malloc and free is example:

void* BulletAlloc(size_t size)
{
	return malloc(size);
}

void BulletFree(void* ptr)
{
	free(ptr);
}

btAlignedAllocSetCustom(BulletAlloc, BulletFree);

// For aligned allocator, _aligned_malloc and _aligned_free is example:

void* BulletAlignAlloc(size_t size, int alignment)
{
	return _aligned_malloc(size, alignment);
}

void BulletAlignFree(void* ptr)
{
	_aligned_free(ptr);
}

btAlignedAllocSetCustomAligned(BulletAlignAlloc, BulletAlignFree);
```

### 2. Introduction to physics world

After creation the physics world we need to take a look on objects, which will be live in the world.

Some of types:
* Collision Object - non-movable object. For example static level collision.
* Rigid Body - rigid body or movable object, have mass, velocity and gravity will affect on them. For example: moving boxes
* Ghost Objects - objects which allow to pass through self. For example: Triggers
* Action - interface for object custom behaviour. Helpful for character controller and something non-standart which you can imagine.

In my engine I used *Collision Object* for static level collision, *Rigid Body* for entities who supposed to have physics interaction and *Ghost Objects* for triggers.

### N. Ghost Body

Ghost body is useful class to create physics world object which is supposed to be a "filter" for others movable objects. 

This is snippet for creation and adding the ghost body for trigger (please don't be scare of hungarian notation :P):
```cpp
m_shape = new btBoxShape(halfBoxExtends);

m_ghostObject = new btPairCachingGhostObject();
m_ghostObject->setCollisionShape(shape);

// Disable contant response
m_ghostObject->setCollisionFlags(m_ghostObject->getCollisionFlags() | btCollisionObject::CF_NO_CONTACT_RESPONSE);

// user pointer is pointer to the object owner entity
m_ghostObject->setUserPointer(this);

// Add to world with character filter only. Last filter can be anything, not only related to btBroadphaseProxy::CollisionFilterGroups
g_physicsSystem.GetWorld()->addCollisionObject(m_ghostObject, btBroadphaseProxy::SensorTrigger, btBroadphaseProxy::CharacterFilter);
```

Trigger update function which is update gathered object collisions:

```cpp
void Trigger::Update(float dt)
{
	Entity* entity = nullptr;

	int numOverlappingObjects = m_ghostObject->getNumOverlappingObjects();
	for (int i = 0; i < numOverlappingObjects; i++)
	{
		btCollisionObject* collisionObject = m_ghostObject->getOverlappingObject(i);

		// Use upcast function rathen dynamic_cast
		btRigidBody* rigidBody = btRigidBody::upcast(collisionObject);
		if (rigidBody)
			entity = (Entity*)rigidBody->getUserPointer();

		btGhostObject* ghostObject = btGhostObject::upcast(collisionObject);
		if (ghostObject)
			entity = (Entity*)rigidBody->getUserPointer();
			
		if (entity && entity->IsPlayer())
		{
			// do something ...
		}
	}
}
```

### N. Bullet debugging.

Bullet doesn't have separate application like PhysX or Havok, so debugging is little harder. But helpfuly for us we have btIDebugDraw interface.
Interface have methods for drawing the lines, contancts and print useful information to the output. 

This is my implementation of debug drawer:
```cpp
class PhysicsDebugDraw : public btIDebugDraw
{
public:
	PhysicsDebugDraw();
	~PhysicsDebugDraw();

	// btIDebugDraw inherit
	void drawLine(const btVector3& from, const btVector3& to, const btVector3& color) override;
	void drawContactPoint(const btVector3& PointOnB, const btVector3& normalOnB, btScalar distance, int lifeTime, const btVector3& color) override;
	void reportErrorWarning(const char* warningString) override;
	void draw3dText(const btVector3& location, const char* textString) override;
	void setDebugMode(int debugMode) override;
	int getDebugMode() const override;

private:
	int m_debugMode;
};

PhysicsDebugDraw() :
	m_debugMode(0)
{
}

void PhysicsDebugDraw::drawLine(const btVector3& from, const btVector3& to, const btVector3& color)
{
	DebugRender::DrawLine(
		glm::vec3(from.x(), from.y(), from.z()),
		glm::vec3(to.x(), to.y(), to.z()),
		glm::vec3(color.x(), color.y(), color.z())
	);
}

void PhysicsDebugDraw::drawContactPoint(const btVector3& PointOnB, const btVector3& normalOnB, btScalar distance, int lifeTime, const btVector3& color)
{
	drawLine(
		PointOnB,
		PointOnB + normalOnB,
		color
	);
}

void PhysicsDebugDraw::reportErrorWarning(const char* warningString)
{
	Logger::Print("%s", warningString);
}

void PhysicsDebugDraw::draw3dText(const btVector3& location, const char* textString)
{
	Logger::Print("%s: %s", "CPhysicsDebugDraw::draw3dText", textString);
}

void PhysicsDebugDraw::setDebugMode(int debugMode)
{
	m_debugMode = debugMode;
}

int PhysicsDebugDraw::getDebugMode() const
{
	return m_debugMode;
}
```

Before using we should setup instance of our debug drawer to the world:

```cpp
static PhysicsDebugDraw debugDrawer;
dynamicsWorld->setDebugDrawer(&debugDrawer);
```

After we should select drawing mode to see the Bullet world:

```cpp
int debugDrawMode = 0;
debugDrawMode |= btIDebugDraw::DBG_DrawAabb; // Draw the axis-aligned bounding box of objects
debugDrawMode |= btIDebugDraw::DBG_DrawWireframe; // Draw wireframe shapes of objects
physicsDebugDraw.setDebugMode(debugDrawMode);

dynamicsWorld->debugDrawWorld();
```

Don't forget to flush lines to the renderer!