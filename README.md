# Mass Entity System for Unreal Engine 5

> A beginner-friendly guide to Unreal Engine's high-performance Entity Component System



> [!IMPORTANT]
> Information have been collected from multiple sources and AI has been used to analyze it and create a well-structured presentation.
It may be error proned but page is updated manually from time to time to fix these.


> [!TIP]
> For a more deep dive into mass we recommend [MassSample GitHub](https://github.com/Megafunk/MassSample) - Community sample project with documentation

## Table of Contents

- [What is Mass?](#what-is-mass)
- [Why Use Mass?](#why-use-mass)
- [When NOT to Use Mass](#when-not-to-use-mass)
- [Core Concepts](#core-concepts)
- [Architecture Overview](#architecture-overview)
- [Getting Started](#getting-started)
- [Additional Resources](#additional-resources)

---
<a name="what-is-mass"></a>
## What is Mass?

Mass is Unreal Engine 5's **data-oriented framework** for handling massive numbers of entities with high performance. Think of it as a completely different way of building game systems compared to traditional Actor-based programming.

### The Traditional Way vs. Mass

**Traditional Unreal (Actor-Based):**
- Each entity is an Actor with its own logic and data
- Great for flexibility, but slow when you have thousands of entities
- Data is scattered in memory, causing CPU cache misses

**Mass (Data-Oriented):**
- Separates data from behavior
- Stores similar data together in memory for lightning-fast processing
- Can handle thousands of entities efficiently
- Perfect for crowds, flocks, particles, and more

**Mix (Using both):**
- By adding a UMassAgentComponent to an actor, a mass entity will be created for it based on the fragments/tags or traits you set. This acts as a bridge between the general UE5 actor framework and Mass.
- Special proccesors called UMassTranslators can then exchange data in either direction (or both ways).
- Example: The player can be an actor that has it´s location synced to it´s corresponding mass entity. Other mass entities (like zombies) can use that data for it´s logic.
- [Read more about translators](#Translator)
---

<a name="why-use-mass"></a>
## Why Use Mass?
Mass is built for games or simulations that need to handle lots of entities efficiently. Here's why it shines:

### **1. It’s Super Fast**
Mass is designed with performance in mind. Here’s how it helps your game run smoother:

- It stores similar data together in memory, so the CPU can grab it quickly.

- It can process many entities at once using multiple threads.

- It avoids wasting memory or CPU time on unused data.

If you want your game to handle hundreds or thousands of things happening at once — like bullets, enemies, or crowds — Mass is a great fit.

### **2. It’s Easy to Mix and Match**
Mass uses fragments to define what an entity can do. That means:

- You can build different types of entities by combining different fragments.

- Want a flying enemy? Add a “Flight” fragment.

- Want a boss enemy? Add “Health,” “AI,” and “ExplosionOnDeath” fragments.

No need to create a new class for every variation — just plug in the pieces you need.

### **3. It’s Built for Optimization**
Mass organizes data into Archetypes and Chunks — fancy terms for grouping similar entities together. This helps:

- Reduce unnecessary code branching (no more “if this is a player…” checks everywhere).

- Process entities in batches, which is much faster than one-by-one.


### **4. It Encourages Clean, Modular Design**
Mass pushes you to think in terms of data and behavior, not inheritance. That means:

- Your code is easier to maintain and extend.

- You can mix and match fragments to create different entity types.

- You can avoid the spaghetti code that often comes with deep class trees.


---

<a name="when-not-to-use-mass"></a>
## When NOT to Use Mass

Mass is powerful, but it’s not always the right tool. Here’s when you might want to avoid it:

### **1. If You Want Behavior Attached to Objects**
In Mass, logic doesn’t live inside the entities. Instead, it lives in separate systems called processors. This means:

- You can’t just open an entity and see what it does.

- You have to look at the processors to understand behavior.

- It’s a different way of thinking than traditional object-oriented programming (OOP).

If you prefer the classic “this object does this” model, Mass might feel confusing.

### **2. If You Constantly Add or Remove Components**
Changing an entity’s structure (like adding or removing fragments) is expensive. Why?

- Every time you do it, the entity might need to move to a different memory layout (called an archetype).

- That takes time and resources.

- Mass can handle this in certain ways, but if your game logic changes entities all the time, performance might suffer.

### **3. If You Only Have a Few Entities**
Mass is designed for lots of entities — think hundreds or thousands. If you’re only dealing with a handful (like 10–50), it’s probably overkill. In those cases, using traditional Actors or Components is simpler and easier to debug.

### **4. If You Break Things into Too Many Tiny Pieces**
Mass encourages breaking data into fragments — but don’t go overboard. If you split everything into tiny fragments:

- You’ll might end up with too many memory chunks.

- That may ruin performance and makes things harder to manage.

Instead, group related data together when it makes sense.

### **5. If You’re Not Ready to Manage Data Carefully**
Mass gives you a lot of control — but that means more responsibility. You need to:

- Know when data is read-only, shared, or writable.

If you’re not comfortable managing memory access patterns, Mass can be tricky to use correctly.

---



<a name="core-concepts"></a>
## Core Concepts

### Mass Terminology (ECS Translation)

Mass uses different names to avoid confusion with existing Unreal code:

| Standard ECS Term | Mass Term | Description |
|-------------------|-----------|-----------|
| Entity | **Entity** | Lightweight IDs that point to data |
| Component | **Fragment** | The actual data (transform, velocity, health, etc.) |
| System | **Processor** | The logic that operates on the data |


### 1. **Entity**

An Entity is just a **unique ID** that points to a collection of Fragments. It's like a container that holds different pieces of data.

```
Entity #42 = [Transform Fragment, Velocity Fragment, Health Fragment]
```
```c++
const FMassEntityHandle Entity = Context.GetEntity(EntityIt);
if (!EntityManager.IsEntityValid(Entity))
{
	// Entity is invalid
}
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

```c++
USTRUCT()
struct FHealthFragment : public FMassFragment
{
	GENERATED_BODY()

	uin16 Health = 100;

};
```

**Special Types of Fragments:**

- **Shared Fragment**: Data shared across multiple entities to save memory (e.g., all enemies of the same type share the same stats)
```c++
USTRUCT()
struct FMassStateTreeSharedFragment : public FMassConstSharedFragment
{
	GENERATED_BODY()

	FMassStateTreeSharedFragment() = default;

	UPROPERTY()
	TObjectPtr<UStateTree> StateTree = nullptr;
};
```
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

<a name="Translator"></a>
### 8. **Translator**



Translators are special processors that moves data between UE objects and Mass fragments.

By adding a UMassAgentComponent: to an Actor, it automatically creates a Mass entity for that Actor based on the traits you assign on the component.

A special UMassAgentSyncTrait can be used to create sync capabilities between UObject and mass.

This gives the possibilites to sync in different directions (Actor → Mass, Mass → Actor, or Both).
- MassToActor → The Mass entity is sending data to the actor.
- ActorToMass → Actor is sending data to it´s mass entity.

> [!IMPORTANT]
> Each of these sync direction needs it´s own translator.



#### Example: A Health Component on an actor that can sync both ways.
##### Trait
```c++

USTRUCT()
struct FHealthComponentWrapperFragment : public FObjectWrapperFragment
{
	GENERATED_BODY()
	TWeakObjectPtr<UHealthComponent> Component;
};

UCLASS(MinimalAPI, BlueprintType, EditInlineNew, CollapseCategories, meta = (DisplayName = "Health Component Sync"))
class UHealthSyncTrait : public UMassAgentSyncTrait
{
	GENERATED_BODY()

protected:
	UE_API virtual void BuildTemplate(FMassEntityTemplateBuildContext& BuildContext, const UWorld& World) const override;
};

void UHealthSyncTrait::BuildTemplate(FMassEntityTemplateBuildContext& BuildContext, const UWorld& World) const
{
	BuildContext.AddFragment<FHealthComponentWrapperFragment>();
	BuildContext.AddFragment<FHealthFragment>();
	
	BuildContext.GetMutableObjectFragmentInitializers().Add([=](UObject& Owner, FMassEntityView& EntityView, const EMassTranslationDirection CurrentDirection)
		{
			UHealthComponent* HealthComp = nullptr;
			if (AActor* AsActor = Cast<AActor>(&Owner))
			{
				HealthComp = AsActor->FindComponentByClass<UHealthComponent>();
			}
			if (HealthComp)
			{
				FHealthComponentWrapperFragment& ComponentFragment = EntityView.GetFragmentData<FHealthComponentWrapperFragment>();
				ComponentFragment.Component = HealthComp;

				FHealthFragment& HealthFragment = EntityView.GetFragmentData<FHealthFragment>();

				// the mass entity is the authority
				if (CurrentDirection ==  EMassTranslationDirection::MassToActor)
				{
					HealthComp->Health = HealthFragment.Value;
				}
				// actor is the authority
				else
				{
					HealthFragment.Value = HealthComp->GetHealth();
				}
			}
		});
	//BothWays is bitwised so will be true for both ActorToMass and MassToActor.
	if (EnumHasAnyFlags(SyncDirection, EMassTranslationDirection::ActorToMass) || BuildContext.IsInspectingData())
	{
		BuildContext.AddTranslator<UHealthComponentToMassTranslator>();
	}

	if (EnumHasAnyFlags(SyncDirection, EMassTranslationDirection::MassToActor) || BuildContext.IsInspectingData())
	{
		BuildContext.AddTranslator<UHealthComponentToActorTranslator>();
	}
}
```
##### UMassTranslator
```c++
//This header will be the same for UHealthComponentToActorTranslator
UCLASS(MinimalAPI)
class UHealthComponentToMassTranslator : public UMassTranslator
{
	GENERATED_BODY()

public:
	UE_API UHealthComponentToMassTranslator();

protected:
	UE_API virtual void ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager) override;
	UE_API virtual void Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context) override;

	FMassEntityQuery EntityQuery;
};
```
```c++
UHealthComponentToMassTranslator::UHealthComponentToMassTranslator()
	: EntityQuery(*this)
{
	ExecutionFlags = (int32)EProcessorExecutionFlags::AllNetModes;
	ExecutionOrder.ExecuteInGroup = UE::Mass::ProcessorGroupNames::SyncWorldToMass;
	bRequiresGameThreadExecution = true;
}

void UHealthComponentToMassTranslator::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
	AddRequiredTagsToQuery(EntityQuery);
	EntityQuery.AddRequirement<FHealthComponentWrapperFragment>(EMassFragmentAccess::ReadOnly);
	EntityQuery.AddRequirement<FHealthFragment>(EMassFragmentAccess::ReadWrite);
}

void UHealthComponentToMassTranslator::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	EntityQuery.ForEachEntityChunk(Context, [this](FMassExecutionContext& Context)
	{
		const TConstArrayView<FHealthComponentWrapperFragment> ComponentList = Context.GetFragmentView<FHealthComponentWrapperFragment>();
		const TArrayView<FHealthFragment> HealthList = Context.GetMutableFragmentView<FHealthFragment>();
		
		for (FMassExecutionContext::FEntityIterator EntityIt = Context.CreateEntityIterator(); EntityIt; ++EntityIt)
		{
			if (const UHealthComponent* HealthComponent = ComponentList[EntityIt].Component.Get())
			{
				HealthList[EntityIt].Value = HealthComponent->GetHealth();
			}
		}
	});
}
```

```c++
//----------------------------------------------------------------------//
//  UMassCharacterMovementToActorTranslator
//----------------------------------------------------------------------//
UHealthComponentToActorTranslator::UHealthComponentToActorTranslator()
	: EntityQuery(*this)
{
	ExecutionFlags = (int32)EProcessorExecutionFlags::AllNetModes;
	ExecutionOrder.ExecuteInGroup = UE::Mass::ProcessorGroupNames::UpdateWorldFromMass;
	bRequiresGameThreadExecution = true;
}

void UHealthComponentToActorTranslator::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
	AddRequiredTagsToQuery(EntityQuery);
	EntityQuery.AddRequirement<FHealthComponentWrapperFragment>(EMassFragmentAccess::ReadWrite);
	EntityQuery.AddRequirement<FHealthFragment>(EMassFragmentAccess::ReadOnly);
}

void UHealthComponentToActorTranslator::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{
	EntityQuery.ForEachEntityChunk(Context, [this](FMassExecutionContext& Context)
	{
		const TArrayView<FHealthComponentWrapperFragment> ComponentList = Context.GetMutableFragmentView<FHealthComponentWrapperFragment>();
		const TConstArrayView<FHealthFragment> HealthList = Context.GetFragmentView<FHealthFragment>();

		for (FMassExecutionContext::FEntityIterator EntityIt = Context.CreateEntityIterator(); EntityIt; ++EntityIt)
		{
			if (UHealthComponent* HealthComponent = ComponentList[EntityIt].Component.Get())
			{
				HealthComponent->Value = HealthList[EntityIt].Value;
			}
		}
	});
}
```

<a name="Observers"></a>
### 9. **Observers**

- React to fragment/tag changes or entity initialization
- Only run when specific changes occur (not every frame)
- Use `UMassObserverProcessor` for this pattern

#### Example: A Death Observer that calls a subsystem on the gamethread.
```c++
UCLASS()
class UDeathObserver : public UMassObserverProcessor
{
	GENERATED_BODY()

public:
	UDeathObserver();

protected:
	virtual void ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager) override;
	virtual void Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context) override;

	FMassEntityQuery EntityQuery;

private:

};
```

```c++

UDeathObserver::UDeathObserver()
	: EntityQuery(*this)
{
	bRequiresGameThreadExecution = true;  
	ExecutionFlags = static_cast<int32>(EProcessorExecutionFlags::AllNetModes);
	ObservedType = FDeadTag::StaticStruct();
	Operation = EMassObservedOperation::Add;
}

void UDeathObserver::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
	EntityQuery.AddRequirement<FTransformFragment>(EMassFragmentAccess::ReadOnly);
	
	ProcessorRequirements.AddSubsystemRequirement<UGameModeSubsystem>(EMassFragmentAccess::ReadWrite);

}

void UDeathObserver::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{



	UGameModeSubsystem& GameModeSubsystem = Context.GetMutableSubsystemChecked<UGameModeSubsystem>();

	TArray<FVector> DeathLocations;

	EntityQuery.ForEachEntityChunk(Context, [this, &DeathLocations](FMassExecutionContext& Context)
		{

			DeathLocations.Reserve(DeathLocations.Num() + Context.GetNumEntities());

			
			const TConstArrayView<FTransformFragment> Transforms = Context.GetFragmentView<FTransformFragment>();

			for (FMassExecutionContext::FEntityIterator EntityIt = Context.CreateEntityIterator(); EntityIt; ++EntityIt)
			{
				const FTransformFragment& Transform = Transforms[EntityIt];

				DeathLocations.Add(Transform.GetTransform().GetLocation());

			
			}

			Context.Defer().DestroyEntities(Context.GetEntities());

			
		});

	if (!DeathLocations.IsEmpty())
	{
		GameModeSubsystem.BroadcastUnitsDeath(DeathLocations);
	}
}
```

<a name="Signals"></a>
### 9. **Signals**
- Send named signals between entities
- Lightweight event system for entity communication
- Part of the MassSignals module

#### Example: A Spawn observer that will use signals to tell the statetree to tick.
```c++
UCLASS()
class UEntitySpawnedInWorld : public UMassObserverProcessor
{
	GENERATED_BODY()

public:
	UEntitySpawnedInWorld();

protected:
	virtual void ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager) override;
	virtual void Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context) override;

	FMassEntityQuery EntityQuery;

private:



};
```

```c++


UEntitySpawnedInWorld::UEntitySpawnedInWorld()
	: EntityQuery(*this)
{
	ExecutionFlags = static_cast<int32>(EProcessorExecutionFlags::AllNetModes);
	ObservedType = FEntitySpawnedTag::StaticStruct(); //Can be added by a trait or other processor.
	Operation = EMassObservedOperation::Add;
}

void UEntitySpawnedInWorld::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{

	EntityQuery.AddSubsystemRequirement<UMassSignalSubsystem>(EMassFragmentAccess::ReadWrite);

}

void UEntitySpawnedInWorld::Execute(FMassEntityManager& EntityManager, FMassExecutionContext& Context)
{



	EntityQuery.ForEachEntityChunk(Context, [](FMassExecutionContext& Context)
		{
			UMassSignalSubsystem& SignalSubsystem = Context.GetMutableSubsystemChecked<UMassSignalSubsystem>();

	
			for (FMassExecutionContext::FEntityIterator EntityIt = Context.CreateEntityIterator(); EntityIt; ++EntityIt)
			{
				
				const FMassEntityHandle Entity = Context.GetEntity(EntityIt);
				Context.Defer().RemoveTag<FEntitySpawnedTag>(Entity);
			}

			SignalSubsystem.SignalEntities(UE::Mass::Signals::NewStateTreeTaskRequired, Context.GetEntities());

		});
}

```
#### Example: Listening to signals from a processor.
```c++
UCLASS(MinimalAPI)
class UTestSignalProcessor : public UMassSignalProcessorBase
{
	GENERATED_BODY()

public:
	UTestSignalProcessor(const FObjectInitializer& ObjectInitializer = FObjectInitializer::Get());

	
protected:
	virtual void InitializeInternal(UObject& Owner, const TSharedRef<FMassEntityManager>& EntityManager) override;
	virtual void ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager) override;
	virtual void SignalEntities(FMassEntityManager& EntityManager, FMassExecutionContext& Context, FMassSignalNameLookup& EntitySignals) override;
};
```
```c++
UTestSignalProcessor::UTestSignalProcessor(const FObjectInitializer& ObjectInitializer)
	: Super(ObjectInitializer)
{

	ExecutionOrder.ExecuteBefore.Add(UE::Mass::ProcessorGroupNames::Tasks);

}

void UTestSignalProcessor::InitializeInternal(UObject& Owner, const TSharedRef<FMassEntityManager>& EntityManager)
{
	Super::InitializeInternal(Owner, EntityManager);

	UMassSignalSubsystem* SignalSubsystem = UWorld::GetSubsystem<UMassSignalSubsystem>(Owner.GetWorld());

	SubscribeToSignal(*SignalSubsystem, UE::Mass::Signals::NewStateTreeTaskRequired);
}

void UTestSignalProcessor::ConfigureQueries(const TSharedRef<FMassEntityManager>& EntityManager)
{
	
	EntityQuery.AddSubsystemRequirement<UMassSignalSubsystem>(EMassFragmentAccess::ReadWrite);

	
}

void UTestSignalProcessor::SignalEntities(FMassEntityManager& EntityManager, FMassExecutionContext& Context, FMassSignalNameLookup& EntitySignals)
{

	EntityQuery.ForEachEntityChunk(Context, [&EntitySignals](FMassExecutionContext& Context)
		{
			
			for (FMassExecutionContext::FEntityIterator EntityIt = Context.CreateEntityIterator(); EntityIt; ++EntityIt)
			{
				TArray<FName> Signals;
				EntitySignals.GetSignalsForEntity(Context.GetEntity(EntityIt), Signals);
				if(Signals.Contains(UE::Mass::Signals::NewStateTreeTaskRequired))
				{
					//Do something when NewStateTreeTaskRequired signal is called.
				}
			}
		});
}
```

---

<a name="architecture-overview"></a>
## Architecture Overview

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

### Commands

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
- [Understand MASS with Unreal Engine](https://www.youtube.com/watch?v=eJR82WyIl_U) - Video presentation by Epic
- [Large Numbers of Entities with Mass in UE5](https://www.youtube.com/watch?v=f9q8A-9DvPo) - Video presentation by Epic
- [Understand MASS with Unreal Engine](https://www.youtube.com/watch?v=eJR82WyIl_U) - Video presentation Rogue Entity

  
### Related Systems
- **ZoneGraph**: Spline-based navigation for Mass crowds (sidewalks, roads, etc.). Much cheaper than NavMesh but very limited.
- **StateTree**: Lightweight state machine for Mass AI. 
- **Smart Objects**: Interactive objects that Mass entities can use
- **Mass Avoidance**: Force-based avoidance system
- **Mass Replication**: Network replication for Mass entities (experimental)

---

## Tips

1. **Start Simple**: Begin with a few fragments and one processor before scaling up
2. **Think in Data**: Focus on what data you need, not what objects do
3. **Batch Operations**: Use command buffers to queue changes rather than modifying entities immediately
4. **Profile Early**: Use Unreal's profiling tools to measure performance gains
5. **Use Tags Wisely**: Tags are excellent for filtering with less data overhead. Each archetype holds a bitset that contains the tag presence information.
6. **Learn from Examples**: Study UE5's City Sample to see Mass in action at scale or go to MassSample Github for more examples.
7. **Keep fragments focused**: If a fragment starts holding unrelated data (e.g., health + attack + attack speed), consider splitting it into multiple fragments (HealthFragment,AttackFragment).
8. **Use traits to group fragments**: Traits let you assign multiple fragments together when needed, without bloating one fragment.
9. **Think about update frequency**: If some data is rarely updated, it may belong in a different fragment so processors don’t touch it every frame.
10. **Data type size**: Consider what data type you really need. Maybe you can use uint16 instead of int32? Size matters in the long run.
---

## License

Mass is part of Unreal Engine 5 and follows Epic Games' licensing terms. Check the [Unreal Engine EULA](https://www.unrealengine.com/eula) for details.

---

**Note**: Some of Mass plugins is marked as **Experimental** in Unreal Engine 5. While it's powerful and used in production (like the City Sample), APIs may change and Epic doesn't recommend shipping with experimental features without thorough testing.
