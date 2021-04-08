# Thoughts on SwiftUI navigation

## Introduction

I am using SwiftUI & ComposableArchitecture for a while already. I had built several iOS & macOS apps with it, but there is one topic that I am still struggling with - **navigation**.

I want to share my thoughts with you, show the issues I experienced and how I tried to address them. Ultimately, I would like to develop a flexible solution that can be used to implement complex navigation flows. It should be simple to use and provides a native user experience baked by native SwiftUI APIs.

## Case study

Although most of the apps will contain rather complex navigation flows, let's start with a simplified example.

Consider the following use case: We need to build a basic iOS app with three screens. A user should be able to navigate from the first screen to the second and then to the third one. There should be an option to go back from the third screen directly to the first one. We should be able to control navigation programmatically. The app should feel native. The UI and gestures should be familiar to the user.

This could be fairly easy to implement in UIKit, with a `UINavigationController` controlled imperatively. You can push, pop, or even change the whole stack of view controllers on it, and all happens with a nice animation. However, things look a bit different in SwiftUI's declarative world. We have a `NavigationLink` that can be controlled by `Binding<Bool>`. We will use it, as there are no other options at the moment (besides wrapping `UINavigationController` using `UIViewControllerRepresentable`, which complicates the implementation and bounds it to the iOS platform - let's avoid this road for now).

## Basic implementation

Without any further ado, here is how we can implement it using ComposableArchitecture:

1. Define a state, actions, and reducer for each of the screen views:

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

3. Drive `NavigationLink` with an optional state of the view that we are going to present:

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

## Dismissing screen and canceling its side effects

As soon as we start adding more logic to the reducers, including long-running effects, we will discover that there is no easy way to cancel them when the screen is dismissed. We just naively set the state of the presented view to `nil` to dismiss it. If side effects are running, an action can be dispatched when the state is already `nil`.

### Solution #1

**TODO:** describe manual effects cancellation when dismissing screens.

### Solution #2

**TODO:** describe using `Reducer.presents` extension found on ComposableArchitecture `iso` branch.

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

In other to make it work, we need to add an action to the second screen:

```swift
enum SecondAction {
  // ...
  case didDisappear
}
```

And update the first screen reducer:

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

Then we are able to create `NavigationLink` like this:

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

I've used this approach in several apps so far. It does work as expected. Unfortunately, it adds a lot of complexity to the code. **TODO:** describe the introduced complexity.

### Solution #3

**TODO:** describe storing the last non-`nil` state and using it with the destination

## Actual implementation example

**TODO:** link and describe example project

## Summary

**TODO:** sum up thoughts on navigation
