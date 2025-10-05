---
title: "From Vectors to Gravity: Building a Mini Physics Engine in Rust"
description: "How I started with simple vector math and ended up with bouncing, colliding particles under gravity — all in pure Rust."
pubDate: 2025-10-05
tags: ["Rust", "Physics", "Simulation"]
draft: false
---

While studying physics, i came across a central part of the subject, vector math. Vectors are fundamental in representing quantities that have both magnitude and direction, such as velocity, acceleration, and force. This sparked an idea: what if I could build a simple physics engine from scratch using Rust?

---

## Starting with Vectors

The core building block is the **2D vector**. A vector has both magnitude and direction, and can be represented as

$$
\mathbf{v} = \begin{bmatrix} x \\ y \end{bmatrix}
$$

Basic operations are simple but powerful:

- **Addition**

  $$
  \mathbf{a} + \mathbf{b} =
  \begin{bmatrix}
  a_x + b_x \\
  a_y + b_y
  \end{bmatrix}
  $$

- **Subtraction**

  $$
  \mathbf{a} - \mathbf{b} =
  \begin{bmatrix}
  a_x - b_x \\
  a_y - b_y
  \end{bmatrix}
  $$

- **Magnitude** (length)

  $$
  |\mathbf{v}| = \sqrt{x^2 + y^2}
  $$

- **Dot product**
  $$
  \mathbf{a} \cdot \mathbf{b} = a_x b_x + a_y b_y
  $$

I wrote a simple `Vec2` struct in Rust with these methods. Once I had that, I could start simulating **motion**:

$$
\mathbf{p}\_{t+1} = \mathbf{p}\_t + \mathbf{v}\_t \, \Delta t.
$$

---

## Bouncing Particles

My first simulation was just a set of particles moving inside a box and **bouncing off the walls**. Each particle has a position, velocity, and radius. On each frame:

1. Update position with velocity.
2. If it goes past a wall, invert the velocity component and clamp it back inside.

This gave me satisfying little bouncing dots — but they went _through each other_. Time to fix that.

---

## Elastic Collisions with Impulses

Naively swapping velocities when particles overlap causes chaos — velocities explode because they keep “colliding” every frame. Real physics engines solve this using **impulses**.

When two particles collide, you compute a collision impulse along the **normal** between them:

$$
j = \frac{-(1 + e)(\mathbf{v}_r \cdot \mathbf{n})}{\frac{1}{m_1} + \frac{1}{m_2}},
$$

where

- $\mathbf{v}_r = \mathbf{v}_1 - \mathbf{v}_2$ is the relative velocity,
- $\mathbf{n}$ is the normalized collision normal,
- $e$ is the restitution (bounciness).

Then apply equal and opposite impulses:

$$
\mathbf{v}\_1' = \mathbf{v}\_1 + \frac{j}{m_1}\mathbf{n}
$$

$$
\mathbf{v}\_2' = \mathbf{v}\_2 - \frac{j}{m_2}\mathbf{n}
$$

For equal masses and perfectly elastic collisions ($e = 1$), this simplifies beautifully and gives stable, realistic behavior.

I implemented this in a simple nested loop, resolving each pair of particles once per frame. No runaway speeds, no jittering — just clean bounces.

---

## Moving Beyond ASCII: Macroquad

Originally, I printed particles to the terminal using ASCII grids. It worked, but flickered horribly. The next step was using [**macroquad**](https://github.com/not-fl3/macroquad) — a lightweight graphics library for Rust. It only took a few lines to draw smooth moving circles in a window at 60 FPS.

```toml
# Cargo.toml
[dependencies]
macroquad = "0.4"
```

```rust
draw_circle(
    self.pos.x as f32,
    self.pos.y as f32,
    self.radius as f32,
    self.color,
);
```

Suddenly, my physics engine looked like a real sandbox.

---

## Adding Gravity

The final touch (so far) was adding **gravity** and **restitution**. Since gravity is just a constant acceleration and restitution is a measure of how much energy is conserved in a collision, all I needed to do was:

```rust
const GRAVITY: f64 = 500.0;
const RESTITUTION: f64 = 0.8;

self.vel.y += GRAVITY * dt;

// on every wall collision, and change y based on if it was a vertical or horizontal wall
self.vel.y = -self.vel.y * RESTITUTION;
```

Particles now fall toward the bottom of the screen, collide, bounce, and settle. A simple restitution coefficient makes them lose a bit of energy on each bounce, so it feels natural.

---

What I love about this approach is that every feature builds directly on **vector math and Newton’s laws** — no magic, just physics. At some point i will have to try to combine this with my N-body-simulator project to have particles attract each other with gravity too.

---

## Full Source Code

You can find the full code on my GitHub:
👉 [**vectors**](https://github.com/Lucas8448/vectors)
