# Lesson 2.4: Switching Actor Behavior at Run-time with `Become` and `Unbecome`

In this lesson we're going to learn about one of the really cool things actors can do: [change their behavior at run-time](http://getakka.net/wiki/Working%20with%20actors#hotswap "Akka.NET - Actor behavior hotswap")!

## Key Concepts / Background
Let's start with a real-world scenario in which you'd want the ability to change an actor's behavior.

### Real-World Scenario: Authentication
Imagine you're building a simple chat system using Akka.NET actors, and here's what your `UserActor` looks like - this is the actor that is responsible for all communication to and from a specific human user.

```csharp
public class UserActor : ReceiveActor {
	private readonly string _userId;
	private readonly string _chatRoomId;

	public UserActor(string userId, string chatRoomId) {
		_userId = userId;
		_chatRoomId = chatRoomId;
		Receive<IncomingMessage>(inc => inc.ChatRoomId == _chatRoomId,
			inc => {
				// print message for user
			});
		Receive<OutgoingMessage>(inc => inc.ChatRoomId == _chatRoomId,
			inc => {
				// send message to chatroom
			});
	}
}
```

So we have basic chat working - yay! But&hellip; right now there's nothing to guarantee that this user is who they say they are. This system needs some authentication.

How could we rewrite this actor to handle these same types of chat messages differently when:

* The user is **authenticating**
* The user is **authenticated**, or
* The user **couldn't authenticate**?

Simple: we can use switchable actor behaviors to do this!

### What is switchable behavior?
One of the core attributes of an actor in the [Actor Model](https://en.wikipedia.org/wiki/Actor_model) is that an actor can change its behavior between messages that it processes.

This capability allows you to do all sorts of cool stuff, like build [Finite State Machines](http://en.wikipedia.org/wiki/Finite-state_machine) or change how your actors handle messages based on other messages they've received.

Switchable behavior is one of the most powerful and fundamental capabilities of any true actor system. It's one of the key features enabling actor reusability, and helping you to do a massive amount of work with a very small code footprint.

How does switchable behavior work?

#### The Behavior Stack
Akka.NET actors have the concept of a "behavior stack". Whichever method sits at the top of the behavior stack defines the actor's current behavior. Currently, that behavior is `Authenticating()`:

![Initial Behavior Stack for UserActor](images/behaviorstack-initialization.png)

#### Use `Become` to adopt a new behavior
Whenever we call `Become`, we tell the `ReceiveActor` to push a new behavior onto the stack. This new behavior dictates which `Receive` methods will be used to process any messages delivered to an actor.

Here's what happens to the behavior stack when our example actor `Become`s `Authenticated`:

![Become Authenticated - push a new behavior onto the stack](images/behaviorstack-become.gif)

> NOTE: By default, `Become` will delete the old behavior off of the stack - so the stack will never have more than one behavior in it at a time. This is because most Akka.NET users don't use `Unbecome`.
>
> To preserve the previous behavior on the stack, call `Become(Method(), false)`

#### Use `Unbecome` to revert to old behavior
To make an actor revert to the previous behavior, all we have to do is call `Unbecome`.

Whenever we call `Unbecome`, we pop our current behavior off of the stack and replace it with the previous behavior from before (again, this new behavior will dictate which `Receive` methods are used to handle incoming messages).

Here's what happens to the behavior stack when our example actor `Unbecome`s:

![Unbecome - pop the current behavior off of the stack](images/behaviorstack-unbecome.gif)

### Isn't it problematic for actors to change behaviors?
No, actually it's safe and is a feature that gives your `ActorSystem` a ton of flexibility and code reuse.

Here are some common questions about switchable behavior:

#### When is the new behavior applied?
We can safely switch actor message-processing behavior because [Akka.NET actors only process one message at a time](http://petabridge.com/blog/akkadotnet-async-actors-using-pipeto/). The new message processing behavior won't be applied until the next message arrives.

#### How deep can the behavior stack go?
The stack can go *really* deep, but it's not unlimited.

Also, each time your actor restarts, the behavior stack is cleared and the actor starts from the initial behavior you've coded.

#### What happens if you call `Unbecome` and with nothing left in the behavior stack?
The answer is: *nothing* - `Unbecome` is a safe method and won't do anything unless there's more than one behavior in the stack.


### Back to the real-world example
Okay, now that you understand switchable behavior, let's return to our real-world scenario and see how it is used. Recall that we need to add authentication to our chat system actor.

So, how could we rewrite this actor to handle chat messages differently when:

* The user is **authenticating**
* The user is **authenticated**, or
* The user **couldn't authenticate**?

Here's one way we can implement switchable message behavior in our `UserActor` to handle basic authentication:

```csharp
public class UserActor : ReceiveActor {
	private readonly string _userId;
	private readonly string _chatRoomId;

	public UserActor(string userId, string chatRoomId) {
		_userId = userId;
		_chatRoomId = chatRoomId;

		// start with the Authenticating behavior
		Authenticating();
	}

	protected override void PreStart() {
		// start the authentication process for this user
		Context.ActorSelection("/user/authenticator/")
			.Tell(new AuthenticatePlease(_userId));
	}

	private void Authenticating() {
		Receive<AuthenticationSuccess>(auth => {
			Become(Authenticated); //switch behavior to Authenticated
		});
		Receive<AuthenticationFailure>(auth => {
			Become(Unauthenticated); //switch behavior to Unauthenticated
		});
		Receive<IncomingMessage>(inc => inc.ChatRoomId == _chatRoomId,
			inc => {
				// can't accept message yet - not auth'd
			});
		Receive<OutgoingMessage>(inc => inc.ChatRoomId == _chatRoomId,
			inc => {
				// can't send message yet - not auth'd
			});
	}

	private void Unauthenticated() {
		//switch to Authenticating
		Receive<RetryAuthentication>(retry => Become(Authenticating));
		Receive<IncomingMessage>(inc => inc.ChatRoomId == _chatRoomId,
			inc => {
				// have to reject message - auth failed
			});
		Receive<OutgoingMessage>(inc => inc.ChatRoomId == _chatRoomId,
			inc => {
				// have to reject message - auth failed
			});
	}

	private void Authenticated() {
		Receive<IncomingMessage>(inc => inc.ChatRoomId == _chatRoomId,
			inc => {
				// print message for user
			});
		Receive<OutgoingMessage>(inc => inc.ChatRoomId == _chatRoomId,
			inc => {
				// send message to chatroom
			});
	}
}
```

Whoa! What's all this stuff? Let's review it.

First, we took the `Receive<T>` handlers defined on our `ReceiveActor` and moved them into three separate methods. Each of these methods represents a state that will control how the actor processes messages:

* `Authenticating()`: this behavior is used to process messages when the user is attempting to authenticate (initial behavior).
* `Authenticated()`: this behavior is used to process messages when the authentication operation is successful; and,
* `Unauthenticated()`: this behavior is used to process messages when the authentication operation fails.

We called `Authenticating()` from the constructor, so our actor began in the `Authenticating()` state.

*This means that only the `Receive<T>` handlers defined in the `Authenticating()` method will be used to process messages (initially)*.

However, if we receive a message of type `AuthenticationSuccess` or `AuthenticationFailure`, we use the `Become` method ([docs](http://getakka.net/wiki/ReceiveActor#become "Akka.NET - ReceiveActor Become")) to switch behaviors to either `Authenticated` or `Unauthenticated`, respectively.

### Can I switch behaviors in an `UntypedActor`?
Yes, but the syntax is a little different inside an `UntypedActor`. To switch behaviors in an `UntypedActor`, you have to access `Become` and `Unbecome` via the `ActorContext`, instead of calling them directly.

These are the API calls inside an `UntypedActor`:

* `Context.Become(Receive rec, bool discardPrevious = true)` - pushes a new behavior on the stack or
* `Context.Unbecome()` - pops the current behavior and switches to the previous (if applicable.)

The first argument to `Context.Become` is a `Receive` delegate, which is really any method with the following signature:

```csharp
void MethodName(object someParameterName);
```

This delegate is just used to represent another method in the actor that receives a message and represents the new behavior state.

Here's an example (`OtherBehavior` is the `Receive` delegate):

```csharp
public class MyActor : UntypedActor {
	protected override void OnReceive(object message) {
		if(message is SwitchMe) {
			// preserve the previous behavior on the stack
			Context.Become(OtherBehavior, false);
		}
	}

	// OtherBehavior is a Receive delegate
	private void OtherBehavior(object message) {
		if(message is SwitchMeBack) {
			// switch back to previous behavior on the stack
			Context.Unbecome();
		}
	}
}
```


Aside from those syntactical differences, behavior switching works exactly the same way across both `UntypedActor` and `ReceiveActor`.

Now, let's put behavior switching to work for us!

## Exercise
In this lesson we're going to add the ability to pause and resume live updates to the `ChartingActor` via switchable actor behaviors.

### Phase 1 - Add a New `Pause / Resume` Button to `Main.cs`
This is the last button you'll have to add, we promise.

Go to the **[Design]** view of `Main.cs` and add a new button with the following text: `PAUSE ||`

![Add a Pause / Resume Button to Main](images/design-pauseresume-button.png)

Got to the **Properties** window in Visual Studio and change the name of this button to `btnPauseResume`.

![Use the Properties window to rename the button to btnPauseResume](images/pauseresume-properties.png)

Double click on the `btnPauseResume` to add a click handler to `Main.cs`.

```csharp
private void btnPauseResume_Click(object sender, EventArgs e)
{

}
```

We'll fill this click handler in shortly.

### Phase 2 - Add Switchable Behavior to `ChartingActor`
We're going to add some dynamic behavior to the `ChartingActor` - but first we need to do a little cleanup.

First, add a `using` reference for the Windows Forms namespace at the top of `Actors/ChartingActor.cs`.

```csharp
// Actors/ChartingActor.cs

using System.Windows.Forms;
```

Next we need to declare a new message type inside the `Messages` region of `ChartingActor`.

```csharp
// Actors/ChartingActor.cs - add inside the Messages region
/// <summary>
/// Toggles the pausing between charts
/// </summary>
public class TogglePause { }
```

Next, add the following field declaration just above the `ChartingActor` constructor declarations:

```csharp
// Actors/ChartingActor.cs - just above ChartingActor's constructors

private readonly Button _pauseButton;
```

Move all of the `Receive<T>` declarations from `ChartingActor`'s main constructor into a new method called `Charting()`.

```csharp
// Actors/ChartingActor.cs - just after ChartingActor's constructors
private void Charting()
{
    Receive<InitializeChart>(ic => HandleInitialize(ic));
    Receive<AddSeries>(addSeries => HandleAddSeries(addSeries));
    Receive<RemoveSeries>(removeSeries => HandleRemoveSeries(removeSeries));
    Receive<Metric>(metric => HandleMetrics(metric));

	//new receive handler for the TogglePause message type
    Receive<TogglePause>(pause =>
    {
        SetPauseButtonText(true);
        Become(Paused, false);
    });
}
```

Add a new method called `HandleMetricsPaused` to the `ChartingActor`'s `Individual Message Type Handlers` region.

```csharp
// Actors/ChartingActor.cs - inside Individual Message Type Handlers region
private void HandleMetricsPaused(Metric metric)
{
    if (!string.IsNullOrEmpty(metric.Series) && _seriesIndex.ContainsKey(metric.Series))
    {
        var series = _seriesIndex[metric.Series];
        series.Points.AddXY(xPosCounter++, 0.0d); //set the Y value to zero when we're paused
        while (series.Points.Count > MaxPoints) series.Points.RemoveAt(0);
        SetChartBoundaries();
    }
}
```

Define a new method called `SetPauseButtonText` at the *very* bottom of the `ChartingActor` class:

```csharp
// Actors/ChartingActor.cs - add to the very bottom of the ChartingActor class
private void SetPauseButtonText(bool paused)
    {
        _pauseButton.Text = string.Format("{0}", !paused ? "PAUSE ||" : "RESUME ->");
    }
```

Add a new method called `Paused` just after the `Charting` method inside `ChartingActor`:

```csharp
// Actors/ChartingActor.cs - just after the Charting method
private void Paused()
{
    Receive<Metric>(metric => HandleMetricsPaused(metric));
    Receive<TogglePause>(pause =>
    {
        SetPauseButtonText(false);
        Unbecome();
    });
}
```

And finally, let's **replace both of `ChartingActor`'s constructors**:

```csharp
public ChartingActor(Chart chart, Button pauseButton) : this(chart, new Dictionary<string, Series>(), pauseButton)
{
}

public ChartingActor(Chart chart, Dictionary<string, Series> seriesIndex, Button pauseButton)
{
    _chart = chart;
    _seriesIndex = seriesIndex;
    _pauseButton = pauseButton;
    Charting();
}

private void Charting()
{
    Receive<InitializeChart>(ic => HandleInitialize(ic));
    Receive<AddSeries>(addSeries => HandleAddSeries(addSeries));
    Receive<RemoveSeries>(removeSeries => HandleRemoveSeries(removeSeries));
    Receive<Metric>(metric => HandleMetrics(metric));
    Receive<TogglePause>(pause =>
    {
        SetPauseButtonText(true);
        Become(Paused, false);
    });
}
```

### Phase 3 - Update the `Main_Load` and `Pause / Resume` Click Handler in Main.cs
Since we changed the constructor arguments for `ChartingActor` in Phase 2, we need to fix this inside our `Main_Load` event handler.

```csharp
//Main.cs - Main_Load event handler
_chartActor = Program.ChartActors.ActorOf(Props.Create(() => new ChartingActor(sysChart, btnPauseResume)), "charting");
```

And finally, we need to update our `btnPauseResume` click event handler to have it tell the `ChartingActor` to pause or resume live updates:

```csharp
//Main.cs - btnPauseResume click handler
private void btnPauseResume_Click(object sender, EventArgs e)
{
    _chartActor.Tell(new ChartingActor.TogglePause());
}
```

### Once you're done
Build and run `SystemCharting.sln` and you should see the following:

![Successful Lesson 4 Output](images/dothis-successful-run4.gif)

Compare your code to the code in the [/Completed/ folder](Completed/) to compare your final output to what the instructors produced.

## Great job!
YEAAAAAAAAAAAAAH! We have a live updating chart that we can pause over time!

But wait a minute, what happens if I toggle a chart on or off when the `ChartingActor` is in a paused state?

![Lesson 4 Output Bugs](images/dothis-fail4.gif)

### DOH!!!!!! It doesn't work!

*This is exactly the problem we're going to solve in the next lesson*, using a message `Stash` to defer processing of messages until we're ready.

**Let's move onto [Lesson 5 - Using `Stash` to Defer Processing of Messages](../lesson5).**
