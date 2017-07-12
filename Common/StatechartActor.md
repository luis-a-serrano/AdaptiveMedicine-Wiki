The *Statechart Actor* is a library that define classes that can be used to add statechart capabilities to a actors. It allows any actor that implements it to specify a statechart structure by defining states and transitions. With those in place it can then dispatch events to execute actions and change states.

### StatechartActor Class
In order for an actor type to have statechart capabilities it needs to derive from the `StatechartActor` abstract class instead of the typical `Actor` class. Then by defining nested classes it can describe the structure of the statechart. These classes will derive from the `StatechartClass` abstract class and will use the `InitialStateAttribute`, `StateAttribute` and `TransitionAttribute` attributes extensively. During the creation of an actor type object said attributes might be read in order to construct the whole structure, which in turn might be used by multiple actor instances. Therefore, when defining the structure we assume that it's not tied to a specific instance. Instead, a reference to an instance will be passed to the appropriate state or transition when necessary.

> *Note*: The statechart structure defined by the actor type will be constructed only once for all the actor instances running on the same application domain.

<!-- -->
> *Note*: If the statechart structure doesn't follow the correct format the constructed statechart might have missing states, transitions or throw errors.

The `StatechartActor` class implements the `IStatechartActor` interface which only exposes the `DispatchEventAsync` method. This method is used by an actor instance to dispatch events to the statechart associated with its type. The method requires an event to be passed, which can be constructed by using any of the `StatechartEvent` class constructors. Typically this method is used once inside an method exposed by the actor type interface.

This is an example of how this abstract class would be used.
```C#
internal class VendingMachineActor: StatechartActor, IVendingMachineActor {
   public VendingMachineActor(ActorService actorService, ActorId actorId)
       : base(actorService, actorId) {
   }

   protected override Task OnActivateAsync() {
      return base.OnActivateAsync();
   }

   async Task<bool> IVendingMachineActor.ChooseProductAsync(string productId) {
      await DispatchEventAsync(new StatechartEvent("ProductChosen", DateTime.Now, productId));
      return true;
   }

   async Task<bool> IVendingMachineActor.DepositMoneyAsync(float moneyAmount) {
      await DispatchEventAsync(new StatechartEvent("MoneyDeposited", DateTime.Now, moneyAmount));
      return true;
   }
}
```
> *Note*: If you want to override the `OnActivateAsync` method, then make sure that you call the base implementation as well. Also, any `DispatchEventAsync` calls inside of here need to happen after that call. Otherwise, the statechart will not function correctly.

### StatechartState Class
The `StatechartState` abstract class is used to define the states that form part of a statechart structure. The class that derives from this one need to be nested inside the class that derives from the `StatechartActor`, and either the `InitialStateAttribute` or `StateAttribute` attributes need to be used on the class itself. An enum or string identifier must be passed to the attribute; the identifier is used to name the state with an unique name. The class also needs to follow the singleton pattern and create an `Instance` public static property that holds a reference to an instance object of this class.

> *Note*: The `InitialStateAttribute` attribute must be used only on one of the state classes nested inside the parent statechart class. The rest should use the `StateAttribute`.

<!-- -->
> *Note*: When implementing the singleton pattern make sure that you call the base constructor as part of your private constructor. Otherwise, the statechart will not function correctly.

The `StatechartState` class implements the `IState` interface which exposes the `Type` property. This property returns the identifier used for the class. The class also exposes the `EntryActionAsync` and `ExitActionAsync` methods which can be overridden. These methods are executed whenever a running statechart enters or exits (respectively) this state after following a transition originating from *another* state or targeting *another* state. The final exposed method is `GetActivatedTransition` which returns the transition method that would be activated for this state if the provided event is dispatched and the running statechart is at this state. This method is mostly used by the `DispatchEventAsync` method of the parent statechart class.

This is an example of how this abstract class would be used.
```C#
[InitialState("Uninitialized")]
private sealed class Uninitialized: StatechartState {
   #region Singleton Pattern
   private static readonly Lazy<Uninitialized> _Lazy = new Lazy<Uninitialized>(() => new Uninitialized());
   public static Uninitialized Instance { get { return _Lazy.Value; } }
   private Uninitialized() : base() { }
   #endregion

   public override async Task<bool> EntryActionAsync(Actor actor) {
      await actor.StateManager.SetStateAsync<bool>("ReadyForData", true);
      return true;
   }
}
```

### StatechartTransition Class and TransitionAttribute Attribute
The `StatechartTransition` abstract class is mainly used internally by the `StatechartState` class in order to keep track of the transitions that originate from that state and the actions executed with them. This is accomplished thanks to the `TransitionAttribute` attribute, which is applied to methods inside a state. The methods are the actions executed when a particular event is consumed and a transition occurs.
The first argument that must be passed to the attribute is the name/category of the event that would trigger the transition. The second argument, that is optional, is the name of the state to which we transition when the event is consumed. If no state is specified then we consider the target state to be the same as the originator. A method doesn't need to be specific for each transition, thus multiple transition attributes can be specified for it.

> *Note*: When specifying the transition attributes make sure that no event name is repeated inside the same state. Otherwise, the statechart will not function correctly.

The signature of the method should be `Task<IEnumerable<IEvent>> MethodName(IEvent anEvent, Actor actor)` in order to work properly. The first argument is an `IEvent` object which holds information about the current event. The second argument is a reference to the actor currently running the statechart (because the state class is running as a singleton object it needs to be passed this reference). It is possible for an action to produce events, which would be consumed immediately after the current action. To achieve this the action simply needs to create and return said events.

This is an example of how this attribute would be used.
```C#
[InitialState("Uninitialized")]
private sealed class Uninitialized: StatechartState {
   #region Singleton Pattern
   // Implementation code.
   #endregion

   [Transition("Initialize", "Initialized")]
   public async Task<IEnumerable<IEvent>> SetConfigurationAsync(IEvent anEvent, Actor actor) {
      var config = (string) anEvent.Input;
      if (config != null) {
         await actor.StateManager.SetStateAsync<string>("configuration", config);
      }
      
      return null;
   }

   [Transition("Delete")]
   [Transition("Error", "Illegal")]
   public Task<IEnumerable<IEvent>> DoNothingAsync(IEvent anEvent, Actor actor) {
      return Task.FromResult<IEnumerable<IEvent>>(null);
   }
}
```

### StatechartEvent Class
The `StatechartEvent` class is used to create events that are consumed by the statechart, either by means of a call to `DispatchEventAsync` or by being returned from a method that's being annotated with the `TransitionAttribute` attribute. The `StatechartEvent` class implements the `IEvent` interface which exposes the `Type` property. This property returns the name/category used for the event. The class also exposes the `Input` property, which can be used to pass pertinent information to the action that will be executed as part of the transition. Finally, the exposed `Id` property holds a timestamp that represents the moment that the event was conceptually created.

> *Note*: While the `Input` property is mostly used by the actions, the`Type` and  `Id` properties are used by the running statechart to determine if it should consume said event. More specifically, for each category the statechart keeps track of the last event that was consumed. If an event arrives of a category that it has handled before then the statechart will only consume that event if the timestamp is newer than the previously consumed one. If not then the event is discarded.

This is an example of how this abstract class would be used.
```C#
[InitialState("Uninitialized")]
private sealed class Uninitialized: StatechartState {
   #region Singleton Pattern
   // Implementation code.
   #endregion

   [Transition("Initialize", "Initialized")]
   public async Task<IEnumerable<IEvent>> SetConfigurationAsync(IEvent anEvent, Actor actor) {
      var forwardedEvents = new List<IEvent>();
      var config = (string) anEvent.Input;
      if (config != null) {
         await actor.StateManager.SetStateAsync<string>("configuration", config);
      } else {
         forwardedEvents.Add(new StatechartEvent("Error", anEvent.Id, "No configuration data received."));
      }
      
      return forwardedEvents;
   }

   // Other transitions.
}
```