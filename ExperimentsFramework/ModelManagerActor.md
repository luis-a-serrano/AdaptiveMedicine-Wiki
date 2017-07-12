The *Model Manager* actor type defines a class that creates and takes care of a set of model instances. A reference to all these instances is kept in case it's necessary to update any of them; be it because of new data or a new algorithm to use. Each actor instance of this type is independent and isolated of each other and manages a *different* participant for a *specific* experiment.

### Statechart Backbone
The following is a statechart diagram that describes the core behavior of an instance of the *Model Manager* actor type.

| ![Model Manager Actor Statechart](images/ModelManagerActorStatechart.png) |
| --- |
| Some transitions have multiple events tied to them. This only applies if the action related to the transition is the same for all the events listed. Otherwise, the transitions would be presented as separate ones. |

What follows are descriptions of the states and the transitions that originate from them.

#### Uninitialized State
This state implies that the model manager doesn't have any model references stored, either because they've never been set or because they were removed.

##### Initialize Event
An *Initialize* event is necessary to move the statechart to the *Initialized* state. This transition executes an action that uses the configuration details tied to the consumed event and creates the necessary algorithm (models) instances. For each instance a reference is saved for future communications with those instances. If for some reason the data is missing then an *Error* event is consumed right after the transition reaches the new state.

##### Delete Event
A *Delete* event tells the statechart instance to remove the saved references, forcing it to be at an *Uninitialized* state.

##### Error Event
An *Error* event always make the instance move to the *Illegal* state as it is produced, for the most past, when a problem arises during the processing of an action.

#### Initialized State
This state implies that the model references produced by a previous *Initialize* event were stored, and therefore the instance is now ready to process requests that make use of said information.

##### Delete Event
A *Delete* event tells the statechart instance to remove the saved model references, forcing it to be at an *Uninitialized* state.

##### Initialize and Error Events
An *Initialize* event only make sense if the statechart doesn't have any model references saved for this instance. Therefore, if it does then the statechart moves to the *Illegal* state. An *Error* event always make the instance move to the *Illegal* state as it is produced, for the most past, when a problem arises during the processing of an action.

#### Illegal State
This state implies that an error occurred during the processing of a transition action, which might further imply that at some point there was a problem during the path traversal of the instance. In order for the statechart to be "legal" again it would need to transition to the initial state before processing any new event. Thus, for any event dispatched while the instance is on this state it simply forwards it to the initial state.

### Actor Language
The *Model Manager* actor type is realized by `ModelManagerActor`. The class implements the `IModelManagerActor` interface which exposes the `ConfigurateAsync` method. This method is used to setup all the details related to the models. This creates and dispatches an *Initialize* event on the statechart backbone. This method is mostly used by the API gateway instead of by other actors.
