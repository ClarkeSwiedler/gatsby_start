---
title: Declarative Animation in Flutter
date: '2019-07-25'
published: true
layout: post
tags: ['flutter', 'dart', 'animation']
category: software
---

This article will go over methods of animating Flutter widgets based on state, with some solutions to make it a little more straightforward.

## The Short of It
If you know what you're looking for, are familiar with Flutter animations and Widget lifecycles, and just want to know how to trigger animations based on state changes without reading a whole article, the answer is that you need to override the ```didUpdateWidget``` method in ```State<StatefulWidget>```. Here's a quick example, followed by some discussion on Flutter animations and generalized solutions.

```dart
//An Widget that animates visiblity based on a "visible" property.
class MyWidget extends StatefulWidget{
  final bool visible;

  MyWidget({@required this.visible});

  @override
  _MyWidgetState createState() => _MyWidgetState();
}

class _MyWidgetState extends State<MyWidget> with SingleTickerProviderStateMixin {
  AnimationController _controller;
  Animation _animation;

  @override
  void initState() {
    super.initState();
    /*Your setup logic for _controller and _animation*/
  }

  //Here's the important bit
  //This method is called whenever the StatefulWidget is called with different properties
  @override
  void didUpdateWidget(MyWidget oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.visible != oldWidget.visible) {
      if (widget.visible) {
        _controller.forward();
      } else {
        _controller.reverse();
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    /*Your widget that has some property that depends on the _animation*/
  }
}
```
We will go into much greater detail about below.

## The Long of It
Flutter's declarative programming style works extremely well for building a UI and populating it with data, allowing changes in the data state to update the UI state automatically. It's very easy to tell the framework that, in terms of the UI, you would like something to _be different_ based on the state. However it is less straightforward to tell the framework that you would like something to __happen__ based on the state.

As an example, let's say we're using a ```StreamBuilder```, and we want different widgets to be visible based on the state of the stream. If the stream has no data, a widget with a loading indicator should slide into view, and, importantly, should slide back out of view when the stream has data.

The common way to handle loading states for streams is something like the following:

```dart
Widget build(context) {
  return StreamBuilder(
    stream: _dataStream,
    builder: (context, snapshot) {
      if (!snapshot.hasData) {
        return LoadingWidget()
      }
      return DataWidget(data: snapshot.data);
    }
  );
}
```

This will cause the screen to transition completely between one widget and the other when the status of the snapshot data changes. However, let's say that we actually want to have the Loading widget sit on top of the DataWidget when the stream has no data, either because the DataWidget still has some interactivity even when it doesn't have stream data, or because we just like the look of a floating progress indicator. We might do something like this:

```dart
Widget build(context) {
  StreamBuilder(
    stream: _dataStream,
    builder: (context, snapshot) {
      return Stack(children: <Widget>[
        //Here we assume the DataWidget can handle a null value
        DataWidget(data: snapshot.data),
        if (!snapshot.hasData) LoadingWidget()
      ]);
    }
  );
}
```

Okay, so now our LoadingWidget sits on top of the DataWidget when the stream has no data, but rather than just appearing on top, we want it to slide in from the top of the screen. There are several ways to do this, but the most straightforward might be to use either a ```SlideTransition``` widget, or an ```AnimatedPositioned``` widget. Both have some drawbacks in this scenario.

The ```SlideTransition``` widget requires that it be provided with an ```AnimationController```, which must be checked and controlled from somewhere within the ```build``` method.

```dart
class ParentWidget extends StatefulWidget {
  createState() => ParentWidgetState();
}
class ParentWidgetState extends State<ParentWidget>
  with SingleTickerProviderStateMixin {

  AnimationController _controller;
  Animation _slideAnimation;

  @override
  initState() {
    super.initState();
    _controller = AnimationController(vsync: this, duration: Duration(milliseconds: 200));
    _slideAnimation = Tween<Offset>(begin: Offset(0, -1), end: Offset(0, 0)).animate(_controller);
  }

  @override
  Widget build(BuildContext context) {
    return StreamBuilder(
      stream: _dataStream,
      builder: (context, snapshot) {
        if (!snapshot.hasData && _controller.status == AnimationStatus.dismissed) {
          _controller.forward();
        } else if (snapshot.hasData && _controller.status == AnimationStatus.completed) {
          _controller.reverse();
        }
        return Stack(children: <Widget>[
          DataWidget(data: snapshot.data),
          SlideTransition(
            position: _slideAnimation,
            child: LoadingWidget(),
          )
        ]);
      }
    );
  }
}
```
This introduces a lot of extra code and, more importantly, our ```build``` method is suddenly beset by a big block of imperative code nestled uncomfortably amist all of our nice declarative UI code. It only gets worse if there are several overlay widgets that are animated based on various aspects of the stream state.

Alright then, let's see if we can do better with ```AnimatedPositioned```.

```dart
class ParentWidget extends StatelessWidget {

  double loadingWidgetHeight = 50;
  double hiddenOvershoot = 10;

  @override
  Widget build(BuildContext context) {
    Size screenSize = MediaQuery.of(context).size;
    return StreamBuilder(
      stream: _dataStream,
      builder: (context, snapshot) {
        return Stack(children: <Widget>[
          DataWidget(data: snapshot.data),
          AnimatedPositioned(
            top: 0,
            right: 0,
            left: 0,
            bottom: snapshot.hasData
              ? screenSize.height + hiddenOvershoot
              : screenSize.height - loadingWidgetHeight,
            child: LoadingWidget(),
          )
        ]);
      }
    );
  }
}
```

Not too bad, and certainly not the only way to achieve this effect with ```AnimatedPositioned```, but there are still some drawbacks. First, the ```AnimatedPositioned``` widget MUST be a child of a ```Stack```. Second, we lose the handy ```Offset``` parameter that makes the ```SlideTransition``` so easy to use. We have to work with the screensize and an explicit container size in order to know where we should tell the ```AnimatedPositioned``` widget to put itself.

### Let's build a widget to make this easy
Now we'll flesh out the example at the very top of this article and turn it into a reusable widget that will animate it's child with a ```SlideTransition``` based on a ```visible``` property. We'll also go ahead and give the widget some properties to pass into it's animation so that we can get some more fine grained control of the animation if we want.

```dart
class SlideVisible extends StatefulWidget {

  ///Whether or not this widget should be shown.
  final bool visible;

  ///The widget that is shown or hidden by this widget.
  final Widget child;

  ///The Offset of this widget when visible is false.
  ///
  ///Defaults to Offset(0, -1.1), which will slide the widget up by
  ///slightly more than its height when the widget is hidden.
  final Offset hiddenOffset;

  ///The [Offset] of this widget when [visible] is true.
  ///
  ///Defaults to Offset(0,0).
  final Offset visibleOffset;

  ///The [Duration] of the slide animation.
  ///
  ///Defaults to 300ms.
  final Duration duration;

  ///The [Curve] that the slide animation will follow.
  ///
  ///Defaults to [Curves.linear]
  final Curve curve;

  SlideVisible({
    Key key,
    @required this.visible,
    @required this.child,
    this.hiddenOffset = const Offset(0, -1.1),
    this.visibleOffset = const Offset(0, 0),
    this.duration = const Duration(milliseconds: 300),
    this.curve = Curves.linear,
  }) : super(key: key);

  @override
  _SlideVisibleState createState() => _SlideVisibleState();
}

class _SlideVisibleState extends State<SlideVisible>
    with SingleTickerProviderStateMixin {
  AnimationController _controller;
  Animation _slideAnimation;

  @override
  void initState() {
    super.initState();

    _controller = AnimationController(vsync: this, duration: widget.duration);

    _slideAnimation =
        Tween<Offset>(begin: widget.hiddenOffset, end: widget.visibleOffset)
            .animate(CurvedAnimation(
      curve: widget.curve,
      parent: _controller,
    ));

    //Ensure that the animation will not play if the widget starts as visible.
    if (widget.visible) {
      _controller.value = _controller.upperBound;
    }
  }

  @override
  void didUpdateWidget(SlideVisible oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.visible != oldWidget.visible) {
      if (widget.visible) {
        _controller.forward();
      } else {
        _controller.reverse();
      }
    }
  }

  @override
  Widget build(BuildContext context) {
    return SlideTransition(
      position: _slideAnimation,
      child: widget.child,
    );
  }
}
```

### Using our new widget

Now we will use ```SlideVisible``` from the above example to accomplish the same goal as the rest of the examples from this article.

```dart
class ParentWidget extends StatlessWidget {
  @override
  Widget build(BuildContext context) {
    return StreamBuilder(
      stream: _dataStream,
      builder: (context, snapshot) {
        return Stack(children: <Widget>[
          DataWidget(),
          Align(
            alignment: Alignment.topCenter,
            child: SlideVisible(
              visible: !snapshot.hasData,
              hiddenOffset: const Offset(0, -1.1),
              child: LoadingWidget(),
            )
          ),
        ]);
      }
    )
  }
}
```
Everything about the widget and the animation is contained in one place, and all we have to do is tell it whether or not it should currently be ```visible```. We can change the ```hiddenOffset``` to control the direction from which it enters and exits the screen, and we don't have to worry too much about the size of the widget itself, or mess around with ```AnimationControllers``` to get them to agree with our state.

Optionally we can pass in a ```Duration``` and/or a ```Curve``` for finer control of the animation, and this information is situated within the widget declaration, rather than in the ```initState``` method elsewhere in the parent widget class.

That's all there is to it. Of course there are more animation options in Flutter than have been discussed here. I haven't even mentioned ```AnimatedBuilder``` or ```AnimatedWidget```, either of which might be used to achieve similar effects, that's a topic for a later date.
