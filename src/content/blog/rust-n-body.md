---
title: "From Equations to Orbits: Building an N-Body Simulator in Rust"
description: "How I went from not knowing Rust to simulating stable planetary systems at high speed using Rust's blend of safety and performance."
pubDate: 2025-09-24
tags: ["Rust", "Physics", "Simulation"]
draft: false
---

# A "small" performance test of Rust

For a long time, I had heard a lot of great stats and opinions about Rust, but never had the chance to really try it out on a project. I figured I would start out small with something maths and physics heavy, to test the performance against a language I know like Python or Javascript. There are many classic "small" projects that are often used to benchmark languages, and one of them is an N-body simulator.

The concept is straightforward: simulate multiple celestial bodies interacting through Newtonian gravity. However, the implementation and logic is not as it seems.

Repository: [Lucas8448/N-body-simulator](https://github.com/Lucas8448/N-body-simulator)

---

## The Physics Core

When I first looked at gravity equations, I'll admit — I was really intimidated. But once I (and ChatGPT Study Mode) broke it down step by step, it clicked. Here's how I understood it.

### Step 1: Understanding Gravity Between Two Objects

Newton figured out that any two objects with mass pull on each other. The force depends on three things:

1. How heavy the first object is (mass $m_i$)
2. How heavy the second object is (mass $m_j$)
3. How far apart they are (distance $r$)

The formula looks scary at first:

$$
F = G \frac{m_i m_j}{r^2}
$$

But let's decode it:

- $F$ is the strength of the pull between them
- $G$ is just a tiny constant (gravitational constant ≈ 6.674 × 10⁻¹¹ N·m²/kg² in SI units) that makes the units work out. In bigger simulations you often rescale units, but in this repository I kept the Sun–Earth example in meters, kilograms, and seconds so the familiar number drops straight in.
- The masses multiply together — heavier objects = stronger pull
- The distance is **squared** in the denominator — double the distance = quarter the force

The weirdest part for me was that $r^2$ in the denominator. It means gravity gets weaker really fast as things move apart.

### Step 2: Making It a Vector (Direction Matters)

That formula only gives us the _strength_ of the force. But force needs a direction too — is the object being pulled left? Up? Diagonally?

This is where it gets slightly more complex. We need a **unit vector** $\hat{\mathbf{r}}_{ij}$ — basically an arrow pointing from object $i$ to object $j$, with length 1.

The full gravity force becomes:

$$
\mathbf{F}_{ij} = G \frac{m_i m_j}{r^2} \hat{\mathbf{r}}_{ij}
$$

### Step 3: Adding Up All the Forces

Here's where it got messy for me at first. In a simulation with multiple bodies, _every_ object pulls on _every other_ object.

So for one object (let's call it body $i$), you have to:

1. Calculate the force from body 1 pulling on it
2. Calculate the force from body 2 pulling on it
3. Calculate the force from body 3 pulling on it
4. ...and so on for all other bodies

Then you **add all those force vectors together** to get the total force on body $i$:

$$
\mathbf{F}_{\text{total}} = \mathbf{F}_{i1} + \mathbf{F}_{i2} + \mathbf{F}_{i3} + \ldots
$$

Or in compact math notation:

$$
\mathbf{F}_{\text{total}} = \sum_{j \ne i} \mathbf{F}_{ij}
$$

(The $\sum$ symbol just means "add them all up." The $j \ne i$ means "skip yourself — objects don't pull on themselves.", very logical from a programming perspective.)

### Step 4: From Force to Acceleration

Remember Newton's second law: $F = ma$? We can rearrange it:

$$
a = \frac{F}{m}
$$

So once we have the total force on an object, we divide by its mass to get its **acceleration** — how much its velocity changes:

$$
\mathbf{a}_i = \frac{1}{m_i} \sum_{j \ne i} \mathbf{F}_{ij}
$$

Heavier objects accelerate less for the same force. A tiny moon gets whipped around easily. A massive star barely budges.

### The Computational Cost

Here's the catch: if you have $n$ bodies, you need to compute $n \times (n-1)$ force pairs. That's **O(n²)** — it scales badly. 10 bodies = 90 calculations. 100 bodies = 9,900 calculations. Every single timestep.

For my first version, I kept it simple and brute-forced the O(n²) approach. It works fine for small systems (~100 bodies). Bigger simulations need tricks like Barnes–Hut trees, but I wasn't ready for that rabbit hole yet.

### The Integration Problem

So I had the physics down. I knew how to calculate forces. The next question was simple to ask but hard to get right: **how do you actually move things forward in time?**

My first pass was the textbook Euler integrator:

```
new_velocity = old_velocity + acceleration * timestep
new_position = old_position + velocity * timestep
```

It works, but it also steadily leaks energy into the system. Those runaway orbits were a good reminder that "simple" does not mean "physically accurate."

For this first pass I stayed with the simplest option, plain **forward Euler integration**. It is easy to reason about and only needs the acceleration from the current timestep:

```rust
fn step(&mut self) {
    let n = self.bodies.len();
    let mut accelerations = vec![[0.0, 0.0]; n];

    for i in 0..n {
        for j in (i + 1)..n {
            let dx = self.bodies[j].pos[0] - self.bodies[i].pos[0];
            let dy = self.bodies[j].pos[1] - self.bodies[i].pos[1];
            let dist_sq = dx * dx + dy * dy;
            let dist = dist_sq.sqrt();

            if dist == 0.0 {
                continue;
            }

            let force = self.g * self.bodies[i].mass * self.bodies[j].mass / dist_sq;
            let fx = force * dx / dist;
            let fy = force * dy / dist;

            accelerations[i][0] += fx / self.bodies[i].mass;
            accelerations[i][1] += fy / self.bodies[i].mass;
            accelerations[j][0] -= fx / self.bodies[j].mass;
            accelerations[j][1] -= fy / self.bodies[j].mass;
        }
    }

    for i in 0..n {
        self.bodies[i].vel[0] += accelerations[i][0] * self.dt;
        self.bodies[i].vel[1] += accelerations[i][1] * self.dt;
        self.bodies[i].pos[0] += self.bodies[i].vel[0] * self.dt;
        self.bodies[i].pos[1] += self.bodies[i].vel[1] * self.dt;
    }
}
```

Euler does introduce numerical drift, especially for orbits, so long-term stability is shaky. I kept a short simulation horizon in this repository and logged each step to make the drift obvious. Upgrading to a symplectic integrator like leapfrog is high on the “next iteration” list, but I wanted to keep the baseline version honest about what it currently does.

---

## Rust's Role: Safe Speed

This is where I really got to see Rust shine. Even with a minimalist `Body` struct that stores positions and velocities as `[f64; 2]` arrays, the ownership rules nudged me toward a tight double loop that shares work between body pairs and avoids aliasing bugs.

Key lessons from Rust here:

- **Borrowing rules** naturally led to the `(i, j)` pairing in the snippet above, so each interaction is computed exactly once.
- The compiler was happy to optimize the naive math in release mode; I did not need unsafe tricks to get reasonable performance.
- Pattern matching and enums would make future refactors (e.g., switching to a `Vector2` type) straightforward without sacrificing safety.

---

## Stability, Scale, and Surprises

The most surprising part wasn't getting it to _run_—it was making it run **stably**.

With Euler integration, picking a timestep is always a compromise. The example in the repository uses `dt = 10.0` seconds and just two bodies (Sun and Earth) expressed in SI units. A thousand steps only covers about 10,000 seconds—barely a sliver of an Earth year—so you see just the first hint of the orbital arc before numerical drift creeps in.

I do not use gravitational softening yet; the code simply skips interaction if two bodies occupy the exact same spot. That keeps the force finite but effectively lets bodies pass through each other. More elaborate scenarios or larger systems would need both a better integrator and a softening term, so those are on the backlog for the next version.