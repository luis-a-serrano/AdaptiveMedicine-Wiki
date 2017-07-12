The *Experiment Manager* actor type defines a class that takes care of the configuration details for an experiment as well as providing an access point to all the other actor instances related to an experiment. Therefore, this actor is the gatekeeper of the whole experiment and the only one that directly acts upon the experiment as a result of human interaction. Each actor instance from this type manages a different experiment, and each experiment is independent and isolated from the others.

### Statechart Backbone
The following is a statechart diagram that describes the core behavior of an instance of the *Experiment Manager* actor type.

| ![Experiment Manager Actor Statechart](images/ExperimentManagerActorStatechart.png) |
| --- |
| Some transitions have multiple events tied to them. This only applies if the action related to the transition is the same for all the events listed. Otherwise, the transitions would be presented as separate ones. |

What follows are descriptions of the states and the transitions that originate from them.

#### Uninitialized State
This state implies that the experiment manager doesn't have any configuration details stored, either because they've never been set or because they were removed.

##### AddParticipant and Error Events
An *AddParticipant* event only make sense if the statechart already has the configuration details saved for this instance. Therefore, if this is not true the statechart moves to the *Illegal* state. An *Error* event always make the instance move to the *Illegal* state as it is produced, for the most past, when a problem arises during the processing of an action.

##### Initialize Event
An *Initialize* event is necessary to move the statechart to the *Initialized* state. This transition executes an action that saves the configuration details tied to the consumed event. If for some reason the data is missing then an *Error* event is consumed right after the transition reaches the new state.

##### Delete Event
A *Delete* event tells the statechart instance to remove the saved configuration details, forcing it to be at an *Uninitialized* state.

#### Initialized State
This state implies that the configuration details for the experiment manager were stored, and therefore the instance is now ready to process requests that make use of said information.

##### AddParticipant Event
The purpose of an *AddParticipant* event is to add a new participant to the experiment. The transition action executed when said event is dispatched uses it to get information about the participant that wants to join. A reference to the participant is saved and then, by using the configuration details previously saved it creates the instances that will manage that patients involvement with the experiment.

##### Delete Event
A *Delete* event tells the statechart instance to remove the saved configuration details, forcing it to be at an *Uninitialized* state.

##### Initialize and Error Events
An *Initialize* event only make sense if the statechart doesn't have any configuration details saved for this instance. Therefore, if it does then the statechart moves to the *Illegal* state. An *Error* event always make the instance move to the *Illegal* state as it is produced, for the most past, when a problem arises during the processing of an action.

#### Illegal State
This state implies that an error occurred during the processing of a transition action, which might further imply that at some point there was a problem during the path traversal of the instance. In order for the statechart to be "legal" again it would need to transition to the initial state before processing any new event. Thus, for any event dispatched while the instance is on this state it simply forwards it to the initial state.

### Actor Language
The *Experiment Manager* actor type is realized by `ExperimentManagerActor`. The class implements the `IExperimentManagerActor` interface which exposes the `ConfigurateAsync` and `AddParticipantAsync` methods. The first method is used to setup all the details related to the experiment. This creates and dispatches an *Initialize* event on the statechart backbone. The second method is used to add a new participant to an existing, and running, experiment. This creates and dispatches an *AddParticipant* event on the statechart backbone. These methods are mostly used by the API gateway instead of by other actors.
