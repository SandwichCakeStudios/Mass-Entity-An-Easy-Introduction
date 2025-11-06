# Mass Entity System for Unreal Engine 5

> A beginner-friendly guide to Unreal Engine's high-performance Entity Component System

For more details we recommend [MassSample GitHub](https://github.com/Megafunk/MassSample) - Community sample project with documentation

Code Examples will be added.

## Table of Contents

- [What is Mass?](#what-is-mass)
- [Why Use Mass?](#why-use-mass)
- [Core Concepts](#core-concepts)
- [Architecture Overview](#architecture-overview)
- [Getting Started](#getting-started)
- [When NOT to Use Mass](#when-not-to-use-mass)
- [Additional Resources](#additional-resources)

---
<a name="what-is-mass"></a>
## What is Mass?

Mass is Unreal Engine 5's **data-oriented framework** for handling massive numbers of entities with high performance. Think of it as a completely different way of building game systems compared to traditional Actor-based programming.

### The Traditional Way vs. Mass

**Traditional Unreal (Actor-Based):**
- Each character is an Actor with its own logic and data
- Great for flexibility, but slow when you have thousands of entities
- Data is scattered in memory, causing CPU cache misses

**Mass (Data-Oriented):**
- Separates data from behavior
- Stores similar data together in memory for lightning-fast processing
- Can handle thousands of entities efficiently
- Perfect for crowds, flocks, particles, and more

**Mix (Using both):**
- By adding a UMassAgentComponent to an actor, a mass entity will be created for it based on the fragments/tags or traits you set.
- A bridge between the general UE5 actor framework and Mass. A type of fragment that turns entities into "Agents" that can exchange data in either direction (or both ways).
- Example: The player can be an actor that has it´s location synced to it´s corresponding mass entity. Other mass entities can use that data for it´s logic.

### Key Philosophy

Mass follows the **Entity Component System (ECS)** architecture pattern:
- **Entities** = Lightweight IDs that point to data
- **Fragments** (Components) = The actual data (transform, velocity, health, etc.)
- **Processors** (Systems) = The logic that operates on the data

---
<a name="why-use-mass"></a>
## Why Use Mass?

### **Performance**
- **Cache-friendly memory layout**: Similar data stored contiguously in arrays
- **Multi-threading**: Process different chunks of entities in parallel
- **Minimal cache waste**: CPU can load and process data efficiently

### **Scalability**
- Handle **thousands of entities** simultaneously
- Add/remove components without inheritance complexity
- No deep class hierarchies to manage

### **Optimization**
- Data grouped into **Archetypes** and **Chunks**
- Reduced code branching
- Batch processing for efficiency

### **Composability**
- Build behavior by adding/removing components
- Mix and match fragments to create different entity types
- No need for complex inheritance trees

---

<a name="when-not-to-use-mass"></a>
## When NOT to Use Mass

While Mass is powerful, it's not always the right choice:

### **Indirect Behavior**
- Logic lives in processors, not on entities
- Harder to trace behavior for specific entities
- Different mental model than traditional OOP

### **Costly Structural Changes**
- Adding/removing fragments frequently can be expensive
- Causes entities to migrate between archetypes
- Mitigated by deferred command buffers, but still has overhead

### **Overhead for Small Scales**
- Data systems have setup and processing overhead
- Not efficient for small numbers of entities (< 50)
- Traditional Actors might be simpler and faster

### **Over-Granular Fragments**
- Too many tiny fragments can hurt performance
- Creates excessive chunks and ruins cache locality
- Solution: Group fields that are processed together

### **Complex Data Management**
- Requires careful control over read/write access
- Misusing Shared or Const-Shared data can bloat processing
- Steeper learning curve than traditional programming

---

<a name="core-concepts"></a>
## Core Concepts

### 1. **Entity**

An Entity is just a **unique ID** that points to a collection of Fragments. It's like a container that holds different pieces of data.

```
Entity #42 = [Transform Fragment, Velocity Fragment, Health Fragment]
```

Entities have **no logic** themselves—they're purely data containers.

### 2. **Fragment**

Fragments are **pieces of data** stored in tightly-packed arrays. They represent the state of an Entity.

**Common Examples:**
- Transform (position, rotation, scale)
- Velocity (movement speed and direction)
- Current LOD (level of detail)
- Health points
- AI state

**Special Types of Fragments:**

- **Shared Fragment**: Data shared across multiple entities to save memory (e.g., all enemies of the same type share the same stats)
- **Chunk Fragment**: Applied to a group of entities rather than individual ones
- **Tag**: A fragment with no data—used purely for filtering (like a label)

### 3. **Archetype**

An Archetype is a **unique combination of Fragments**. All entities with the exact same fragment composition belong to the same archetype.

**Example:**
```
Archetype A = [Transform, Velocity, Health]
Archetype B = [Transform, Velocity, Health, AIState]
```

If you add or remove a fragment from an entity, it **moves to a different archetype**.

### 4. **Chunks**

Chunks are **fixed-size blocks** of tightly-packed entity data within an archetype. They ensure that data for similar entities is stored contiguously in memory for optimal cache performance.

```
Archetype [Transform, Velocity]
  └─ Chunk 1: [Entity 1-100]
  └─ Chunk 2: [Entity 101-200]
```

### 5. **Processor**

Processors contain the **logic and behavior** that operates on fragments. They're stateless and run on batches of entities that match specific criteria.

**Key Features:**
- Define queries to select which entities to process
- Declare read/write requirements for fragments
- Execute in specific processing phases
- Can run in parallel for performance

**Example:** A movement processor might query for entities with Transform and Velocity fragments, then update their positions each frame.

### 6. **Entity Query**

Queries are how processors **filter and select** which entities to operate on based on fragment requirements.

```cpp
Query Requirements:
  - Must have: Transform, Velocity
  - Must NOT have: Frozen tag
  - Optional: DebugInfo
```

### 7. **Trait**

Traits are **templates** that bundle fragments and tags together. Think of them as blueprints or prefabs for entity configurations.

**Example:**
```
Movement Trait = [Transform Fragment + Velocity Fragment + Moving Tag]
```

Traits make it easy to create entities with predefined behaviors.

---

<a name="architecture-overview"></a>
## Architecture Overview

### Mass Terminology (ECS Translation)

Mass uses different names to avoid confusion with existing Unreal code:

| Standard ECS Term | Mass Term |
|-------------------|-----------|
| Entity | Entity |
| Component | **Fragment** |
| System | **Processor** |

### Processing Flow

Here's how Mass processes entities each frame:

```
1. Processor Configuration
   └─ Define EntityQuery with fragment requirements
   
2. Query Execution
   └─ Match requirements with Archetypes
   └─ Filter chunks based on criteria
   
3. Chunk Processing
   └─ Process entities chunk-by-chunk
   └─ Access individual entities via FMassExecutionContext
   └─ Execute logic (usually with lambda functions)
   
4. Deferred Changes
   └─ Queue composition changes (add/remove fragments)
   └─ Apply changes at sync points between phases
```

### Processing Phases

Processors execute in specific phases aligned with Unreal's tick groups:

1. **PrePhysics** (default)
2. **StartPhysics**
3. **DuringPhysics**
4. **EndPhysics**
5. **PostPhysics**
6. **FrameEnd**

Processors within a phase can run in **parallel** for maximum performance.

---

<a name="getting-started"></a>
## Getting Started

### Basic Workflow

1. **Create Fragments**: Define your data structures (Transform, Velocity, etc.)
2. **Create Traits**: Bundle fragments together for reusable entity templates
3. **Create Processors**: Write logic to operate on entities with specific fragments
4. **Create Mass Entity Config**: Define entity templates using traits
5. **Spawn Entities**: Use Mass Spawner to add entities to your level

### Key Classes and Systems

- **FMassEntityManager**: Owns all entities and archetypes, handles composition changes
- **UMassEntitySubsystem**: World subsystem that holds the Entity Manager
- **UMassProcessor**: Base class for all processors
- **FMassEntityQuery**: Query definition for filtering entities
- **FMassCommandBuffer**: Deferred command queue for composition changes
- **UMassSpawnerSubsystem**: System for spawning entities at runtime
```c++
bool UTestSpawnerSubsystem::SpawnEntity_Internal(UMassEntityConfigAsset* MassEntityConfig,const FTransform& SpawnTransform)
{
	UWorld* World = GetWorld();
	check(World);

  if (UMassSpawnerSubsystem* SpawnerSubsystem = World->GetSubsystem<UMassSpawnerSubsystem>())
	{
		
      if (MassEntityConfig)
      {
          const FMassEntityTemplate& EntityTemplate = MassEntityConfig->GetOrCreateEntityTemplate(*World);
          if (EntityTemplate.IsValid())
          {
              FInstancedStruct SpawnData; 
              SpawnData.InitializeAs<FMassTransformsSpawnData>();
        
              FMassTransformsSpawnData& Transforms = SpawnData.GetMutable<FMassTransformsSpawnData>();
        
              Transforms.Transforms.Add(SpawnTransform);
        
              TArray<FMassEntityHandle> Entities;
              SpawnerSubsystem->SpawnEntities(EntityTemplate.GetTemplateID(), 1, SpawnData, UMassSpawnLocationProcessor::StaticClass(), Entities);
              return true;
          }
      }
	}
	return false;
}
```
- **MassEntityConfig**: Asset defining entity templates with traits

### Commands and Signals

**Deferred Commands:**
- Queue operations to add/remove fragments or create/destroy entities
- Executed in batches at sync points to avoid data corruption
- Accessed via `FMassCommandBuffer`
```c++

void UDamageProcessor::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	EntityQuery.ForEachEntityChunk(Context, [&EntityManager](FMassExecutionContext& Context)
		{
			
			const TArrayView<FHealthFragment> HealthList = Context.GetMutableFragmentView<FHealthFragment>();
		
			for (FMassExecutionContext::FEntityIterator EntityIt = Context.CreateEntityIterator(); EntityIt; ++EntityIt)
			{

				FHealthFragment& HealthFragment = HealthList[EntityIt];

				const FMassEntityHandle Entity = Context.GetEntity(EntityIt);

				if (HealthFragment.Health > HealthFragment.DamageAmount)
				{
					HealthFragment.Health -= HealthFragment.DamageAmount;
				}
				else
				{
					HealthFragment.Health = 0;
					Context.Defer().AddTag<FDeadTag>(Entity);
				}

				HealthFragment.DamageAmount = 0;

				
			}
		});
}
```

**Observers:**
- React to fragment/tag changes or entity initialization
- Only run when specific changes occur (not every frame)
- Use `UMassObserverProcessor` for this pattern

**Signals:**
- Send named signals between entities
- Lightweight event system for entity communication
- Part of the MassSignals module

---

<a name="additional-resources"></a>
## Additional Resources

### Official Documentation
- [Mass Entity Overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-mass-entity-in-unreal-engine) - Official Epic Games documentation
- [Mass Gameplay Overview](https://dev.epicgames.com/documentation/en-us/unreal-engine/overview-of-mass-gameplay-in-unreal-engine) - Gameplay-focused Mass features

### Community Resources
- [MassSample GitHub](https://github.com/Megafunk/MassSample) - Community sample project with documentation
- [Your First 60 Minutes with Mass](https://dev.epicgames.com/community/learning/tutorials/JXMl/unreal-engine-your-first-60-minutes-with-mass) - Beginner tutorial
- [State of Unreal: Large Numbers of Entities with Mass](https://www.youtube.com/watch?v=f9q8A-9DvPo) - Video presentation by Epic

### Related Systems
- **ZoneGraph**: Spline-based navigation for Mass crowds (sidewalks, roads, etc.). Much cheaper than NavMesh but very limited.
- **StateTree**: Lightweight state machine for Mass AI. 
- **Smart Objects**: Interactive objects that Mass entities can use
- **Mass Avoidance**: Force-based avoidance system
- **Mass Replication**: Network replication for Mass entities (experimental)

---

## Pro Tips

1. **Start Simple**: Begin with a few fragments and one processor before scaling up
2. **Think in Data**: Focus on what data you need, not what objects do
3. **Batch Operations**: Use command buffers to queue changes rather than modifying entities immediately
4. **Profile Early**: Use Unreal's profiling tools to measure performance gains
5. **Group Related Data**: Keep fragments that are always processed together in the same chunk
6. **Use Tags Wisely**: Tags are free and excellent for filtering without data overhead
7. **Learn from Examples**: Study UE5's City Sample to see Mass in action at scale

---

## License

Mass is part of Unreal Engine 5 and follows Epic Games' licensing terms. Check the [Unreal Engine EULA](https://www.unrealengine.com/eula) for details.

---

**Note**: Mass is marked as **Experimental** in Unreal Engine 5. While it's powerful and used in production (like the City Sample), APIs may change and Epic doesn't recommend shipping with experimental features without thorough testing.
