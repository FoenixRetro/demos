# Multiplexed bouncing balls

This demo demonstrates a sprite multiplexing technique for the Foenix Retro Systems F256 line of modern retro computers.

The demo can be [downloaded here](demos/balls.bin)

The platform provides 64 hardware sprites, the size of which can be 8x8, 16x16, 24x24 or 32x32 pixels. This demo uses 8x8 sprites and displays 280 free-moving sprites using 44 hardware sprites.

Since only 44 8x8 hardware sprites are used, this technique can be suitable for "bullet hell" shooters, where the just the smaller bullets are multiplexed, leaving 20 hardware sprites for the hero and baddies.

To try out the demo, the machine must be started in "Boot to RAM" mode. The demo must then be loaded to address 0, and the reset button will likely have to be pressed once to start it.

Depending on your platform, the following command may be suitable:
```
fnxmgr --address 0 --binary balls.bin
```

# Concept

"Multiplexing" is a generic term that often means sharing one resource between several tasks. In the case of hardware it therefore means using one hardware to display not one, but several objects on screen. If the platform does not latch sprite X and Y coordinates, one can race the beam and modify sprite register during field scanning, before the new object's Y position is reached by the beam.

# Summary of algorithm

A "bullet hell" shooter is a great use case that requires many objects to be displayed at once. Let's work towards that goal.

Reusing sprites requires some way of keeping track of the Y order of sprites, and when it's safe to reuse a sprite, so the old object is not cut off prematurely.

This demo takes the approach of dividing the screen into "buckets" and puts each object into the bucket corresponding the object's Y position after logic/physics has completed. A bucket is several lines tall, and also a power of two so it's a matter of shifting the Y position right to find the right bucket index. This approach is generally known as [bucket sort](https://en.wikipedia.org/wiki/Bucket_sort) and is often used on lower end hardware, for instance the first PlayStation had hardware support for drawing buckets of polygons.

With the objects neatly sorted into buckets, it's time to race the beam. The platform's line interrupt is used for this, and is triggered every "bucket height" lines. In the interrupt the sprite registers are updated so the next bucket of objects is displayed properly.

# Implementation

Thinking about buckets and objects we realize that an object will, most of the time, not be fully contained within one bucket. It will straddle more than one bucket when it gets closer to the last line of its bucket. It might even straddle more than two buckets if it's taller than one bucket. Straddling more than two buckets is going to cause a headache, so let's decide to match the bucket height and maximum sprite size. In our case this is 8 lines.

Buckets should be a short as possible. The taller they are, the more objects can fit in a bucket, which will require using more hardware sprites in order to display every object without flicker or cutting parts off. We have a self-imposed limit of only using 8x8 sprites, but the upshot of this is that we can display more objects in a bucket, which means more objects on screen in total. More bullets, and bullets are small, so 8x8 is fine. This seems like a fair limitation for our use case.

With buckets 8 lines tall, we should then fire off a line interrupt every 8 lines and update some sprite registers for the upcoming objects. How do we keep track of when it's safe to reuse a sprite? To do this the technique of double buffering is used. In even buckets one set of sprites is used, and in odd buckets another set. This way we can update one set of sprites while displaying another, alternating between sets thus avoiding any objects being cut off prematurely.

Or so you would think. Let's draw a little and see what happens. We'll use buckets four lines tall in the illustration. Each line in the diagram is a scan line, and the line is prefixed with the bucket it belongs to, either `+` or `-`. Let's say we have three objects we want to display, but only two hardware sprites. The hardware sprites are illustrated by `0` and `1`. The three objects are in separate buckets

This is an ideal case:
```
+  0
+  0
+  0
+  0
-    1
-    1
-    1
-    1
+      0
+      0
+      0
+      0
```

This is also pretty good:
```
+  
+   
+  0
+  0
-  0 1
-  0 1
-    1
-    1
+      0
+      0
+      0
+      0
```

But wait, we need to update sprite 0's registers while displaying bucket `-`. And sprite 0 is still in use.

A worst case for sprite 0 is this:
```
+  
+   
+  
+  0
-  0
-  0
-  0 1
-    1
+    1 0
+    1 0
+      0
+      0
```

The first instance of sprite 0 can't be any further down the screen, then it would change bucket. Which means we have one scan line to update the sprite registers. We need to hit the line exactly, which is easy enough, but then we also need to update the sprite registers.

Cut to the future where we determine through empirical testing that we need 22 sprites for this use case.

We need to update 22 sets of sprite registers with a 6 MHz 6502. That's a pretty tall order.

We need to blast over coordinates as quickly as possible, and in the toolbox we find our old friend, self-modifying code. 

This is a sequence of instructions for the first two hardware sprite registers:
```
lda	#<X_POS_0
sta	$D904
lda	#>X_POS_0
sta	$D905
lda	#<Y_POS_0
sta	$D906
lda	#>Y_POS_0
sta	$D907

lda	#<X_POS_1
sta	$D90C
lda	#>X_POS_1
sta	$D90D
lda	#<Y_POS_1
sta	$D90E
lda	#>Y_POS_1
sta	$D90F

; and so forth ...
```

This sequence sets the sprite positions for 22 hardware sprites and is fast enough to complete without flicker.

We have a sequence of instructions like this for each bucket, and sprite positions are poked directly into the instructions after (or while) running object logic.

Yes, we burn through memory - each sprite update is 20 bytes, times 22 times the number of buckets (32), roughly 14 KiB. Oh, and we need to double buffer these too - we display one set while writing into another. 28 KiB. But that's the tradeoff if we need many objects on screen.

You will probably have noticed the demo displays 4 differently colored objects, but we have only updated positions. Ideally we want to have arbitrary object graphics, and of course we can, but we need to include the three graphics pointers:

```
lda	#<Graphics
sta	$D901
lda	#>Graphics
sta	$D902
lda	#^Graphics
sta	$D903
lda	#>X_POS_0
sta	$D905
lda	#<Y_POS_0
sta	$D906
lda	#>Y_POS_0
sta	$D907
```

This is of course much slower and will likely reduce the number of sprites that can be displayed per bucket without flicker. Or maybe the flicker is acceptable, that is up to you. In the demo, only one sprite register is updated, the least significant byte. Handily there's only 256 bytes of graphics and they happen to be page aligned, so the other pointer registers don't have to be updated.

With slower moving objects Z-fighting will be much more noticable. That's another trade-off - sprite priorities are not handled in this demo and two objects' Z order may be swapped back and forth depending on other objects' positions.

# Conclusion

Multiplexing free-moving sprites can be a useful technique, especially if you are prepared to set certain limitations. It can be generalized, but at the expense of the total number of sprites that can be displayed, which may limit it's usefulness. However, 128 sprites would still be better than 64.

Hopefully this has inspired you to implement something like this yourself or improving the technique.

Happy coding!
