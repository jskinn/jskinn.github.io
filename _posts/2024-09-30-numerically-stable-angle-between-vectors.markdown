---
layout: post
title:  "Numerically Stable Angles Between Vectors"
date:   2024-09-30 21:18:12 +1000
categories:
  - numerical-stability
  - trigonometry
  - vectors
  - geometry
---

TL;DR: the correct way to compute the angle between two vectors $$v_1$$ and $$v_2$$ is[^1]:

$$
\theta = 2 \arctan\left(\left| \frac{v_1}{|v_1|} - \frac{v_2}{|v_2|} \right|, \left| \frac{v_1}{|v_1|} + \frac{v_2}{|v_2|} \right|\right)
$$

Note that the order of the arguments varies between programming languages.
The given order is correct for pytorch, the first term is the "opposite" side of the relevant triangle, and the second term the "adjacent".

This post owes a lot to Prof. W. Kahan, and is basically a restatement of this StackOverflow question: [https://scicomp.stackexchange.com/questions/27689/numerically-stable-way-of-computing-angles-between-vectors](https://scicomp.stackexchange.com/questions/27689/numerically-stable-way-of-computing-angles-between-vectors).


## First Attempts: Arccos

When asked to calculate the angle between two vectors $$v_1$$ and $$v_2$$, many developers will recall that the dot product is:

$$
v_1 \cdot v_2 = \vert v_1 \vert \vert v_2 \vert \cos{\theta}
$$

and end up with an implementation something like this:
```python
def vector_angle_arccos(v1: np.ndarray, v2: np.ndarray) -> np.ndarray:
    dot_product = np.dot(v1, v2)
    denominator = np.linalg.norm(v1) * np.linalg.norm(v2)
    return np.arccos(dot_product / denominator)
```

Most of the time this will work, particularly for nicely spaced vectors with an angle like $$\frac{\pi}{4}$$ between them.
However, there are a few key edge cases to consider:
- When the length of either vector is much larger than the other
- When the angle between the vectors approaches zero
- When the angle approaches $$\pi$$ (the maximum possible angle)

The first of these is mostly an issue when $$\vert v_1 \vert \vert v_2 \vert$$ either overflows or becomes zero, causing NaNs.
In practice, I didn't find many issues with vectors of very different sizes, adjusting the formula to

$$\arccos\left(\frac{v_1}{\vert v_1 \vert} \cdot \frac{v_2}{\vert v_2 \vert}\right)$$

is about the best we can do to mitigate those issues.

The bigger issues arise from the second two points.
Below is a chart showing the error between the true angle, and the angle calculated between $$[1.0, 0.0, 0.0]$$ and $$[\sin{\theta}, \cos{\theta}], 0.0$$:

{% include posts/2024-09-30-numerically-stable-angle-between-vectors/arccosine_plot.html %}

The error rises significantly near $$0°$$ and $$180°$$.
If the angle is being either _minimised_ or _maximised_, such as as part of an objective function,
then the optimisation will drive the vectors into exactly the range where the error is greatest.
Zooming in on the output values for small angles quickly reveals the issue:

{% include posts/2024-09-30-numerically-stable-angle-between-vectors/arccosine_plot_small.html %}

The $$\arccos$$ operation can clearly only produce a limited range of values below $$0.1°$$.
In the context of minimisation again, small changes in the input vectors can have no effect on the resulting angle,
making optimisation difficult.

## Next Steps: Getting the cross product involved

The next piece of information that might spring to mind is that the cross product scales with the sine of the angle:

$$
v_1 \times v_2 = \vert v_1 \vert \vert v_2 \vert \sin{\theta} \hat{n} 
$$

The cross product is harder to compute than the dot product,
and are only well-defined in three (or apparently [seven](https://en.wikipedia.org/wiki/Seven-dimensional_cross_product)) dimensions.
Pressing on anyway, it's possible to rearrange the above equation and use $$\arcsin$$ to compute the angle.
However, this instead has the same issues as $$\arccos$$ around $$90°$$:

{% include posts/2024-09-30-numerically-stable-angle-between-vectors/arcsin_plot_small.html %}

Notice as well that for angles above $$90°$$, it becomes ambiguous, the values are the same as for angles below $$90°$$.

A more princpled approach is to combine the two previous methods and use the arctangent:

$$
\theta = \arctan2\left(\left| v_1 \times v_2 \right|, \left| v_1 \cdot v_2 \right|\right)
$$

This does address the issues with the precision of $$\arcsin$$ and $$\arccos$$ around the singularities.
However, the issues with the cross product remain.

## A Better Way

An improved formulation comes from ["Computing Cross-Products and Rotations in 2- and 3-Dimensional Euclidean Spaces"](https://people.eecs.berkeley.edu/~wkahan/MathH110/Cross.pdf) by W. Kahan[^1]:

$$
\theta = 2 \arctan\left(\left| \frac{v_1}{|v_1|} - \frac{v_2}{|v_2|} \right|, \left| \frac{v_1}{|v_1|} + \frac{v_2}{|v_2|} \right|\right)
$$

Kahan describes this equation as "unfamiliar",
however it has a relatively intuitive geometric derivation, which should hopefully allow it to become more familiar.

![Diagram]({{ "/images/posts/2024-09-30-numerically-stable-angle-between-vectors/rohmbus_sum.svg" | relative_url }})

Consider the rhombus formed by adding $$\frac{v_1}{\vert v_1 \vert}$$ and $$\frac{v_2}{\vert v_2 \vert}$$ in different orders.
Since both vectors are normalised, all the sides have the same length.
The diagonals of a rhombus bisect each other, are at right angles to each other, and bisect the interior angles of the rhombus.
One diagonal will be $$\left| \frac{v_1}{\vert v_1 \vert} + \frac{v_2}{\vert v_2 \vert} \right|$$, and the other $$\left| \frac{v_1}{\vert v_1 \vert} - \frac{v_2}{\vert v_2 \vert} \right|$$.
The four inner triangles therefore have side lengths of half the diagonals, and an internal angle half of the target angle between the vectors.
This angle can be found with the arctangent (the halves cancel), producing the above equation.

{% include posts/2024-09-30-numerically-stable-angle-between-vectors/arctangent_plot_small.html %}

This plot shows the calulated angle using this formula, over the same range as the cosine plot above.
The arctangent exhibits none of the precision issues demonstrated for $$\arccos$$.

[^1]: Kahan, W. (1981). Computing Cross-Products and Rotations in 2-and 3-Dimensional Euclidean Spaces. University of California at Berkeley, 2016.

