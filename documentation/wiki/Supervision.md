---
layout: wiki
title: Supervision
---
This chapter outlines the concept behind supervision, the primitives offered and their semantics. For details on how that translates into real code, please refer to the corresponding chapters for Scala and Java APIs.

##What Supervision Means
As described in Actor Systems supervision describes a dependency relationship between actors: the supervisor delegates tasks to subordinates and therefore must respond to their failures. When a subordinate detects a failure (i.e. throws an exception), it suspends itself and all its subordinates and sends a message to its supervisor, signaling failure. Depending on the nature of the work to be supervised and the nature of the failure, the supervisor has a choice of the following four options:

Resume the subordinate, keeping its accumulated internal state
Restart the subordinate, clearing out its accumulated internal state
Stop the subordinate permanently
Escalate the failure, thereby failing itself
It is important to always view an actor as part of a supervision hierarchy, which explains the existence of the fourth choice (as a supervisor also is subordinate to another supervisor higher up) and has implications on the first three: resuming an actor resumes all its subordinates, restarting an actor entails restarting all its subordinates (but see below for more details), similarly terminating an actor will also terminate all its subordinates. It should be noted that the default behavior of the preRestart hook of the Actor class is to terminate all its children before restarting, but this hook can be overridden; the recursive restart applies to all children left after this hook has been executed.

Each supervisor is configured with a function translating all possible failure causes (i.e. exceptions) into one of the four choices given above; notably, this function does not take the failed actor’s identity as an input. It is quite easy to come up with examples of structures where this might not seem flexible enough, e.g. wishing for different strategies to be applied to different subordinates. At this point it is vital to understand that supervision is about forming a recursive fault handling structure. If you try to do too much at one level, it will become hard to reason about, hence the recommended way in this case is to add a level of supervision.

Akka implements a specific form called “parental supervision”. Actors can only be created by other actors—where the top-level actor is provided by the library—and each created actor is supervised by its parent. This restriction makes the formation of actor supervision hierarchies implicit and encourages sound design decisions. It should be noted that this also guarantees that actors cannot be orphaned or attached to supervisors from the outside, which might otherwise catch them unawares. In addition, this yields a natural and clean shutdown procedure for (sub-trees of) actor applications.

>**Warning**<br/>
>Supervision related parent-child communication happens by special system messages that have their own <br/>
>mailboxes separate from user messages. This implies that supervision related events are not deterministically <br/>
>ordered relative to ordinary messages. In general, the user cannot influence the order of normal messages and <br/>
>failure notifications. For details and example see the Discussion: Message Ordering section.<br/>

## The Top-Level Supervisors

![Top level supervisors](images/TopLevelSupervisors.png)

An actor system will during its creation start at least three actors, shown in the image above. For more information about the consequences for actor paths see Top-Level Scopes for Actor Paths.

### `/user`: The Guardian Actor
The actor which is probably most interacted with is the parent of all user-created actors, the guardian named `"/user"`. Actors created using `system.ActorOf()` are children of this actor. This means that when this guardian terminates, all normal actors in the system will be shutdown, too. It also means that this guardian’s supervisor strategy determines how the top-level normal actors are supervised. Since Akka.NET 1.0 it is possible to configure this using the setting `akka.actor.guardian-supervisor-strategy`, which takes the fully-qualified class-name of a `SupervisorStrategyConfigurator`. When the guardian escalates a failure, the root guardian’s response will be to terminate the guardian, which in effect will shut down the whole actor system.

### `/system`: The System Guardian
This special guardian has been introduced in order to achieve an orderly shut-down sequence where logging remains active while all normal actors terminate, even though logging itself is implemented using actors. This is realized by having the system guardian watch the user guardian and initiate its own shut-down upon reception of the `Terminated` message. The top-level system actors are supervised using a strategy which will restart indefinitely upon all types of Exception except for `ActorInitializationException` and `ActorKilledException`, which will terminate the child in question. All other exceptions are escalated, which will shut down the whole actor system.

### `/`: The Root Guardian
The root guardian is the grand-parent of all so-called “top-level” actors and supervises all the special actors mentioned in Top-Level Scopes for Actor Paths using the `SupervisorStrategy.StoppingStrategy`, whose purpose is to terminate the child upon any type of Exception. All other throwables will be escalated … but to whom? Since every real actor has a supervisor, the supervisor of the root guardian cannot be a real actor. And because this means that it is “outside of the bubble”, it is called the “bubble-walker”. This is a synthetic ActorRef which in effect stops its child upon the first sign of trouble and sets the actor system’s isTerminated status to true as soon as the root guardian is fully terminated (all children recursively stopped).

## What Restarting Means
When presented with an actor which failed while processing a certain message, causes for the failure fall into three categories:

* Systematic (i.e. programming) error for the specific message received
* (Transient) failure of some external resource used during processing the message
* Corrupt internal state of the actor

Unless the failure is specifically recognizable, the third cause cannot be ruled out, which leads to the conclusion that the internal state needs to be cleared out. If the supervisor decides that its other children or itself is not affected by the corruption—e.g. because of conscious application of the error kernel pattern—it is therefore best to restart the child. This is carried out by creating a new instance of the underlying Actor class and replacing the failed instance with the fresh one inside the child’s `ActorRef`; the ability to do this is one of the reasons for encapsulating actors within special references. The new actor then resumes processing its mailbox, meaning that the restart is not visible outside of the actor itself with the notable exception that the message during which the failure occurred is not re-processed.

The precise sequence of events during a restart is the following:

1. Suspend the actor (which means that it will not process normal messages until resumed), and recursively suspend all children.
2. Call the old instance’s `PreRestart` hook (defaults to sending termination requests to all children and calling postStop)
3. Wait for all children which were requested to terminate (using `context.Stop()`) during `PreRestart` to actually terminate; this—like all actor operations—is non-blocking, the termination notice from the last killed child will effect the progression to the next step.
4. Create new actor instance by invoking the originally provided factory again.
5. Invoke `PostRestart` on the new instance (which by default also calls `PreStart`)
6. Send restart request to all children which were not killed in step 3; restarted children will follow the same process recursively, from step 2
7. Resume the actor.

##What Lifecycle Monitoring Means
>**Note**<br/>
>Lifecycle Monitoring in Akka is usually referred to as `DeathWatch`

In contrast to the special relationship between parent and child described above, each actor may monitor any other actor. Since actors emerge from creation fully alive and restarts are not visible outside of the affected supervisors, the only state change available for monitoring is the transition from alive to dead. Monitoring is thus used to tie one actor to another so that it may react to the other actor’s termination, in contrast to supervision which reacts to failure.

Lifecycle monitoring is implemented using a `Terminated` message to be received by the monitoring actor, where the default behavior is to throw a special `DeathPactException` if not otherwise handled. In order to start listening for Terminated messages, invoke `ActorContext.Watch(targetActorRef)`. To stop listening, invoke `ActorContext.Unwatch(targetActorRef)`. One important property is that the message will be delivered irrespective of the order in which the monitoring request and target’s termination occur, i.e. you still get the message even if at the time of registration the target is already dead.

Monitoring is particularly useful if a supervisor cannot simply restart its children and has to terminate them, e.g. in case of errors during actor initialization. In that case it should monitor those children and re-create them or schedule itself to retry this at a later time.

Another common use case is that an actor needs to fail in the absence of an external resource, which may also be one of its own children. If a third party terminates a child by way of the `system.Stop(child)` method or sending a `PoisonPill`, the supervisor might well be affected.

###One-For-One Strategy vs. All-For-One Strategy
There are two classes of supervision strategies which come with Akka: `OneForOneStrategy` and `AllForOneStrategy`. Both are configured with a mapping from exception type to supervision directive (see above) and limits on how often a child is allowed to fail before terminating it. The difference between them is that the former applies the obtained directive only to the failed child, whereas the latter applies it to all siblings as well. Normally, you should use the `OneForOneStrategy`, which also is the default if none is specified explicitly.

![One for one](images/OneForOne.png)

The `AllForOneStrategy` is applicable in cases where the ensemble of children has such tight dependencies among them, that a failure of one child affects the function of the others, i.e. they are inextricably linked. Since a restart does not clear out the mailbox, it often is best to terminate the children upon failure and re-create them explicitly from the supervisor (by watching the children’s lifecycle); otherwise you have to make sure that it is no problem for any of the actors to receive a message which was queued before the restart but processed afterwards.

![All for one](images/AllForOne.png)

Normally stopping a child (i.e. not in response to a failure) will not automatically terminate the other children in an all-for-one strategy; this can easily be done by watching their lifecycle: if the `Terminated` message is not handled by the supervisor, it will throw a `DeathPactException` which (depending on its supervisor) will restart it, and the default `PreRestart` action will terminate all children. Of course this can be handled explicitly as well.

Please note that creating one-off actors from an all-for-one supervisor entails that failures escalated by the temporary actor will affect all the permanent ones. If this is not desired, install an intermediate supervisor; this can very easily be done by declaring a router of size 1 for the worker, see [[Routing]].

####Example supervisor strategy

```csharp
public class MyActor : UntypedActor
{
    private ActorRef logger = Context.ActorOf<LogActor>();

    // if any child, e.g. the logger above. throws an exception
    // apply the rules below
    // e.g. Restart the child, if 10 exceptions occur in 30 seconds or 
    // less, then stop the actor
    protected override SupervisorStrategy SupervisorStrategy()
    {
        return new OneForOneStrategy( //or AllForOneStrategy
            maxNumberOfRetries: 10, 
            duration: TimeSpan.FromSeconds(30), 
            decider: x =>
            {
                //Maybe we consider ArithmeticException to not be application critical
                //so we just ignore the error and keep going.
                if (x is ArithmeticException) return Directive.Resume;

                //Error that we cannot recover from, stop the failing actor
                else if (x is NotSupportedException) return Directive.Stop;

                //In all other cases, just restart the failing actor
                else return Directive.Restart;
            });
    }
    
    ...
}
```