# Flutter_doc_CokBK_Efct_Create_gradient_chat_bubbles
 https://docs.flutter.dev/cookbook/effects/gradient-bubbles#replace-original-background-widget
Create gradient chat bubbles
============================

1.  [Cookbook](https://docs.flutter.dev/cookbook)
2.  [Effects](https://docs.flutter.dev/cookbook/effects)
3.  [Create gradient chat bubbles](https://docs.flutter.dev/cookbook/effects/gradient-bubbles)

Traditional chat apps display messages in chat bubbles with solid color backgrounds. Modern chat apps display chat bubbles with gradients that are based on the bubbles' position on the screen. In this recipe, you'll modernize the chat UI by implementing gradient backgrounds for the chat bubbles.

The following animation shows the app's behavior:

![Scrolling the gradient chat bubbles](https://docs.flutter.dev/assets/images/docs/cookbook/effects/GradientBubbles.gif)

[](https://docs.flutter.dev/cookbook/effects/gradient-bubbles#understand-the-challenge)Understand the challenge
---------------------------------------------------------------------------------------------------------------

The traditional chat bubble solution probably uses a `DecoratedBox` or a similar widget to paint a rounded rectangle behind each chat message. That approach is great for a solid color or even for a gradient that repeats in every chat bubble. However, modern, full-screen, gradient bubble backgrounds require a different approach. The full-screen gradient, combined with bubbles scrolling up and down the screen, requires an approach that allows you to make painting decisions based on layout information.

Each bubble's gradient requires knowledge of the bubble's location on the screen. This means that the painting behavior requires access to layout information. Such painting behavior isn't possible with typical widgets because widgets like `Container` and `DecoratedBox` make decisions about background colors before layout occurs, not after. In this case, because you require custom painting behavior, but you don't require custom layout behavior or custom hit test behavior, a [`CustomPainter`](https://api.flutter.dev/flutter/rendering/CustomPainter-class.html) is a great choice to get the job done.

info Note: In cases where you need control over the child layout, but you don't need control over the painting or hit testing, consider using a [`Flow`](https://api.flutter.dev/flutter/widgets/Flow-class.html) widget.

In cases where you need control over the layout, painting, *and* hit testing, consider defining a custom [`RenderBox`](https://api.flutter.dev/flutter/rendering/RenderBox-class.html).

[](https://docs.flutter.dev/cookbook/effects/gradient-bubbles#replace-original-background-widget)Replace original background widget
-----------------------------------------------------------------------------------------------------------------------------------

Replace the widget responsible for drawing the background with a new stateless widget called `BubbleBackground`. Include a `colors` property to represent the full-screen gradient that should be applied to the bubble.

content_copy

```
BubbleBackground(
  // The colors of the gradient, which are different
  // depending on which user sent this message.
  colors: message.isMine
      ? const [Color(0xFF6C7689), Color(0xFF3A364B)]
      : const [Color(0xFF19B7FF), Color(0xFF491CCB)],
  // The content within the bubble.
  child: DefaultTextStyle.merge(
    style: const TextStyle(
      fontSize: 18.0,
      color: Colors.white,
    ),
    child: Padding(
      padding: const EdgeInsets.all(12),
      child: Text(message.text),
    ),
  ),
);
```

[](https://docs.flutter.dev/cookbook/effects/gradient-bubbles#create-a-custom-painter)Create a custom painter
-------------------------------------------------------------------------------------------------------------

Next, introduce an implementation for `BubbleBackground` as a stateless widget. For now, define the `build()` method to return a `CustomPaint` with a `CustomPainter` called `BubblePainter`. `BubblePainter` is used to paint the bubble gradients.

content_copy

```
@immutable
class BubbleBackground extends StatelessWidget {
  const BubbleBackground({
    super.key,
    required this.colors,
    this.child,
  });

  final List<Color> colors;
  final Widget? child;

  @override
  Widget build(BuildContext context) {
    return CustomPaint(
      painter: BubblePainter(
        colors: colors,
      ),
      child: child,
    );
  }
}

class BubblePainter extends CustomPainter {
  BubblePainter({
    required List<Color> colors,
  }) : _colors = colors;

  final List<Color> _colors;

  @override
  void paint(Canvas canvas, Size size) {
    // TODO:
  }

  @override
  bool shouldRepaint(BubblePainter oldDelegate) {
    // TODO:
    return false;
  }
}
```

[](https://docs.flutter.dev/cookbook/effects/gradient-bubbles#provide-access-to-scrolling-information)Provide access to scrolling information
---------------------------------------------------------------------------------------------------------------------------------------------

The `CustomPainter` requires the information necessary to determine where its bubble is within the `ListView`'s bounds, also known as the `Viewport`. Determining the location requires a reference to the ancestor `ScrollableState` and a reference to the `BubbleBackground`'s `BuildContext`. Provide each of those to the `CustomPainter`.

content_copy

```
BubblePainter(
  colors: colors,
  bubbleContext: context,
  scrollable: ScrollableState(),
),
```

content_copy

```
class BubblePainter extends CustomPainter {
  BubblePainter({
    required ScrollableState scrollable,
    required BuildContext bubbleContext,
    required List<Color> colors,
  })  : _scrollable = scrollable,
        _bubbleContext = bubbleContext,
        _colors = colors;

  final ScrollableState _scrollable;
  final BuildContext _bubbleContext;
  final List<Color> _colors;

  @override
  bool shouldRepaint(BubblePainter oldDelegate) {
    return oldDelegate._scrollable != _scrollable ||
        oldDelegate._bubbleContext != _bubbleContext ||
        oldDelegate._colors != _colors;
  }
}
```

[](https://docs.flutter.dev/cookbook/effects/gradient-bubbles#paint-a-full-screen-bubble-gradient)Paint a full-screen bubble gradient
-------------------------------------------------------------------------------------------------------------------------------------

The `CustomPainter` now has the desired gradient colors, a reference to the containing `ScrollableState`, and a reference to this bubble's `BuildContext`. This is all the information that the `CustomPainter` needs to paint the full-screen bubble gradients. Implement the `paint()` method to calculate the position of the bubble, configure a shader with the given colors, and then use a matrix translation to offset the shader based on the bubble's position within the `Scrollable`.

content_copy

```
class BubblePainter extends CustomPainter {
  BubblePainter({
    required ScrollableState scrollable,
    required BuildContext bubbleContext,
    required List<Color> colors,
  })  : _scrollable = scrollable,
        _bubbleContext = bubbleContext,
        _colors = colors;

  final ScrollableState _scrollable;
  final BuildContext _bubbleContext;
  final List<Color> _colors;

  @override
  bool shouldRepaint(BubblePainter oldDelegate) {
    return oldDelegate._scrollable != _scrollable ||
        oldDelegate._bubbleContext != _bubbleContext ||
        oldDelegate._colors != _colors;
  }

  @override
  void paint(Canvas canvas, Size size) {
    final scrollableBox = _scrollable.context.findRenderObject() as RenderBox;
    final scrollableRect = Offset.zero & scrollableBox.size;
    final bubbleBox = _bubbleContext.findRenderObject() as RenderBox;

    final origin =
        bubbleBox.localToGlobal(Offset.zero, ancestor: scrollableBox);
    final paint = Paint()
      ..shader = ui.Gradient.linear(
        scrollableRect.topCenter,
        scrollableRect.bottomCenter,
        _colors,
        [0.0, 1.0],
        TileMode.clamp,
        Matrix4.translationValues(-origin.dx, -origin.dy, 0.0).storage,
      );
    canvas.drawRect(Offset.zero & size, paint);
  }
}
```

    canvas.drawRect(Offset.zero & size, paint);

  }

}
