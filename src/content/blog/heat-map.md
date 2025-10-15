---
title: "Simulating Heat Diffusion in Rust"
description: "A deep dive into how I built a 2D heat simulation — complete with real physics, interactive heating, and glowing steel colors — entirely in Rust."
pubDate: 2025-10-15
tags: ["Rust", "Physics", "Simulation", "Macroquad", "Heat Transfer"]
draft: false
---

While exploring physics beyond mechanics, I wanted to see how **heat** actually moves through materials — not as equations on paper, but visually.  
That curiosity led me to build a **2D heat diffusion simulator** in Rust using **Macroquad**, based on the real heat equation used in thermodynamics.

---

## The Physics Behind It

Heat spreads from hot regions to cold ones.  
In continuous form, that’s expressed by the **heat equation**:

$$
\frac{\partial T}{\partial t} = \alpha \left(
\frac{\partial^2 T}{\partial x^2} +
\frac{\partial^2 T}{\partial y^2}
\right)
$$

Here:

- $ T(x, y, t)$: temperature at each point,
- $\alpha$: thermal diffusivity — how quickly heat spreads.

To simulate it on a computer, I discretized the space into a grid and replaced derivatives with finite differences:

$$
T_{new}(x, y) = T(x, y) + \alpha \, \big[
T_{L} + T_{R} + T_{U} + T_{D} - 4T(x, y)
\big]
$$

That one line drives the entire simulation.

---

## From Theory to Code

I started by making a 2D grid in Rust — a simple `Vec<Vec<f32>>` where each element is a temperature.  
Then, on every frame, I applied the diffusion formula above to compute the next grid.  
To make sure it looked alive, I visualized each cell as a colored rectangle using [**macroquad**](https://github.com/not-fl3/macroquad).

```toml
[dependencies]
macroquad = "0.4"
```

Once I saw a red spot fading into blue, I knew it worked.

---

## Adding Interaction

Next, I made it interactive.
By checking the mouse position each frame, I could inject heat wherever I clicked — the longer I held, the hotter it got.
The code looked like this:

```rust
if is_mouse_button_down(MouseButton::Left) {
    let (mx, my) = mouse_position();
    let gx = (mx / SCALE) as i32;
    let gy = (my / SCALE) as i32;
    grid[gy as usize][gx as usize] += 0.02;
}
```

That tiny block transformed the simulation from a passive demo into a playground.

---

## Cooling and Realistic Physics

In reality, heat doesn’t just spread — it also **fades** into the environment.
To capture that, I added a simple cooling term:

$$
T_{new} = T - \beta (T - T_{ambient})
$$

I used real thermal properties of **steel**:

- Thermal diffusivity ( $\kappa = 1.18 \times 10^{-5} , m^2/s$ )
- Grid spacing ( $\Delta x = 1 , \text{cm}$ )
- Time step ( $\Delta t = 0.5 , \text{s}$ )

and set

$$
\alpha = \kappa \frac{\Delta t}{\Delta x^2} \approx 0.059
$$

This gave physically stable and realistic heat flow. Every frame now represented half a second of real time.

---

## Making It Beautiful

I wanted it to _look_ like real glowing metal.
Cold steel has a bluish tint, while hot steel glows red, orange, then white.
So I mapped temperature values to those hues and added a metallic gray background.
Suddenly, the screen looked like a steel plate heating up under a torch.

Here’s the result in motion:
drag your cursor across the plate, and watch the glow spread and fade like molten iron cooling.

---

## Full Source Code

You can find the complete code here:
👉 [**2d-heat-diffusion**](https://github.com/Lucas8448/2d-heat-diffusion)
