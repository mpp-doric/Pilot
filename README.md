#Pilot

![Pilot Mascot](https://raw.githubusercontent.com/doridori/Pilot/master/gfx/pilot_mascot.png)

An abstract application stack which facilitates:

- Presenter lifecycle management
- Presenter -> Presenter control flow
- Fine grained data scope management
- Presenter lifecycle events

**If you have not already I highly recommened you read the `Motivations` post found [here]() as they lay a foundation for the solutions presented in this README.**

This README will have the following structure:

1. Outline the core library components
2. Outline the general concepts
3. How to use!
4. Highlight what still needs to be worked on
5. Mention any current known limitations


#Intro

**Pilot** is an effort to abstract a simple Application Stack from the common approaches to Controller/Presenter (from now on I will just use the word controller to cover all approaches here) based Android application architecture. An aim is to not tie any implementations to any specifc controller type or 3rd party dependencies i.e. to be as flexible as possible.

You may read the below and think 'thats what `FragmentManager` is for' or 'I use the Activity backstack for that' and if you find those solutions are working for you thats great. I find that there is often cases where these two api concepts either over-complicate or restrict the things I need to do and feel there is a good case for some applications to use an abstracted stack instead.

#Core Components

##The `PilotStack`

The `PilotStack` is the main core of this mini library which has the following properties:

- Holds an internal collection of `PilotFrame` objects that can be manipulated in various ways (Push, Pop, Clear at frame, clear all).
- Stack events can be listened to
- Implements `Serializable`

##The `PilotFrame`

- Has lifecycle methods
- Holds reference to parent stack
- Implements `Serializable`

**The `PilotFrame` can also be a controller itself or hold a reference to one.**

##The `@InvisibleFrame`

Annotation that can be applied to a `PilotFrame` subclass which signifies that it should be ignored for all stack callback operations. Very useful for handling scoped data within the stack

#General Concepts

##Single-Activity Application

I find a lot of the applications I have build in the last couple of years sit inside a single `Activity` and just manipulate `Fragments` and/or `View`. You can use a single `Activity` which holds a single `PilotStack` for the simplest use case. This however does not mean you cant jump out to seperate activitys, for example when integrating with 3rd party libs which require this.

Advantage: Simple

##Presenter-to-Presenter control flow

As mentioned in the `Motivations` post above this kind of control flow (which bypasses `android.*` classes completly has its advantages) and is made very simple with Pilot. An example below

```java
public class FirstViewPresenter extends PilotFrame
{
    ...
    public void somethingHappened()
    {        
        getParentStack().pushFrame(new SecondViewPresenter());
    }
}
```

Advantage: This means for each and quick JVM testing and any interested renderer could just listen out for `PilotStack` changes and work out how to render them. You could imagine it would be trivial to replace the whole interface on an application with a command line and you would not have to change a line of controller logic (ok so you may never do this but it highlights a good degree of decoupling!).

##Presenter lifecycle management

As is the root goal of most frameworks that interact with controller lifecycle management is an important point. Using the 'PilotStack' the 'PilotFrames' (and therefore any controllers they may be composed of and provide) live just as long as they need to without worrying about config-changes _or_ the lifecycle of any attached view/activity component. This happens for **free** as all app navigation happens via the `PilotStack` in the first place. For example see below

![Presenter Scope Example Diagram](https://raw.githubusercontent.com/doridori/Pilot/master/gfx/presenter_scope_example.png)

This may seem overly simple - but in my mind thats a good thing! There are many approaches to this singular issue that can easily get over-complicated when controllers and liveness are tied too much into an Activity or view components particular lifecycle.

##How Presenters are attached to views.

This is an important question as it often instantly highlights any usecase issues when using a library with similar concerns to Pilot.

This is probably most easily illustrated with a Sequence diagram. The main idea is that when a PilotFrame hits the top of the PilotStack a callback is fired, which the PilotActivity uses to create and add the new View to the defined ViewGroup. The PilotActivity knows which View takes which PilotFrame based upon the `@Presenter` annotation. `PilotActivity.init()` handles the state on Activity creation as the sequence diagram below shows - PilotStack updates then just fire the `topVisibleFrameUpdate()` method and exercise the code in the second half of the sequence.

![Init sequence diagram](https://raw.githubusercontent.com/doridori/Pilot/master/gfx/init_sequence.png)

###What about after a config-change?

After a config-change the PilotStack has been preserved and the init() call will trigger again - which will auto show the view at the top of the PilotStack. This is the same as first Activity creation apart from ignoring the `freshStart` conditional.

###What about re-inflated Views that are now presenter-less?

These will either have their Presenter pushed back to them via `init()` or will be removed if they represent a PilotFrame that is not longer top of the PilotStack.

###What about child views that also have Presenters?!

There are many forms these can take but a simple way to handle these are to maintain a collection of child-Presenter objects inside the PilotFrame and push / pull them out depending on their own individual lifecycle. 

For example you may have a RecyclerView which contains child Views which are each backed by their own Presenter. These could be pulled from the main PilotFrame backed Presenter by some meta data i.e. id.

##Scoped Data

This is an interesting one. As mentioned in the `Motivations` post linked above, scoped data will make people think of differnt implementations depending on if any DI frameworks (think Dagger) are in use. I will restate here that this solution is not to replace Dagger (and also does not require it) but is a way of (optionally) handling data scopes (and the method by which the scoped data could be added and removed from any DI frameworks at runtime).

The notion is very simple really. Is it that there is the idea of a PilotFrame that can just represent data only (as opposed to the default, which is a PilotFrame that represents an app-state which is typically used to back a view-state). Operations can be performed upon these data-frames (e.g. clear this data-frame and all above it) just like any other frame. The one difference is that data-frames do **not** get passed through any `PilotStack` `Listeners`. These data-frames are currently signified by applying the `@InvisibleFrame` annotation to a `PilotFrame` subclass.

An example may help here. A common kind of scoped data in most apps is some kind of `Session` data (say data that exists while a user is logged in).

![Data Scope Example Diagram](https://raw.githubusercontent.com/doridori/Pilot/master/gfx/data_scope_example.png)

In the above diagram we show the SessionDataFrame being popped from the stack along with any in-session frames above (in this example just the one). This could be from user log-out or session timeout (timeout counter could live in the SessionDataFrame itself) and would auto fire a `PilotStack` `Listener` callback containing the top `PilotFrame` in the stack, which may be a login screen in this case.

Inside the SessionDataFrame the data can be made available to the stack by either
- If using Dagger the PilotFrame lifecycle methods (push/pop) couple optionally add / remove dependencies from any Dagger ObjectGraph (i.e. some LoggedInUser object)
- If not using Dagger (as per the current repo example) the SessionData can be pulled from the PilotStack directly (see below code snippit)

```java
public class SecondPilotFrame extends PilotFrame
{
    ...
    public String getSessionDataToDisplay()
    {
        SessionDataFrame sessionScopedData = getParentStack().getFrameOfType(SessionDataFrame.class);
        return sessionScopedData.getSomeSessionData();
    }
}
```

This has benefits of scoped data being able to handle its own lifecycle and also being cleared by default when the stack is cleared but also if clearing the stack on `Activity.finish()` you get all your sensitive data cleaned up for free. Happy days.

##Controller Lifecycle Events

WIP see [Issue](https://github.com/doridori/Pilot/issues/2) representing this task.

##Callbacks

One advantage of using presenter-to-presenter comms is that we can just pass operation callbacks directly. This is a lot simpler that using things like `startActivityForResult()` and having to convert your operations to ints and bundles. You can just pass a callback class / anon class / closure (if using retroLambda) directly to the instantianted Presenter.

"But what about PilotStack state persistence and callbacks?" I hear you ask. Yes this is a common issue when passing callback objects to `Fragment` and `View` implementations as on rotation these view objects are reinflated and your callbacks are lost resulting in a fun NPE. As the PilotStack is persisted on config change we dont have to worry about this. Regardless callbacks being retained on process death, this is quite interesting. If we are Serializing the stack (as talked about elsewhere in this readme) and we have a callback like

```java
	getParentStack().pushFrame(new SecondViewPresenter(new SomeCallback
	{
		public void doSomething()
		{
			//doing something!
	    }
	}
	}));
```

This relationship will be preserved on Deserialization! This however does require callbacks are Serializable. Alpternativly you could fall back to using a version of result codes and just pass primitives around to trigger callback behaviour.

##Handling Process Death

Some times you want your apps state to survive process death. This can be done by serializing the PilotStack in the SavedState `Bundle`. This happens for free inside the `PilotActivity` class.

Some may complain that serialization is slow and is not ideal. You are right! A pending improvement is to use Parcelable or @AutoParcel in place of Serialization. See [related issue](https://github.com/doridori/Pilot/issues/7).

#How To Use

##Extend PilotActivity

Fundamentally you can extend `PilotActivity` as this handles:

- Initiating / saving / restoring the current PilotStack
- Passing `PilotFrames` to the correct Views

and your subclass needs to:

- call `PilotActivity.init(FrameLayout rootView)` with the parent view of any `PilotFrame` backed View that will be shown
- Override `protected Class<? extends PresenterBasedFrameLayout>[] getRootViewClasses()` and pass back an array of Views which should be handled automatically when their corresponding `PilotFrames` hit the top of the 'PilotStack'.
- Override `protected abstract PilotFrame getLaunchPresenterFrame();` and return the starting `PilotFrame` for the app

and the subclass can optionally:

- Override `boolean interceptTopFrameUpdatedForCategory(PilotFrame topVisibleFrame, Direction direction)` for any `PilotFrames` that are not represented in the `getRootViewClasses()` array above and perform custom rendering logic i.e. show a DialogFragment / switch Activity etc.

##Create some `PilotFrame` subclasses

These represent the individual states and data-scopes of your application and are the things that will live on the stack.

##Example App

See the simple example app for an idea how this stuff works in practise.

#WIP

Please see the projects Issues list for what still needs to be worked on. The main thing at present is the PilotFrame lifecycle callbacks and will do this ASAP. Interested to hear if any input on the thoughts I have had on it.

##Limitations

There are some things that I never really end up using in the 5-6 years I have professionally building Android apps. These include:

- 2 pane tablet layouts
- Tasks

For this reason I have not spent much time thinking about how to work with them. That said they are represented on the Issues list and I do have ideas how these can work with Pilot. Personally I will wait till I need them or they are requested before spending many thought-cycles on them.

##F.A.Q.

No one has asked me any questions about this as yet but if they do I will create an F.A.Q. :)


