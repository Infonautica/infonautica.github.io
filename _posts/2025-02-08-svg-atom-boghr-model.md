---
layout: post
title: "Generate SVG Bohr Model of an atom"
date: 2025-02-08 12:00:00 +0
categories: blog
---

Once i decided to re-create periodic table of elements in web. One particular feature i really wanted to implement is a visualization of an atom. After brief searching i figured that there are existing solutions, but they’re old and problematic to use, so i decided to make one from scratch.

<!--more-->

![Simplified visualization of an atom with electrons spinning around it](/img/post/atom.gif)

Please note that steps below do not include installation commands, file structures and similar smaller details.

### Step 1. Choose how to render

There are obviously many ways to render something dynamically in the browsers:

- Pure HTML and CSS
- JS Canvas
- SVG

I decided to go with SVG as it gives reliable results with very little code.

### Step 2. Get basic structure

Each an every atom has a nucleus, at least one electron orbit and at least one electron. More complex atoms are simply an extension with more orbits and more electrons.

I didn’t have any software to draw it myself, so i simply asked ChatGPT:

> Generate PUG code that renders an svg of bohr representation of atom of helium

And received this elegant solution:

```pug
svg(width='200', height='200', viewBox='0 0 200 200', xmlns='http://www.w3.org/2000/svg')
  // Nucleus
  circle(cx='100', cy='100', r='20', fill='#FFCC00')
  // Electron orbit as path
  path(d='M 20,100 A 80,80 0 1 1 180,100 A 80,80 0 1 1 20,100', fill='none', stroke='black', 'stroke-width'='1')
  // Electrons
  circle(cx='20', cy='100', r='5', fill='black')
  circle(cx='180', cy='100', r='5', fill='black')
```

From this template it’s easy to add more orbits and electrons.

### Step 3. Bring in JavaScript

In order to have a full control and flexibility of the rendering i decided to transfer code generation from PUG to JS.

I started with defining a set of smaller functions that provide me different parts of the image:

```typescript
// Generate the main SVG node
function generateSVG() {
  const svgNode = document.createElementNS("http://www.w3.org/2000/svg", "svg");
  svgNode.classList.add("atom_svg");

  svgNode.setAttribute("viewBox", "0 0 500 500");

  return svgNode;
}

// Generate nucleus that would sit in the middle of the picture
function generateNucleus() {
  const nucleusNode = document.createElementNS(
    "http://www.w3.org/2000/svg",
    "circle",
  );
  nucleusNode.setAttribute("class", "atom_nucleus");
  nucleusNode.setAttribute("r", "15");

  nucleusNode.setAttribute("cx", "250");
  nucleusNode.setAttribute("cy", "250");

  return nucleusNode;
}

// Generate a small circle representing an electron
function generateElectron() {
  const electronNode = document.createElementNS(
    "http://www.w3.org/2000/svg",
    "circle",
  );
  electronNode.setAttribute("class", "atom_electron");
  electronNode.setAttribute("r", "5");

  return electronNode;
}
```

And a slightly more difficult part that renders orbits with different radius based on the number (1 is the closest to the nucleus, 2 is second from it, and so on)

```typescript
function generateOrbit(orbitNumber: number) {
  const orbitNode = document.createElementNS(
    "http://www.w3.org/2000/svg",
    "path",
  );
  orbitNode.setAttribute("class", "atom_orbit");
  orbitNode.setAttribute("id", `orbit_${orbitNumber}`);

  // Distance between each orbit
  const spacing = 30;

  // Path commands
  const moveToStart = `M ${250 - spacing * orbitNumber},250`;

  const arcRadius = spacing * orbitNumber;
  const archFirstHalf = `A ${arcRadius},${arcRadius} 0 1 1 ${250 + arcRadius},250`;
  const archSecondHalf = `A ${arcRadius},${arcRadius} 0 1 1 ${250 - arcRadius},250`;

  orbitNode.setAttribute(
    "d",
    `${moveToStart} ${archFirstHalf} ${archSecondHalf}`,
  );

  return orbitNode;
}
```

But before combining it, i needed data about every atom.

### Step 4. Get data

I copied some data from the internet, adjusted it to a desired shape and added a couple of convenient getter functions. Result of this later was extracted to its own NPM package:

[https://www.npmjs.com/package/simple-periodic-table-data](https://www.npmjs.com/package/simple-periodic-table-data)

Most importantly it had the definition for how many electrons should be presented on each of the orbits.

### Step 5. Style with SCSS

I eventually moved styles from inline to SCSS and added some extra (like a glow for electrons using shadow):

```scss
$color_nucleus: #22ddcc;
$color_electron: #7df9ff;
$color_electronGlow: #fff;
$color_orbit: #ccc;

.atom {
  display: flex;
  flex-direction: row;
  justify-content: center;

  &_nucleus {
    fill: $color_nucleus;
    filter: drop-shadow(0 0 5px $color_electronGlow);
  }

  &_orbit {
    stroke: $color_orbit;
    stroke-width: 1px;
    fill: none;
  }

  &_electron {
    fill: $color_electron;
    fill-opacity: 1;
    fill-rule: evenodd;
    stroke-width: 1px;
    stroke-linecap: butt;
    stroke-linejoin: miter;
    stroke-opacity: 1;
    filter: drop-shadow(0 0 5px $color_electronGlow);
  }
}
```

### Step 6. Combine it together

All parts are in place and now i needed the main function to orchestrate generation:

```typescript
export function renderAtom(options: RenderAtomOptions) {
  const element = getChemElement(options.elementPeriodicNumber);

  const container = document.querySelector(".atom");
  container.innerHTML = "";

  const svg = generateSVG();
  svg.innerHTML = "";

  container.append(svg);

  // Add Nucleus
  const nucleusNode = generateNucleus();
  svg.appendChild(nucleusNode);

  element.electronConfig.forEach((electronsCount, index) => {
    const orbitNumber = index + 1;

    // Add Orbit
    const orbitNode = generateOrbit(orbitNumber);
    svg.appendChild(orbitNode);

    // Add electrons
    const electronsCollection = new Array(electronsCount).fill(null);
    const electrons = electronsCollection.map(() => generateElectron());

    // Attach electrons to SVG
    electrons.forEach((electronNode) => {
      svg.appendChild(electronNode);
    });
  });
}
```

At this stage electrons are defined in 0, 0 coordinates and not aligned on the orbits. In order to solve this problem, while avoiding complex mathematics of figuring out their positions, plus to animate electrons i used “gsap” package and its “MotionPath” plugin.

Please note that there are other ways to animate SVG:

- Using CSS animations
- Using `<animateTransform>` tag

### Step 7. Animate

GSAP is a very powerful, flexible and easy to use package. Here’s what little i had to do:

Import the package, plugin and enable it:

```typescript
import { gsap } from "gsap";
import { MotionPathPlugin } from "gsap/MotionPathPlugin";

gsap.registerPlugin(MotionPathPlugin);
```

Then in the part where i iterate over electron configuration i defined a dedicated timeline for each of the orbits. This allows me to have full control of the behavior and do things like random direction (clockwise or counter-clockwise), different speeds and so on:

```typescript
const electronTimeline = gsap.timeline({
  repeat: -1,
  repeatDelay: 0,
});
```

And then i animate freshly generated electrons using simple API:

```typescript
// Add electrons
const electronsCollection = new Array(electronsCount).fill(null);
const electrons = electronsCollection.map(() => generateElectron());

// Attach electrons to SVG
electrons.forEach((electronNode) => {
  svg.appendChild(electronNode);
});

const animationDuration = 10;

// Animate electrons on the orbit
electronTimeline.to(electrons, {
  duration: animationDuration,
  ease: "none",
  transformOrigin: "-50% -50%",
  stagger: {
    // This is the part that sets electrons equally apart from each other
    each: animationDuration / electronsCount,
    repeat: -1,
  },
  motionPath: {
    path: `#orbit_${orbitNumber}`,
  },
});

// Skip the first iteration of animation to avoid orbits population phase
electronTimeline.seek(animationDuration);
```

Eventually i added some minor things to make it look more organic. For example randomized animation direction and duration:

```typescript
const isReversed = Math.random() > 0.4;

// Animate electrons
const minimumDuration = 6;
const maximumDuration = 15;

// Randomize values in range from min to max
const animationDuration =
  Math.floor(Math.random() * (maximumDuration - minimumDuration + 1)) +
  minimumDuration;

// Animate electrons on the orbit
electronTimeline.to(electrons, {
  duration: animationDuration,
  ease: "none",
  transformOrigin: "-50% -50%",
  stagger: {
    each: animationDuration / electronsCount,
    repeat: -1,
  },
  motionPath: {
    path: `#orbit_${orbitNumber}`,
    start: isReversed ? 1 : 0,
    end: isReversed ? 0 : 1,
  },
});
```

You can check the full and final [code on GitHub](https://github.com/Infonautica/render-atom-bohr-js) or play with it [here](https://chem.infonautica.pro/)

### Step 8. Wrap-up and publish

After that i added ability to configure some of the properties and published it to NPM:

[https://www.npmjs.com/package/render-atom-bohr-js](https://www.npmjs.com/package/render-atom-bohr-js)

So now the usage can be simple as:

```typescript
import { renderAtom } from "render-atom-bohr-js";

renderAtom({
  elementPeriodicNumber: 20,
  containerSelector: "#atom",
});
```

## Summary

I’m really glad that i decided to do it on my own as i got to learn GSAP animation and plugins, some specifics of atomic structure, how paths are rendered in SVG and many more things.

But most importantly i hope this will help someone in the future.
