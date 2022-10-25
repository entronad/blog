# Gesture

As a touch-first GUI framework, interactions in Flutter is based on the gesture system.

There are two layers of the gesture system. The first layer has raw pointer events that describe the location and movement of pointers (for example, touches, mice, and styli) across the screen.The second layer has *gestures* that describe semantic actions that consist of one or more pointer movements. Note that these gestures include not only touches, but also all other pointer kinds on multi-platforms.

Since Graphic is a widget level visualization library, we chose the gestures layer as the foundation for its interaction system.

The widely used widget that handles gestures is the [GestureDetector](https://api.flutter.dev/flutter/widgets/GestureDetector-class.html). It defines all the gesture types in its callback properties (like [onTap](https://api.flutter.dev/flutter/widgets/GestureDetector-class.html)), and developers are familiar with them. So Graphic inherits this taxonomy and the [GestureType](https://pub.dev/documentation/graphic/latest/graphic/GestureType.html)s of Graphic have same names (without the `on` prefix) and meanings to their corresponding callback properties in GestureDetector, such as `GestureType.tap` behaves all the same as `GestureDetector.onTap`. This keeps Graphic consistent with the Flutter gesture system and friendly to beginners.



# Signal

[Signal](https://pub.dev/documentation/graphic/latest/graphic/Signal-class.html)s are also called "events" in some other systems. They are emitted when users or external changes interact with the chart. They carry the information about the interaction for updater callbacks to handle with.

Except the [GestureSignal](https://pub.dev/documentation/graphic/latest/graphic/GestureSignal-class.html) for user interactions, there are also [ChangeDataSignal](https://pub.dev/documentation/graphic/latest/graphic/ChangeDataSignal-class.html) and [ResizeSignal](https://pub.dev/documentation/graphic/latest/graphic/ResizeSignal-class.html) for the external changes that effects the chart, which are also "interactions" in a broad sense.

Internally, different signals, no matter its kind or emitter, will be broadcasted by a reducer to all signal updaters. This makes developers unconstrained to decide which signal the updater will respond to.

# Selection



# Updater



# Channel