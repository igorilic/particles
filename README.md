# Particles

A real-time, interactive 3D particle system controlled by your webcam. Hand gestures and facial expressions reshape 4,000 additive-blended particles into spheres, clouds, text, a flapping butterfly, or a smiling/frowning skull — all in a single HTML file.

![demo](https://img.shields.io/badge/three.js-0.160-000?logo=three.js)
![demo](https://img.shields.io/badge/mediapipe-hands%20%2B%20face_mesh-4285F4)
![demo](https://img.shields.io/badge/single--file-HTML-orange)

## Features

| Trigger | Effect |
|---|---|
| ✊ **Fist** (0 fingers extended) | Particles contract into a tight sphere |
| ✋ **Open palm** (3+ fingers extended) | Particles spread into a wide cloud |
| ☝️ **One finger** (index only) | Forms the text **"hello, world"** |
| ✌️ **Two fingers** (index + middle) | Forms the text **"I'am Qwen"** |
| 🖖 **Four fingers** (no thumb) | 💥 Outward explosion burst, then re-forms into **"Hello Najdan!"** |
| 🦋 **Butterfly hands** (two hands, thumbs touching, fingers spread) | Forms a butterfly silhouette with flapping wings |
| 💀 **No hands + face visible** | Forms a human skull whose mouth mirrors your expression — 🙂 smile or 🙁 frown |
| *Partial openness* | Sphere ↔ cloud interpolates continuously with your palm openness |

All shapes are rendered with 3D rotation, additive-blended sprite particles, and a spring-toward-target physics loop, so transitions feel fluid and react in real time.

### Customizing the text

Click **⚙ Customize text** in the top-right to set your own strings for the three text-forming gestures. Changes regenerate the particle targets live (no reload) and persist across sessions via `localStorage`. Hit **Reset defaults** to clear them.

## Stack

- **[Three.js 0.160](https://threejs.org/)** — particle rendering (`THREE.Points`, additive blending, per-particle HSL colors, soft sprite texture)
- **[MediaPipe Hands](https://developers.google.com/mediapipe/solutions/vision/hand_landmarker)** — up to 2 hands, 21 landmarks each
- **[MediaPipe Face Mesh](https://developers.google.com/mediapipe/solutions/vision/face_landmarker)** — 468 face landmarks for smile/frown classification
- Single self-contained `index.html`, no build step, all dependencies loaded via CDN

## Running locally

Camera APIs require `localhost` or HTTPS — they will not work from a `file://` URL.

```sh
cd particles
python3 -m http.server 8000
# then open http://localhost:8000
```

Or any other static server (`npx serve`, `caddy file-server`, etc).

On first load, click **Start Camera** and allow webcam permission.

## How it works

### Particle system
4,000 particles are stored in a `THREE.BufferGeometry` with `position` and `color` attributes. Every frame, each particle is pulled toward a per-mode target with a damped spring:

```js
v = (v + (target - pos) * stiffness * dt) * damping
pos += v * dt
```

Stiffness is `20` for sphere/cloud, `30` for text/butterfly/skull, and drops to `0.5` during the 120ms explosion window so the burst is visible before particles snap back into place.

### Shape targets
Every non-physics shape (text, butterfly, skull) is produced by **rendering the shape to an offscreen 2D canvas and sampling the bright pixels** into world coordinates. This is the same trick for all of them — change the canvas drawing and you have a new shape.

- **Text** — `ctx.fillText(...)` with a large bold font
- **Butterfly** — body + four wing-lobe ellipses + curved antennae, in the XY plane. Wing flap is applied per-frame as `z += sin(t) * |x| * amplitude` so wing tips move most.
- **Skull** — cranium and jaw filled, then `globalCompositeOperation = 'destination-out'` cuts the eye sockets, nose triangle, and the smile/frown mouth curve

### Gesture classification

#### Hands
- Per finger, a tip is "extended" when its distance from the wrist exceeds its PIP joint's distance by ≥5%
- Counted across index/middle/ring/pinky to drive mode selection
- 2-hand butterfly fires when both thumb tips are within `0.9 ×` average palm size and each hand has at least 3 fingers spread
- A 1-frame debounce smooths mode transitions

#### Face
- `smileScore = (mouthCenterY − cornersY) / faceHeight` from landmarks 61/291 (corners), 13/14 (mouth center), 10/152 (head/chin)
- Asymmetric thresholds with hysteresis (`> 0.012` happy, `< -0.004` sad) prevent flicker at neutral

### Frame loop
Both Hands and Face Mesh run on every camera frame. The hands callback owns mode selection when hands are visible; when both hands are out of frame, it falls back to skull mode if a face was seen within the last 1.5s, otherwise relaxes to a sphere.

## Project layout

```
particles/
└── index.html    # everything — Three.js scene, MediaPipe wiring, target generators
```

## License

MIT
