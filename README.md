# StrokeWidthTransform
A playground implementing Epshtein, Ofek, and Wexler's Stroke Width Transform

Motivation
==========
Optical character recognition (OCR) typically works well on clean images, but
poorly on noisy ones. The [Stroke Width
Transform](http://research.microsoft.com/pubs/149305/1509.pdf) is a technique
used to extract text from a noisy image by isolating connected shapes that
share a consistent _stroke width_. The resulting image is much more reliable
when processed via OCR.

How it Works
============
SWT works by first identifying high contrast edges in a given image. By
traversing the image at each edge pixel, in the direction normal to the edge,
until another normal edge is found, we can effectively identify _strokes_ in
the image. A stroke is an element of finite width with two roughly parallel
sides, as you might find in a pen stroke.

By measuring the width of this stroke, and connecting adjacent pixels with
similar associated stroke widths, we can extract each "pen stroke" from the
image.  A contiguous stroke typically represents a man-made character, since
most natural or otherwise noisy shapes don't exhibit stroke-like
characteristics, and those that do usually don't possess consistent stroke width.

An OpenCV Implementation with Python
====================================
Inspired by [@aperrau's DetectText
project](https://github.com/aperrau/detecttext), I wanted to play with this
solution in Python. To get things rolling, I installed OpenCV with homebrew,
set up a virtual environment, and symlinked the new OpenCV modules into my
environment. Note that I'm using Python 2.7 and OpenCV 2.4.12 here on OSX:

    $ brew tap homebrew/science
    $ brew install opencv

    $ virtualenv env
    $ cd env/lib/python2.7/site-packages
    $ ln -s /usr/local/Cellar/opencv/2.4.12/lib/python2.7/site-packages/cv.py cv.so
    $ ln -s /usr/local/Cellar/opencv/2.4.12/lib/python2.7/site-packages/cv2.so cv2.so

I have a basic (but slow and disorganized) prototype of SWT implemented at
[github.com/mypetyak/StrokeWidthTransform](github.com/mypetyak/StrokeWidthTransform).

SWT operates in a few steps:  

1. Use OpenCV to extract image edges using [Canny edge
detection](https://en.wikipedia.org/wiki/Canny_edge_detector)

2. Calculate the x- and y-derivatives of the image, which can be superimposed
to calculate the image gradient. The gradient describes, for each pixel, the
direction of greatest contrast. In the case of an edge pixel, this is
synonymous with the vector normal to the edge.

3. For each edge pixel, traverse in the direction θ of the gradient until the
next edge pixel is encountered (or you fall off the image). If the
corresponding edge pixel's gradient is pointed in the opposite direction
(θ - π), we know the newly-encountered edge is roughly parallel to the
first, and we have just cut a slice (line) through a stroke. Record the stroke width,
in pixels, and assign this value to all pixels on the slice we just traversed.

4. For pixels that may belong to multiple lines, reconcile differences in those
line widths by assigning all pixels the median stroke width value. This allows 
the two strokes you might encounter in an 'L' shape to be considered with the
same, most common, stroke width.

5. Connect lines that overlap using a union-find (disjoint-set) data
structure, resulting in a disjoint set of all overlapping stroke slices. Each
set of lines is likely a single letter/character.

6. Apply some intelligent filtering to the line sets; we should eliminate
anything to small (width, height) to be a legible character, as well as
anything too long or fat (width:height ratio), or too sparse (diameter:stroke
width ratio) to realistically be a character.

7. Use a k-d tree to find pairings of similarly-stroked shapes (based on
stroke width), and intersect this with pairings of similarly-sized shapes
(based on height/width). Calculate the angle of the text between these two
characters (sloping upwards, downwards?).

8. Use a k-d tree to find pairings of letter pairings with similar
orientations. These groupings of letters likely form a word. Chain similar
pairs together.

9. Produce a final image containing the resulting words.

