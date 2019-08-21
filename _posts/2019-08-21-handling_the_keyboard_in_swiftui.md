---
layout:     post
title:      Handling the Keyboard in SwiftUI
date:       2019-08-21
summary:
categories: iOS SwiftUI Combine
---

Adjusting your UI to avoid the keyboard is a common task in iOS development. Whether you're building a form and want to keep the current field visible, or you're building a chat client and want to adjust your text-entry component to always sit on top of the keyboard, you will have to handle the keyboard animating on and off of the screen.

As I started to explore SwiftUI, I quickly encountered the keyboard covering my UI elements. Unfortunately, for all the great features and funcationality build into SwiftUI, the `TextField` doesn't yet offer the concept of an `inputAccessoryView`, or some way to 

## How do we solve this in UIKit apps?

When using UIKit, you would typically observe keyboard notifications, and adjust our view based on the content of that notification. It looks something like this:

```swift

class KeyboardObservingViewController: UIViewController {

    override func viewWillAppear(animated: Bool) {
        super.viewWillAppear(animated: animated)

        NotificationCenter.default.addObserver(
            self, for)

    }

    @objc func handleKeyboardNotification(_ notification: Notification) {
        // Parse animation curve, animation duration, and keyboard height from notification, then update your UI based on the keyboard height.
    }
}
```

This has been a reliable approach for the community, so let's see if we can recreate this in SwiftUI!

## Create a new SwiftUI View

To start, let's create a SwiftUI view that we'll update through this excercise. This view will just pin a `TextField` to the bottom of the screen like you might do for a chat application.

```swift

struct KeyboardObservingView: View {

  @State var message: String = ""

  var body: some View {
    VStack {
      Spacer()
      TextField("Your Message", text: $message)
        .padding()
        .background(Color.gray)
        .edgesIgnoringSafeArea(.bottom)    
    }
  }
}
```

## Adding some local state

If we think back to how we solve this in UIKit, we determine the animationDuration and keyboard height using the notification, and then update the view with those values. In SwiftUI, these values could be considered state that is local to your view. For hyper-local state like this, the new `@State` property wrapper is a perfect choice. 

```swift
struct KeyboardObservingView: View {

  @State var message: String = ""
  @State var keyboardHeight: CGFloat = 0
  @State var keyboardAnimationDuration: TimeInterval = 0

  var body: some View {
    VStack {
      Spacer()
      TextField("Your Message", text: $message)
        .padding()
        .background(Color.gray)
        .edgesIgnoringSafeArea(.bottom)    
    }
  }
}
```

## Combine to the Rescue

We've now created a place to store our state, but we don't yet have a way of getting to those values by observing notifications.

For this, we'll turn to the new Combine framework. SwiftUI View's have an `onReceive` method that takes a Combine `Publisher`. Fortunately for us, `NotificationCenter` can now expose `Publishers` for specific notifications!

```swift
struct KeyboardObservingView: View {

  @State var message: String = ""
  @State var keyboardHeight: CGFloat = 0
  @State var keyboardAnimationDuration: TimeInterval = 0

  var body: some View {
    VStack {
      Spacer()
      TextField("Your Message", text: $message)
        .padding()
        .background(Color.gray)
        .edgesIgnoringSafeArea(.bottom)    
    }
  }
  .onReceive(
    NotificationCenter.default.publisher(for: UIResponder.keyboardWillChangeFrameNotification)
      .receive(on: RunLoop.main),
    perform: updateKeyboardHeight
  )

  private func updateKeyboardHeight(_ notification: Notification) {
    guard let info = notification.userInfo else { return }
    // Get the duration of the keyboard animation
    keyboardAnimationLength = (info[UIResponder.keyboardAnimationDurationUserInfoKey] as? Double) ?? 0.25

    guard let keyboardFrame = info[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect else { return }
    // If the top of the frame is at the bottom of the screen, set the height to 0.
    if keyboardFrame.origin.y == UIScreen.main.bounds.height {
      keyboardHeight = 0
    } else {
      // IMPORTANT: This height will _include_ the SafeAreaInset height.
      keyboardHeight = keyboardFrame.height
    }
  }
}
```

One important thing to note from the snippet above is that I've chosen to include the bottom SafeAreaInset height in our keyboardHeight. We'll take that into account later.

## Updating the View

Now that we're observing notifications and updating our keyboard-related state, we need to update the view to take that state into account.

To do that, we'll add three view modifiers to our `VStack`:

```swift
  // Pad the bottom of the view to raise it above the keyboard
  .padding([.bottom], keyboardHeight)
  // Ignore the bottom safe area if the keyboard is on screen. We need this because
  // our keyboardHeight value includes the safe area.
  .edgesIgnoringSafeArea((keyboardHeight > 0) ? [.bottom] : [])
  // Animate the change. To my eyes, easeOut looks correct, but feel free to play around
  // with other animation types.
  .animation(.easeOut(duration: keyboardAnimationLength))
```

All together is should look something like this:

```swift
struct KeyboardObservingView: View {
  
  @State var message: String = ""
  @State var keyboardHeight: CGFloat = 0
  @State var keyboardAnimationDuration: TimeInterval = 0

  var body: some View {
    VStack {
      Spacer()
      TextField("Your Message", text: $message)
        .padding()
        .background(Color.gray)
        .edgesIgnoringSafeArea(.bottom)    
    }
  }
  .padding([.bottom], keyboardHeight)
  .edgesIgnoringSafeArea((keyboardHeight > 0) ? [.bottom] : [])
  .animation(.easeOut(duration: keyboardAnimationLength))
  .onReceive(
    NotificationCenter.default.publisher(for: UIResponder.keyboardWillChangeFrameNotification)
      .receive(on: RunLoop.main),
    perform: updateKeyboardHeight
  )

  private func updateKeyboardHeight(_ notification: Notification) {
    guard let info = notification.userInfo else { return }
    // Get the duration of the keyboard animation
    keyboardAnimationLength = (info[UIResponder.keyboardAnimationDurationUserInfoKey] as? Double) ?? 0.25

    guard let keyboardFrame = info[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect else { return }
    // If the top of the frame is at the bottom of the screen, set the height to 0.
    if keyboardFrame.origin.y == UIScreen.main.bounds.height {
      keyboardHeight = 0
    } else {
      // IMPORTANT: This height will _include_ the SafeAreaInset height.
      keyboardHeight = keyboardFrame.height
    }
  }
}
```

At this point, you should see the view updating as the keyboard is shown!

## Can we do better?

So we've got a working solution, but can we do better? This seems like a lot of code to copy in every time a view needs to avoid the keyboard.

The good news is that we can definitely do better. SwiftUI provides a `ViewModifier` protocol that allows us to add or change aspects of our `Views`. Now we'll take what we've written, and adapt it into a `ViewModifier` for much easier reuse!


## What are ViewModifiers?




```
struct KeyboardObserver: ViewModifier {

  @State var keyboardHeight: CGFloat = 0
  @State var keyboardAnimationLength: Double = 0

  func body(content: Content) -> some View {
    content
      .padding([.bottom], keyboardHeight)
      .edgesIgnoringSafeArea((keyboardHeight > 0) ? [.bottom] : [])
      .animation(.easeOut(duration: keyboardAnimationLength))
      .onReceive(
        NotificationCenter.default.publisher(for: UIResponder.keyboardWillChangeFrameNotification)
          .receive(on: RunLoop.main),
        perform: updateKeyboardHeight
      )
  }

  func updateKeyboardHeight(_ notification: Notification) {
    guard let info = notification.userInfo else { return }
    // Get the duration of the keyboard animation
    keyboardAnimationLength = (info[UIResponder.keyboardAnimationDurationUserInfoKey] as? Double) ?? 0.25

    guard let keyboardFrame = info[UIResponder.keyboardFrameEndUserInfoKey] as? CGRect else { return }
    // If the top of the frame is at the bottom of the screen, set the height to 0.
    if keyboardFrame.origin.y == UIScreen.main.bounds.height {
      keyboardHeight = 0
    } else {
      // IMPORTANT: This height will _include_ the SafeAreaInset height.
      keyboardHeight = keyboardFrame.height
    }
  }
}
```

Now we can make a view observe the keyboard like this:

```swift
struct OurBetterView: View {
  
  @State var message: String = ""

  var body: some View {
    VStack {
      Spacer()
      TextField("Your Message", text: $message)
        .padding()
        .background(Color.gray)
        .edgesIgnoringSafeArea(.bottom)    
    }
    .modifier(KeyboardObserving())
  }
}
```

Let's not stop there though! We can actually take the ergonomics one step further by adding a `keyboardObserving` method to the View protocol.

```swift
extension View {
    func keyboardObserving() -> some View {
      self.modifier(KeyboardObserving())
    }
}
```

Now, instead of the `.modifier` syntax, we can add `.keyboardObserving()` directly!

```swift
struct OurBetterView: View {
  
  @State var message: String = ""

  var body: some View {
    VStack {
      Spacer()
      TextField("Your Message", text: $message)
        .padding()
        .background(Color.gray)
        .edgesIgnoringSafeArea(.bottom)    
    }
    .keyboardObserving()
  }
}
```

## In conclusion...



I've made the functionality we've built through this post, along with a `Keyboard` type that conforms to the `ObservableObject` protocol for your other keyboard-related needs available as a Swift Package. Check out [KeyboardObserving](https://github.com/nickffox/KeyboardObserving) on Github!

I'd love to hear what you though of this post. Please feel free to share any question, thoughts, or feedback with me on [Twitter](https://www.twitter.com/nickffox)!

