---
layout: post
title:  "Representations of SO(3)"
date:   2024-02-04 20:51:23 +1000
categories:
  - pose-estimation
  - SO(3)
  - 3D-geometry
  - geometry
---

This post covers different common mathematical representations of rotations in 3 dimensions, otherwise known as SO(3).

# Rotation Matrices

The first way to express a 3D rotation that most people learn about is as a matrix.
Rotation matrices are $$3 \times 3$$ matrices with the following additional properties:
- The rows are orthonormal. Equivalently, this implies that the matrix is invertible, and that $$R^{-1} = R^T$$.
- The determinant of the matrix is 1

Rotation matrices are relatively intuitive to use for those familiar with general matrices.
Their group operation is matrix multiplication, as is the process for rotating a vector.
Tools like Matlab or numpy make working with rotation matrices relatively straightforward.

Rotation matrices are most easily constructed as a set of basis vectors, which can be found through orthonormalisation.
For instance, to find a rotation for one point $$a \in \mathcal{R}^3$$ "looking at" another pont $$b \in \mathcal{R}^3$$,
start with the vector from the source to the target, and normalise it as $$\hat{u}$$:

$$
\hat{u} = \frac{b - a}{\vert b - a \vert}
$$

Then, choose an arbitrary "up" direction $$v$$ that is not parallel to $$\hat{u}$$, since roll around that axis is as yet unconstrained (often $$[0, 0, 1]$$ or $$[0, 1, 0]$$).
Then, orthogonalise $$v$$ by subtracting any component parallel to $$\hat{u}$$, and normalise:

$$
\begin{align}
v' &= v - (v \cdot \hat{u}) \hat{u} \\
\hat{v} &= \frac{v'}{\vert v' \vert}
\end{align}
$$

Finally, compute a third basis vector as the cross product of $$\hat{u}$$ and $$\hat{v}$$.
These three vectors can be stacked as columns to produce a $$3 \times 3$$ rotation matrix:

$$
\begin{align}
\hat{w} &= \hat{u} \times \hat{v} \\
R &= \begin{bmatrix}
\hat{u}_0 & \hat{v}_0 & \hat{w}_0 \\
\hat{u}_1 & \hat{v}_1 & \hat{w}_1 \\
\hat{u}_2 & \hat{v}_2 & \hat{w}_2
\end{bmatrix} 
\end{align}
$$

Pre-multiplication by $$R$$ will map $$[1, 0, 0]$$ to $$\hat{u}$$, $$[0, 1, 0]$$ to $$\hat{v}$$, and $$[0, 0, 1]$$ to $$\hat{w}$$.
Adjust the order of the columns as appropriate for the semantic meanings of the axes.

Rotation matrices are often familiar and easy to use.
However, they are over-constrained, using nine values with only three degrees of freedom.
From a pure data efficiency standpoint, three times as much storage is being used as seems necessary.
It is also not always obvious when a $$3 \times 3$$ matrix is a valid rotation matrix,
or how to correct an invalid matrix.

# Quaternions

Another common representation of a rotation is using a unit quaternion (that is, a four-dimensional complex number).
Quaternions consist of a real component, and three different roots of negative one, governed by the identity:

$$
\mathbf{i}^2 = \mathbf{j}^2 = \mathbf{k}^2 = \mathbf{ijk} = -1
$$

These can be intimidating to those unfamiliar, in particular the way multiplying $$\mathbf{i}$$, $$\mathbf{j}$$, and $$\mathbf{k}$$ is not commutative
(although remembering the similarity to rotation matrices, which are also not commutative can make this more intuitive).
Not every quaternion corresponds to a rotation; only unit quaternions, those whose squares sum to one.
Additionally, the negative of a quaternion represents the same rotation, $$q = -q$$.

In practice I've found it instructive, but mostly unnecessary to understand the details of quaternion mathematics to utilise them as rotations.
Instead, the easiest way to think of a unit quaternion is as an anti-clockwise rotation around an axis, defined as:

$$
\hat{q} = \cos{(\frac{\theta}{2})} + \sin{(\frac{\theta}{2})} (a_0 \mathbf{i} + a_1 \mathbf{j} + a_2 \mathbf{k})
$$

where $$\theta$$ is the angle of rotation around unit vector axis $$[a_0 a_1 a_2]$$.
This makes it more apparent why multiplying by negative one produces the same rotation:
flipping the axis and rotating clockwise instead ends up in the same place as the original anti-clockwise rotation.
Similarly, an inverse rotation is the conjugate of the rotation quaternion, or equivalently, flipping the direction of the axis of rotation
(those familiar with complex multiplication will recall that multiplying a complex number by its conjugate results in a purely real value,
which if it also has unit norm is necessarily $$1 + 0 \mathbf{i} + 0 \mathbf{j} + 0 \mathbf{k}$$ or
$$-1 + 0 \mathbf{i} + 0 \mathbf{j} + 0 \mathbf{k}$$, which are equivalent rotations).

Unit quaternions can be combined using the [Hamilton product](https://en.wikipedia.org/wiki/Quaternion#Hamilton_product), which is straightforward to implement,
but for which there are plenty of available library implementations
(for python I recommend [transforms3d](https://matthew-brett.github.io/transforms3d/)).
Similarly, a vector $$v_1 \mathbf{i} + v_2 \mathbf{j} + v_3 \mathbf{k}$$ can be rotated by a quaternion $$q$$ by multiplication by both the quaternion and its conjugate $$q v q'$$,
however again this usually readily available in library implementations.

Using a library largely means being able to treat the quaternion as a 4-vector,
and not having to worry about $$\mathbf{i}$$, $$\mathbf{j}$$, or $$\mathbf{k}$$ or how they combine.
The only real gotchas are remembering to keep the quaternion normalised, and to check equality against both $$q$$ and $$-q$$.
Note also that there are combinations of four values that cannot _quite_ be represented as four float32 values whose squares sum to one.
Repeated division by the norm will oscillate between values just above and just below 1. 

# Axis-Angle

A problem common to both rotation matrices and quaternions is dealing with addition and subtraction.
The group operations for rotation matrices and quaternions are forms of multiplication, and addition can produce invalid matrices or quaternions
(SO(3) is a Lie group, that is, a group that is also a manifold, and addition can produce values off the manifold).
This mostly comes up in the context of optimisation, where we might try to minimise the square error (which involves a subtraction),
or calculate a derivative (which involves adding infinitesimals).
In practice, gradient descent on a quaternion can try things like making the values much larger or smaller,
which is immediately undone by re-normalising back to length one, wasting most of the update step.

SO(3) is a Lie group, and every Lie group has a corresponding Lie algebra.
The Lie algebra for SO(3) is the set of skew-symmetric $$3 \times 3$$ matrices, which can be parameterised as:

$$
\begin{bmatrix}
0 & -z & y \\
z & 0 & -x \\
-y & x & 0 \\
\end{bmatrix}
$$

Note that there are only three different values in the matrix, and it can be expressed in terms of an $$\mathcal{R}^3$$ vector called an Euler vector.
An Euler vector is an axis-angle expression $$\theta \hat{a}$$, where $$\theta$$ is the angle of rotation and $$\hat{a}$$ us the unit vector around which the rotation occurs.

Because skew-symmetric matrices (and correspondingly, Euler vectors) are an algebra rather than a Lie group, they don't need to lie on a manifold.
This means that every such matrix or vector corresponds to a rotation, and operations like addition and differentiation still produce valid rotations.
Optimising an Euler vector is therefore much easier than other expressions of a rotation.
They can also be linearly interpolated to produce valid intermediate rotations.

A Euler vector $$e \in \mathcal{R}^3$$ rotates a vector $$v \in \mathcal{R}^3$$ using [Rodrigues' rotation formula](https://en.wikipedia.org/wiki/Rodrigues%27_rotation_formula).
First, separate the angle of rotation as the length of the vector from the axis of rotation, and then apply the formula for an axis and angle:

$$
\begin{align}
\theta &= \vert e \vert \\
\hat{a} &= \frac{e}{\vert e \vert} \\
v' &= v \cos{\theta} + (\hat{a} \times v) \sin{\theta} + \hat{a} (\hat{a} \cdot v) (1 - \cos{\theta})
\end{align}
$$

There are a couple of edge cases.
If $$e = [0, 0, 0]$$, then $$\hat{a}$$ is undefined but the angle of rotation is zero anyway, so the vector $$v$$ should be unchanged.
It is also worth noting that the Euler vectors $$e$$ and $$e + 2 \pi \hat{a}$$ represent the same rotation.
