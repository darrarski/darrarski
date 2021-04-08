# Thoughts on SwiftUI navigation

...in a [ComposableArchitecture](http://github.com/pointfreeco/swift-composable-architecture) world.

## Introduction

I am using SwiftUI & [ComposableArchitecture](http://github.com/pointfreeco/swift-composable-architecture) for a while already. I had built several iOS & macOS apps with it, but there is one topic that I am still struggling with - **NAVIGATION**.

I want to share my thoughts with you, show the issues I experienced and how I tried to address them. Ultimately, I would like to develop a flexible solution that can be used to implement complex navigation flows. It should be simple to use and provides a native user experience baked by native SwiftUI APIs.

## Case study

Although most of the apps will contain rather complex navigation flows, let's start with a simplified example.

Consider the following use case: We need to build a basic iOS app with three screens. A user should be able to navigate from the first screen to the second and then to the third one. There should be an option to go back from the third screen directly to the first one. We should be able to control navigation programmatically. The app should feel native. The UI and gestures should be familiar to the user.

This could be fairly easy to implement in UIKit, with a `UINavigationController` controlled imperatively. You can push, pop, or even change the whole stack of view controllers on it, and all happens with a nice animation. However, things look a bit different in SwiftUI's declarative world. We have a `NavigationLink` that can be controlled by `Binding<Bool>`. We will use it, as there are no other options at the moment (besides wrapping `UINavigationController` using `UIViewControllerRepresentable`, which complicates the implementation and bounds it to the iOS platform - let's avoid this road for now).

## Basic implementation

Without any further ado, here is how we can implement it using ComposableArchitecture:

1. Define a state, actions, and reducer for each of the screen views.

    ```swift
    struct FirstState {
      var second: SecondState? // state of the second screen
    }
    
    enum FirstAction {
      case presentSecond(Bool) // present or dismiss second screen
      case second(SecondAction) // action of the second screen
    }
    
    let firstReducer = Reducer<FirstState, FirstAction, Void> { state, action, _ in
      switch action {
        case let .presentSecond(present):
          state.second = present ? SecondState() : nil
          return .none

        case .second:
          return .none
      }
    }
    ```

2. Combine reducers using `.optional` and `.pullback` operators.

    ```swift
    let firstReducer = Reducer<FirstState, FirstAction, Void>.combine(
      secondReducer.optional().pullback(
        state: \.second,
        action: /FirstAction.second,
        environment: { _ in () }
      ),
      Reducer {
        // reducer implementation from step 1
      }
    )
    ```

3. Drive `NavigationLink` with an optional state of the view that we will present.

    ```swift
    NavigationLink(
      destination: IfLetStore(
        store.scope(
          state: \.second,
          action: FirstAction.second
        ),
        then: SecondView.init(store:)
      ),
      isActive: viewStore.binding(
        get: { $0.second != nil },
        send: FirstAction.presentSecond
      ),
      label: { 
        Text("Present second screen")
      }
    )
    ```

So far, it doesn't look too complex, and it works. However, there are already several issues with this implementation.

## Programmatically dismissed screens does not animate correctly

We can dismiss the screen by tapping on a back button or performing a swipe-from-screen-edge gesture. In this case, everything looks good. However, if we try to dismiss it programmatically by dispatching an action, things look weird. The dismissed screen disappears before pop animation ends.

This glitch appears because of the way we implemented `NavigationLink`s. We drive it using `Binding<Bool>` that gets its value from `state.second != nil`. When dismiss action is dispatched to the store, we immediately set the state to `nil`, which triggers dismiss animation. At the same time, the destination of `NavigationLink` changes to an empty view, thanks to `IfLetStore`. In this case, we can't actually present the dismissed view because we don't have its state anymore. There are several workarounds for this issue.

### Solution #1

[Majid Jabrayilov suggested in his blog post](https://swiftwithmajid.com/2021/01/27/lazy-navigation-in-swiftui/) to wrap the whole `NavigationLink` in a conditional statement. With this approach, if there is no state for the presented view, we won't render `NavigationLink` at all. Although his approach does not use ComposableArchitecure, it could be easily adjusted to work with `Store` that has an optional `State`:

```swift
extension View {
  func navigate<State, Action, Destination: View>(
    using store: Store<State?, Action>,
    onDismiss: @escaping () -> Void,
    destination: @escaping (Store<State, Action>) -> Destination
  ) -> some View {
    background(
      IfLetStore(
        store,
        then: { destinationStore in
          NavigationLink(
            destination: destination(destinationStore),
            isActive: Binding(
              get: { true },
              set: { isActive in
                if !isActive {
                  onDismiss()
                }
              }
            ),
            label: EmptyView.init
          )
        }
      )
    )
  }
}
```

The code above looks a bit strange, especially the part in which we are hardcoding `isActive` binding getter to `true`. This works because if there is no state to present, the `IfLetStore` will not render the output of the `then` closure, so there will be no `NavigationLink` in the view hierarchy at all.

At first sight, this solution solves our issue, as programmatically dismissed screens animate correctly. Unfortunately, if we apply `StackNavigationViewStyle` to our `NavigationView` like this:

```swift
NavigationView {
  // ...
}
.navigationViewStyle(StackNavigationViewStyle())
```

Things look even worse as push transitions do not animate anymore. Whenever we try to navigate to the next screen, it will just appear immediately without animation. I've created a [gist with code that reproduces the issue](https://gist.github.com/darrarski/22edfc682dc918f6f6cc5e441429c6cf) and reported it to Apple. I'm not sure if it's a bug, or perhaps we should not hardcode the `isActive` binding as we did in this case.

### Solution #2

Another solution that addresses this problem is to decouple the state that drives `NavigationLink` `isActive` binding from the state of view we are want to present:

```swift
struct FirstState {
  var isPresentingSecond: Bool // drives NavigationLink isActive binding
  var second: SecondState?
}
```

To make it work, we need to add an action to the second screen, which we will dispatch when the presented view disappears. 

```swift
enum SecondAction {
  // ...
  case didDisappear // dispatched when presented view disappears using `.onDisappear` SwiftUI modifier
}
```

We will handle the action in the presenter's reducer. Notice that `.presentSecond(false)` action does not remove the presented state anymore. It just toggles `isPresentingSecond` flag to `false`. This flag drives `NavigationLink` and triggers dismission. When the presented screen disappears and `isPresentingSecond` is `false`, it means that the view was dismissed, and its state can be set to `nil`. 

```swift
Reducer<FirstState, FirstAction, Void> { state, action, _ in
  switch action {
  // ...
  case .presentSecond(present):
    state.isPresentingSecond = present
    if present {
      state.second = SecondState()
    }
    return .none

  case .second(.didDisappear):
    if state.isPresentingSecond == false {
      state.second = nil
    }
    return .none
  }
}
```

With the above changes, we can declare `NavigationLink` like this:

```swift
NavigationLink(
  destination: IfLetStore(
    store.scope(
      state: \.second,
      action: FirstAction.second
    ),
    then: SecondView.init(store:)
  ),
  isActive: viewStore.binding(
    get: \.isPresentingSecond,
    send: FirstAction.presentSecond
  ),
  label: {
    Text("Present Second")
  }
)
```

I've used this approach in several apps so far. It does work as expected. Unfortunately, it adds a lot of complexity to the code. Because of decoupling the state that drives the `NavigationLink` (`isPresentingSecond` property in the above example) from the presented state (optional `second` property), we added a lot of boilerplate to the reducers. It's also required to add more code to the reducer if we need to handle dismissing multiple screens at once. There is actually a lot of things that we should care about, which makes the solution rather complex.

### Solution #3

The main problem of the solution mentioned above is the significantly increased complexity of the code that drives navigation. With a fairly simple navigation flow, we needed to add a lot of boilerplate just to address the glitch with animations when programmatically dismissing screens. Thankfully there is a simpler way to achieve the same result. 

We can create a wrapper for the `NavigationLink` that we are using with the ComposableArchitecture's `Store`. The optional state property will drive it without the need to decouple it. To make the destination view stay in the view hierarchy after the state is set to `nil`, we will store the last non-`nil` state and use it instead. This involves creating additional property in the view so we don't pollute our state structs.

```swift
struct NavigationLinkView<State, Action, Destination: View>: View {
  init(
    store: Store<State?, Action>,
    onDismiss: @escaping () -> Void,
    destination: @escaping (Store<State, Action>) -> Destination
  ) {
    self.store = store
    self.onDismiss = onDismiss
    self.destination = destination
    self.viewStore = ViewStore(
      store,
      removeDuplicates: { ($0 != nil) == ($1 != nil) }
    )
  }

  let store: Store<State?, Action>
  let onDismiss: () -> Void
  let destination: (Store<State, Action>) -> Destination
  @ObservedObject var viewStore: ViewStore<State?, Action>
  @SwiftUI.State var lastState: State?

  var body: some View {
    NavigationLink(
      destination: IfLetStore(
        store.scope(state: { $0 ?? lastState }),
        then: destination
      ),
      isActive: Binding(
        get: { viewStore.state != nil },
        set: { isActive in
          if isActive == false {
            onDismiss()
          }
        }
      ),
      label: EmptyView.init
    )
    .onReceive(viewStore.publisher) { state in
      if let state = state {
        lastState = state
      }
    }
  }
}
```

It might look complex, but it's actually rather simple. We can use it like this:

```swift
NavigationLinkView(
  store: store.scope(
    state: \.second,
    action: FirstAction.second
  ),
  onDismiss: {
    viewStore.send(.presentSecond(false))
  },
  destination: SecondView.init(store:)
)
```

It does not require modifying state or reducers showcased in the "Basic implementation" section. It allows for programmatic navigation - both presenting and dismissing screens is possible by simple state mutations inside the reducer. Because we use the last non-`nil` state value when dismissing, there is no glitch. It also works when using `StackNavigationViewStyle` without an issue.

## Dismissing screen and canceling its side effects

As soon as we start adding more logic to the reducers, including long-running effects, we will discover that there is no easy way to cancel them when the screen is dismissed. We just naively set the state of the presented view to `nil` to dismiss it. If side effects are running, an action can be dispatched when the state is already `nil`.

### Solution #1

One way to address this problem would be to cancel all long-running effects when we dismiss screens manually. Because the navigation is state-driven and state mutations happen only inside the reducers, we are in control here.

```swift
Reducer<FirstState, FirstAction, Void> { state, action, _ in
  switch action {
  // ...
  case let .presentSecond(present):
    state.second = present ? SecondState() : nil
    if present == false {
      return .cancel(id: SecondReducerEffectId())
    }
    return .none
  // ...
  }
}
```

This should work as expected. However, it's not a very clean solution. These effects are created in the presented view's reducer, but we have to remember to cancel them in the reducer that presents the view. 

### Solution #2

Another solution comes from the authors of ComposableArchitecture, although it's not (yet) an official one. On a separate branch, there is [an extension to the `Reducer`](https://github.com/pointfreeco/swift-composable-architecture/blob/9ec4b71e5a84f448dedb063a21673e4696ce135f/Sources/ComposableArchitecture/Reducer.swift#L549-L572) that is meant to be used to solve our problem. Instead of combining reducers with `.optional` and `.pullback` operators as proposed earlier, we can use `.presents` operator, like this:

```swift
let firstReducer = Reducer<FirstState, FirstAction, Void> { state, action, _ in
  // basic reducer implementation
}
.presents(
  secondReducer,
  cancelEffectsOnDismiss: true,
  state: \.second,
  action: /FirstAction.second,
  environment: { _ in () }
)
```

It even looks better and is easier to read. I played a bit with this solution, and after a slight modification works well also when multiple views are dismissed at once. I found no issues so far. Here is the code:

```swift
extension Reducer {
  func presents<LocalState, LocalAction, LocalEnvironment>(
    _ localReducer: Reducer<LocalState, LocalAction, LocalEnvironment>,
    cancelEffectsOnDismiss: Bool,
    state toLocalState: WritableKeyPath<State, LocalState?>,
    action toLocalAction: CasePath<Action, LocalAction>,
    environment toLocalEnvironment: @escaping (Environment) -> LocalEnvironment
  ) -> Self {
    let id = UUID()
    return Self { state, action, environment in
      let hadLocalState = state[keyPath: toLocalState] != nil
      let localEffects: Effect<Action, Never>
      if hadLocalState {
        localEffects = localReducer
          .optional()
          .pullback(state: toLocalState, action: toLocalAction, environment: toLocalEnvironment)
          .run(&state, action, environment)
          .cancellable(id: id)
      } else {
        localEffects = .none
      }
      let globalEffects = self.run(&state, action, environment)
      let hasLocalState = state[keyPath: toLocalState] != nil
      return .merge(
        localEffects,
        globalEffects,
        cancelEffectsOnDismiss && hadLocalState && !hasLocalState ? .cancel(id: id) : .none
      )
    }
  }
}
```

## Actual implementation example

I've created an [example project](https://github.com/darrarski/tca-swiftui-navigation-demo) to explore how `NavigationLink` can be used with ComposableArchitecure and how the issues mentioned above can be addressed. I tested several approaches and solutions mentioned above. Feel free to check out commits history to follow my journey.

|Navigation in demo app|
|:-:|
|[![navigation in demo app](https://github.com/darrarski/tca-swiftui-navigation-demo/raw/8b48f3aa2ecf386d74a770f997102f76c2a4e5ee/Demo.gif)](https://github.com/darrarski/tca-swiftui-navigation-demo/raw/8b48f3aa2ecf386d74a770f997102f76c2a4e5ee/Demo.mp4)|

## Final thoughts

There is much more to explore when it comes to navigation in SwiftUI apps. I only focused on an elementary example, which can be used in real applications. It's unfortunately far away from an ultimate solution that allows implementing complex navigation flows. Due to current SwiftUI limitations, we don't have many options in this field compared to what we can do using plain, old UIKit.

`NavigationLink` has not only limited usage capability but also can easily cause a lot of problems. It's definitely not an equivalent of `UINavigationController` despite the fact that it uses it under the hood. Its declarative nature looks nice at first sight but can be a reason for headache when trying to implement a fairly simple flow, like pop-to-root, which in imperative UIKit would be a no-brainer.

The biggest pain point for me is actually the lack of a declaratively-manageable navigation stack in SwiftUI. While UIKit provides `UINavigationController` with `setViewControllers(_:animated:)` function, there is no equivalent of it in a SwiftUI world. I'm missing a declarative API that allows managing a stack of views, so we can easily update it. I would love to see it at the next WWDC.

### Do you want to leave a comment?

Check out [dedicated discussion](https://github.com/darrarski/darrarski/discussions/3).

### Did you like it?

<a href="https://www.buymeacoffee.com/darrarski" target="_blank"><img src="https://cdn.buymeacoffee.com/buttons/v2/default-yellow.png" alt="Buy Me A Coffee" height="60" width="217" style="height: 60px !important;width: 217px !important;" ></a>
