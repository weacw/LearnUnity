## using the Job System with ECS

The iterators we get from ComponentGroups (ComponentDataArray, EntityArray, *et al.*) are valid job fields, but jobs touching entity components should only be created in the context of *JobComponentSystems* (described later).

Like native containers, the entity component iterators have job safety checks:

```csharp
// assume jobs A and B reference the same ComponentDataArray
JobHandle a = jobA.Schedule();
JobHandle b = jobB.Schedule();     // exception!
```

...but because two separate iterators can access the same entity components in memory, the checks have to also detect conflicts between separate iterators:

```csharp
// assume jobs A and B reference separate ComponentDataArrays which 
// access the same components in memory
JobHandle a = jobA.Schedule();
JobHandle b = jobB.Schedule();     // exception!
```

Additionally, right before a system *OnUpdate()* is called, an exception is thrown if any of the system's ComponentGroups conflict with a scheduled job.

#### JobComponentSystem

A **JobComponentSystem** is like a ComponentSystem but helps us create jobs with appropriate dependencies that avoid entity component access conflicts. Unlike in a normal ComponentSystem, the *OnUpdate()* method receives a job handle and returns a job handle:

1. Immediately after each JobComponentSystem's *OnUpdate()* returns, *JobHandle.ScheduleBatchedJobs()* is called, thus starting execution of any jobs scheduled in the *OnUpdate()*.
2. Immediately before *OnUpdate()*, *Complete()* is called on the JobHandle returned by the system's previous *OnUpdate()* call. So a JobComponentSystem is intended for jobs that at most take a frame to complete.
3. The Job System is aware of which ComponentSystems access which ComponentGroups by virtue of each ComponentSystem's *GetComponentGroup()* calls. The job handle passed to *OnUpdate()* combines the job handles returned by the other JobComponentSystems updated previously in the frame which access ComponentGroups conflicting with those accessed in this JobComponentSystem.

For example, say we have JobComponentSystems A, B, C, D, and E, updated in that order. If E's ComponentGroups conflict with A and D's ComponentGroups, then the JobHandle passed to the *OnUpdate()* of E will combine the JobHandles returned by A and D, such that E's jobs can depend upon A and D's.

The sum effect is that, if all jobs created in every JobComponentSystem:

1. ...only use entity-component iterators from their JobComponentSystem's own ComponentGroups
2. ...*and* use their *OnUpdate()*'s input JobHandle as a dependency (direct or indirect)
3. ...*and* are themselves dependencies of their *OnUpdate()*'s returned JobHandle (or *are* the returned JobHandle itself) 

...then all jobs created within each JobComponentSystem will avoid component access conflicts with the jobs created in all other JobComponentSystems.

For the jobs created *within* a single JobComponentSystem, the editor runtime safety checks will detect when two scheduled jobs conflict in their component access, but it is our responsibility to manually resolve these conflicts by making one job the dependency of the other.

If jobs created in two separate JobComponentSystems conflict, we should specify which *OnUpdate()* should run first with the *UpdateBefore* and *UpdateAfter* attributes. For example, if the jobs of JobComponentSystem A should mutate components before they are read in the jobs of JobComponentSystem B, then we should specify that A must update before B.

Nothing stops us from creating jobs in a JobComponentSystem's *OnUpdate()* which do not depend upon the input JobHandle and which are not themselves dependencies of the returned JobHandle&mdash;but doing so generally defeats the purpose of a JobComponentSystem.

If we want to complete a JobComponentSystem's jobs earlier than the system's next update, we can inject a BarrierSystem: before flushing its EntityCommandBuffers in its update, a BarrierSystem completes the job handles returned by any JobComponentSystems which inject the BarrierSystem.

While it's possible and sometimes useful to create jobs that run longer than a frame, we generally avoid multi-frame jobs which access component groups. Jobs which access entity components should only be created in JobComponentSystems, and these jobs are always completed by the system's next update at the latest. For a multi-frame job which needs to read entity components, we can copy the data to native containers and then use those native containers in the job.