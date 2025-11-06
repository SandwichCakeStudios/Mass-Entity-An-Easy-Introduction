# Mass Framework for Unreal Engine: An Easy Introduction

The Mass Framework in Unreal Engine is a powerful, modern system designed to handle massive numbers of entities (like crowds, particles, or complex simulations) with exceptional performance and scalability.

It achieves this by using a Data-Oriented Design (DOD) approach, moving away from the traditional Actor/Component workflow to better utilize modern CPU architectures and memory.

## Why Mass? (The Benefits)
Mass is essentially an Entity Component System (ECS). Its design focuses on efficient data handling, making it great for:

- Optimization & Performance: Data for similar entities is stored in contiguous memory (Arrays of Structs), which is very "cache-friendly." This minimizes costly CPU cache misses and speeds up calculations.

- Scalability: Easily handles thousands of entities because new functionality is added by composing data (Fragments) rather than deep class inheritance.

- Parallelism: The data structure allows safe, easy multithreading and parallel processing across different "chunks" of data.

- Composability: Entity behavior is created by adding and removing data pieces (Fragments), avoiding complex inheritance chains.

## When NOT to Use Mass (The Drawbacks)
While powerful, Mass isn't a replacement for everything:

- Low Entity Counts: The setup and processing systems have an overhead, making it less suitable for systems with only a few entities.

- Indirect Logic: The processing logic is contained in separate Processors, not on the entities themselves, which is a different way of thinking than typical object-oriented programming.

- Costly Structural Changes: Frequently adding or removing Fragments on an Entity (Archetype migration) can be costly. This is typically managed by delaying changes until safe "sync points".

## Mass Key Terminology (ECS for Dummies)
Mass uses slightly different names for standard ECS concepts to avoid confusion with existing Unreal Engine code:

| Standard ECS Term | Mass Terminology | Description |
| ----------- | ----------- | ----------- |
| Entity | Entity | A small, unique ID that represents a single object instance and points to its data. |
| Component | Fragment | Holds the actual data or state of an Entity (e.g., transform, velocity). Stored in tightly packed arrays. |
| System | Processor | Where the logic and execution happens. Processors define queries to filter for entities with specific Fragments and then run logic on them. |


### Core Data Concepts
- Archetype: A unique combination of Fragments and Tags. Entities with the exact same composition belong to the same Archetype.

  - Composition Change: If an Entity gains or loses a Fragment at runtime, it migrates to a new Archetype.

- Chunk: An Archetype owns many Chunks. A Chunk is a fixed-size block of tightly packed, contiguous data that stores the Fragments for a subset of the Archetype's Entities. This chunking is key to memory locality. * Tags: Archetype-level dataless Fragments (empty structs) used primarily by Queries to filter entities based on their presence or absence.

- Traits: Assets that group Fragments and Tags together, often representing a feature (e.g., Movement). They act like a blueprint/prefab for defining an entity's features.

- Shared Fragment: Data that is shared across multiple Entities for memory saving.

# How Mass Works (Processing Flow)
The logic is executed inside Processors, which are assigned to a Processing Phase (like PrePhysics or PostPhysics) that corresponds to an Unreal Tick Group.

1. Configure Query: A Processor starts by setting up an Entity Query (an FMassEntityQuery) to define exactly what Fragments and Tags an Entity must have (or not have) to be processed.

2. Filter Archetypes: The query efficiently matches its requirements to the Archetypes (and filters their Chunks).

3. Batch Processing: The Processor doesn't run on individual entities one by one. Instead, it iterates over the matching data chunk-by-chunk using a function like ForEachEntityChunk.

  - This batch-oriented, data-driven approach is what allows for simple and incredibly high-performance operations.

4. Execution: The Processor performs its logic (e.g., "movement processor adds velocity to transform") on the tightly packed data in the chunk.

### Management
- FMassEntityManager: This core class owns all Entities and Archetypes and manages Composition Changes (Entity migrations).
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

- Commands/Deferred Changes: Changes like adding or removing Fragments are queued into a Command Buffer and applied in batches at safe points to avoid data fragmentation during active processing.
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

## Resources 
This section includes all resources information that was used to generate this page.

  - [[Documentation] MassEntity](https://docs.unrealengine.com/5.0/en-US/overview-of-mass-entity-in-unreal-engine/): Overview of Unreal Engine's MassEntity system.
 - [[Your first 60 minutes of mass] MassEntity](https://dev.epicgames.com/community/learning/tutorials/JXMl/unreal-engine-your-first-60-minutes-with-mass): Tutorial on mass.
 - [Megafunk/MassSample](https://github.com/Megafunk/MassSample): Megafunk github page. Lots of detailed information on mass.
