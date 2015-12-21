Infinite Gest
=============

This is an experiment in gesture drawing. The idea is to make an idea
sketching environment with no tool palette. Instead, you just draw things and
a gesture recogniser tries to figure out what you meant. Think less Photoshop
or Balsamiq and more "slightly overqualified piece of paper".

Under the hood, it connects together [Fabric.js](http://fabricjs.com/) and the
[$P point-cloud
recogniser](https://depts.washington.edu/aimgroup/proj/dollar/pdollar.html)

Currently it's fairly primitive, but decent enough for showing the merit of
the idea. Some things I'd like to add:

- Lots more gestures
- Multi-line gestures (the gesture library supports it but I need some special
  logic for grouping paths to send to the recogniser)
- Nice bubbley pop-out UI for choosing between recognised gestures
- Custom smarts for connecting and aligning lines and shapes (snap to
  centre/sides etc)


Setup
-----

First we build a canvas and do a few bits and bobs to set up Fabric.js. Worth
noting is that, unlike in most Fabric.js sample code, we use `Fabric` as the
name of the library and `fabric` as the name of our instance. (Example code
uses fabric/canvas).

    Gest = (el) ->
      canvas = document.createElement 'canvas'
      el.appendChild canvas
      Fabric = window.fabric
      fabric = new Fabric.Canvas canvas

      resize = ->
        canvas.width = el.offsetWidth
        canvas.height = el.offsetHeight
        fabric.setWidth canvas.width
        fabric.setHeight canvas.height
      resize()

      window.addEventListener 'resize', resize

Start with a few basic shapes and explanatory content.

      rect = new Fabric.Rect
        left: 100, top: 200, fill: 'red', width: 100, height: 100

      rect2 = new Fabric.Rect
        left: 300, top: 200, fill: 'blue', width: 100, height: 100

      text = new Fabric.Text "Tap/click and drag to draw", left: 100, top: 0
      text2 = new Fabric.Text "Long press to select and move", left: 100, top: 50
      text3 = new Fabric.Text "Two fingers to pan and zoom", left: 100, top: 100

      fabric.add rect, rect2, text, text2, text3

Event handling
--------------

Fabric.js has a notion of "drawing mode", which turns out to be a bit of a
pain for us, because we want to be in drawing mode almost all of the time and
occasionally do non-drawing actions.

It would be great if the library supported reconfiguring the input style, but
it doesn't, so we do various hacks to get it to work the way we want.

      fabric.isDrawingMode = true
      fabric.freeDrawingCursor = fabric.defaultCursor

We use long-presses to temporarily drop out of drawing mode to select, move,
and transform existing shapes. After disabling the current drawing state, we
re-trigger the standard mousedown event.

      fabric.on 'touch:longpress', (ev) ->
        console.log "LONGPRESS!"
        return if ev.e.type in ['mouseup', 'touchend']
        ev.e.fromLongPress = true
        fabric._isCurrentlyDrawing = false
        fabric.isDrawingMode = false
        fabric._onMouseDown ev.e

The mousedown and mouseup handlers are used re-enter drawing mode when you
click away from moving a shape. We recognise when you've moused down in an
empty area, reset everything, and turn drawing mode back on when you mouse up
again.

      backToDrawing = false
      fabric.on 'mouse:down', (ev) ->
        return if fabric.isDrawingMode or ev.e.fromLongPress
        console.log "mouse DOWN"
        backToDrawing = true
        target = fabric.findTarget(ev.e)
        shouldClear = fabric._shouldClearSelection ev.e, target
        console.log "mousedown!", ev, "target", target, "shouldClear", shouldClear

        if shouldClear
          pointer = fabric.getPointer ev.e, true
          fabric._clearSelection ev.e, target, pointer
          fabric._groupSelector = null

      fabric.on 'mouse:up', (ev) ->
        return unless backToDrawing
        backToDrawing = false
        fabric.isDrawingMode = true

Zooming and panning
-------------------

The name wouldn't work without an infinite canvas! We support two ways of
gesturing: mouse and touchscreen. There's no unification of those events so we
handle them separately.

      scale = (n) -> if n < 0 then 1/(-(n-1)) else n+1
      currentZoom = 1
      startPointer = null
      pointerTimeout = null

      fabric.wrapperEl.addEventListener 'mousewheel', (ev) ->
        ev.preventDefault()
        if ev.ctrlKey
          currentZoom -= ev.deltaY * 0.2

          startPointer = fabric.getPointer ev, true
          fabric.zoomToPoint startPointer, scale(currentZoom)
        else
          fabric.relativePan {x: -ev.deltaX, y: -ev.deltaY}

The gesture event unfortunately triggers drawing as well, and that's hardcoded
into Event.js, so we do what we have to do...

      fabric.__oldOnTransformGesture = fabric.__onTransformGesture
      fabric.__onTransformGesture = (e, self) ->
        if self.fingers is 2
          fabric._isCurrentlyDrawing = false
          fabric.isDrawingMode = false
        this.__oldOnTransformGesture(e, self)

The gesture event's zoom value is relative to when the gesture started, and
the position is absolute. Unfortunately we want a relative position and an
absolute zoom. C'est la vie.

      gestureZoom = 1
      lastPan = null

      fabric.on 'touch:gesture', (ev) ->
        console.log "touchgesture"
        gestureZoom = ev.self.scale
        fabric.zoomToPoint ev.self, currentZoom * gestureZoom
        if lastPan
          fabric.relativePan {x: ev.self.x - lastPan.x, y: ev.self.y - lastPan.y}
        lastPan = {x: ev.self.x, y: ev.self.y}

On mouseup we reset these values for the next gesture, and also turn drawing
mode back on since we disabled it in the monkey patched gesture handler.

      fabric.on 'mouse:up', (ev) ->
        if lastPan
          console.log "lastpan"
          fabric.isDrawingMode = true

        currentZoom *= gestureZoom
        gestureZoom = 1
        lastPan = null


Gesture recognition
-------------------

This is probably the grossest part of the code. I think the right answer here
is to make a brand new brush class, but instead we just hack up the default
PencilBrush.

      recogniser = new PDollarRecognizer()

On mouseup we commit the shape into an SVG path, so before that point we send
it through the recogniser. If the recogniser gets it, we scale the result back
to the dimensions of what we draw and save that as our path instead.

      Fabric.PencilBrush.prototype.onMouseUp = ->
        points = ({x, y, id: 0} for {x, y} in this._points)
        result = recogniser.Recognize(points)
        console.log "recogniser says", result

        left = this._points[0].x
        right = this._points[0].x
        top = this._points[0].y
        bottom = this._points[0].y
        for {x, y} in this._points
          left = x if x < left
          right = x if x > right
          top = y if y < top
          bottom = y if y > bottom

        norm = ({x, y, id}) ->
          x: (x + 0.5) * (right - left) + left
          y: (y + 0.5) * (bottom - top) + top
          id: id

        if result.score > 0.1
          this._points = (norm point for point in result.result.Points)

        this._finalizeAndAddPath()

The shape we get back from the recogniser may have multiple strokes in it, but
drawn shapes are only designed to have a single stroke. No matter, we just
copy-paste the convert function, patch the `.id` (stroke number) property
through, and hack in multiline support.

      Fabric.PencilBrush.prototype.convertPointsToSVGPath = (points) ->
        path = []
        p1 = new (Fabric.Point)(points[0].x, points[0].y)
        p1.id = points[0].id
        p2 = new (Fabric.Point)(points[1].x, points[1].y)
        p2.id = points[1].id
        path.push 'M ', points[0].x, ' ', points[0].y, ' '
        i = 1
        len = points.length
        while i < len
          if p2.id is p1.id
            midPoint = p1.midPointFrom(p2)
            # p1 is our bezier control point
            # midpoint is our endpoint
            # start point is p(i-1) value.
            path.push 'Q ', p1.x, ' ', p1.y, ' ', midPoint.x, ' ', midPoint.y, ' '
          else # Different id lines means we should move to the next point
            path.push 'L ', p1.x, ' ', p1.y, ' '
            path.push 'M ', p2.x, ' ', p2.y, ' '
          p1 = new (Fabric.Point)(points[i].x, points[i].y)
          p1.id = points[i].id
          if i + 1 < points.length
            p2 = new (Fabric.Point)(points[i + 1].x, points[i + 1].y)
            p2.id = points[i + 1].id
          i++
        path.push 'L ', p1.x, ' ', p1.y, ' '
        path

Exports
-------

No proper module support for now. But, hey, we technically kinda support web
components.

    window.Gest = Gest

    ijs = document.getElementsByTagName 'infinite-jest'
    Gest ij for ij in ijs