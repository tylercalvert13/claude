---
name: remotion
description: Create and develop Remotion video projects. Use when the user wants to create videos programmatically with React, scaffold a new Remotion project, add Remotion to an existing React app, build video compositions with animations and transitions, handle media (images, video, audio, fonts), or render videos to MP4/WebM.
---

# Remotion Video Development

Remotion is a React framework for creating videos programmatically. Videos are React components rendered frame-by-frame. All visual content is built with JSX, CSS, Canvas, SVG, or WebGL, and all animations must be driven by `useCurrentFrame()`.

## Project Setup

### New Project

```bash
npx create-video@latest
```

Choose a template (Hello World, Blank, Next.js, React Three Fiber, TikTok, etc.). The resulting project structure:

```
src/
  index.ts          # Entry point, calls registerRoot()
  Root.tsx           # Registers <Composition> components
  MyComposition.tsx  # Video component
public/             # Static assets (images, fonts, audio)
remotion.config.ts  # Remotion configuration
package.json
```

Start Remotion Studio:

```bash
npm start
```

Opens at `localhost:3000` with a timeline-based editor for previewing compositions.

### Adding to an Existing Project

```bash
npm i remotion @remotion/cli
```

Create the entry point (`src/remotion/index.ts`):

```tsx
import { registerRoot } from "remotion";
import { Root } from "./Root";
registerRoot(Root);
```

Create `src/remotion/Root.tsx` with your compositions (see below). Create `remotion.config.ts` at the project root.

## Core Concepts

### Compositions

A `<Composition>` registers a renderable video in `Root.tsx`:

```tsx
import { Composition } from "remotion";
import { MyVideo } from "./MyVideo";

export const Root: React.FC = () => {
  return (
    <>
      <Composition
        id="MyVideo"
        component={MyVideo}
        durationInFrames={150}
        fps={30}
        width={1920}
        height={1080}
        defaultProps={{ title: "Hello" }}
      />
    </>
  );
};
```

- `<Still>` — single-frame image, no `durationInFrames` or `fps` needed
- `<Folder>` — organizes compositions in the Studio sidebar

Use `type` (not `interface`) for component props to ensure `defaultProps` type safety.

### The Frame-Based Model

```tsx
import { useCurrentFrame, useVideoConfig } from "remotion";

const frame = useCurrentFrame();           // Current frame (0-indexed)
const { fps, durationInFrames, width, height } = useVideoConfig();
```

Convert seconds to frames: `const durationFrames = 2 * fps;`

**CRITICAL**: All animations MUST use `useCurrentFrame()`. Never use CSS transitions, CSS animations, `@keyframes`, or Tailwind animation classes — they will not render correctly and will cause flickering.

## Animation and Timing

### interpolate()

Maps input ranges to output ranges:

```tsx
import { useCurrentFrame, interpolate } from "remotion";

const frame = useCurrentFrame();
const opacity = interpolate(frame, [0, 30], [0, 1], {
  extrapolateLeft: "clamp",
  extrapolateRight: "clamp",
});
```

Always use `extrapolateLeft: "clamp"` and/or `extrapolateRight: "clamp"` to prevent values from going out of range.

Use `interpolateColors()` for color transitions:

```tsx
import { interpolateColors } from "remotion";

const color = interpolateColors(frame, [0, 60], ["#ff0000", "#0000ff"]);
```

### spring()

Physics-based animation from 0 to 1:

```tsx
import { useCurrentFrame, useVideoConfig, spring } from "remotion";

const frame = useCurrentFrame();
const { fps } = useVideoConfig();
const scale = spring({ frame, fps, config: { damping: 200 } });
```

Common spring configs:
- `{ damping: 200 }` — smooth, no bounce
- `{ damping: 20, stiffness: 200 }` — snappy
- `{ damping: 8 }` — bouncy
- `{ damping: 15, stiffness: 80, mass: 2 }` — heavy/slow

Supports `delay` (in frames) and `durationInFrames` to stretch. Use `measureSpring()` to calculate how many frames a spring animation takes.

### Easing

```tsx
import { Easing } from "remotion";
```

Combine modifiers with curves: `Easing.in(Easing.quad)`, `Easing.out(Easing.exp)`, `Easing.inOut(Easing.circle)`. Custom curves: `Easing.bezier(x1, y1, x2, y2)`.

Pass to `interpolate()` as `{ easing: Easing.out(Easing.quad) }`.

## Sequencing and Layout

### Sequence

```tsx
import { Sequence } from "remotion";

<Sequence from={30} durationInFrames={60}>
  <MyScene />
</Sequence>
```

- `from` — frame at which children appear
- `durationInFrames` — how long children are mounted
- `useCurrentFrame()` inside a Sequence returns the **local** frame (resets to 0)
- Sequences nest: a Sequence at frame 60 inside one at frame 30 starts at global frame 90
- Default wraps children in `<AbsoluteFill>`. Use `layout="none"` to disable
- Use `premountFor` prop to preload components before they appear

### Series

Arranges children sequentially without manual `from` calculation:

```tsx
import { Series } from "remotion";

<Series>
  <Series.Sequence durationInFrames={60}><SceneA /></Series.Sequence>
  <Series.Sequence durationInFrames={90}><SceneB /></Series.Sequence>
  <Series.Sequence durationInFrames={45}><SceneC /></Series.Sequence>
</Series>
```

Use negative `offset` on a `<Series.Sequence>` to create overlaps.

### AbsoluteFill

Shorthand for a `<div>` with `position: absolute; top: 0; left: 0; right: 0; bottom: 0`. Use for backgrounds and layered content.

## Transitions

Install:

```bash
npx remotion add @remotion/transitions
```

```tsx
import { TransitionSeries, linearTiming } from "@remotion/transitions";
import { fade } from "@remotion/transitions/fade";

<TransitionSeries>
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneA />
  </TransitionSeries.Sequence>
  <TransitionSeries.Transition
    presentation={fade()}
    timing={linearTiming({ durationInFrames: 15 })}
  />
  <TransitionSeries.Sequence durationInFrames={60}>
    <SceneB />
  </TransitionSeries.Sequence>
</TransitionSeries>
```

Available presentations (each imported from its own subpath):

```tsx
import { fade } from "@remotion/transitions/fade";
import { slide } from "@remotion/transitions/slide";       // directions: "from-left", "from-right", "from-top", "from-bottom"
import { wipe } from "@remotion/transitions/wipe";         // directions: "from-left", "from-top-left", "from-top", etc.
import { flip } from "@remotion/transitions/flip";
import { clockWipe } from "@remotion/transitions/clock-wipe";
```

Timing options:

```tsx
import { linearTiming, springTiming } from "@remotion/transitions";

linearTiming({ durationInFrames: 20 });
springTiming({ config: { damping: 200 }, durationInFrames: 25 });
```

**Duration math**: Transitions overlap scenes, reducing total duration. Two 60-frame sequences with a 15-frame transition = 105 total frames (not 120).

## Media

### Images

Always use `<Img>` from `remotion` (NOT `<img>`, NOT Next.js `<Image>`, NOT CSS `background-image`):

```tsx
import { Img, staticFile } from "remotion";

<Img src={staticFile("photo.png")} />
<Img src="https://example.com/photo.png" />
```

For animated GIFs: `npm i @remotion/gif`, then use `<Gif>` from `@remotion/gif`.

### Video

```bash
npx remotion add @remotion/media
```

```tsx
import { Video } from "@remotion/media";
import { staticFile } from "remotion";

<Video src={staticFile("clip.mp4")} />
```

Props: `startFrom`, `endAt` (in frames), `playbackRate`, `volume`, `muted`, `loop`.

### Audio

```tsx
import { Audio } from "@remotion/media";
import { staticFile, interpolate } from "remotion";

<Audio
  src={staticFile("music.mp3")}
  volume={(f) =>
    interpolate(f, [0, 30], [0, 1], { extrapolateRight: "clamp" })
  }
/>
```

Same props as Video. Wrap in `<Sequence>` to delay start. Multiple audio tracks layer automatically.

### Fonts

Google Fonts:

```bash
npx remotion add @remotion/google-fonts
```

```tsx
import { loadFont } from "@remotion/google-fonts/Lobster";
const { fontFamily } = loadFont();
// Use in style: { fontFamily }
```

Local fonts:

```bash
npx remotion add @remotion/fonts
```

```tsx
import { loadFont } from "@remotion/fonts";
import { staticFile } from "remotion";

loadFont({ family: "MyFont", url: staticFile("MyFont.woff2") });
```

### Static Files

Place all assets in the `public/` folder. Reference them with `staticFile("filename.ext")` from `remotion`. Never use relative imports or `require()` for media files.

## Parametrized Videos

Use Zod schemas to make compositions accept typed, editable props:

```bash
npm i zod
npx remotion add @remotion/zod-types
```

```tsx
import { z } from "zod";
import { zColor } from "@remotion/zod-types";

export const mySchema = z.object({
  title: z.string(),
  color: zColor(),
});

type MyProps = z.infer<typeof mySchema>;
```

Pass `schema` and `defaultProps` to `<Composition>`:

```tsx
<Composition
  id="MyVideo"
  component={MyVideo}
  schema={mySchema}
  defaultProps={{ title: "Hello", color: "#ff0000" }}
  durationInFrames={150}
  fps={30}
  width={1920}
  height={1080}
/>
```

This enables visual editing of props in the Remotion Studio sidebar. The `defaultProps` must be JSON-serializable (no functions, classes, or non-serializable values).

## Player Embedding

Embed interactive Remotion videos in any React app:

```bash
npm i @remotion/player
```

```tsx
import { Player } from "@remotion/player";

<Player
  component={MyVideo}
  compositionWidth={1920}
  compositionHeight={1080}
  durationInFrames={150}
  fps={30}
  controls
  style={{ width: 800 }}
  inputProps={{ title: "Hello" }}
/>
```

Do NOT wrap the component in `<Composition>` when using Player. Use `component` for direct imports or `lazyComponent` (wrapped in `useCallback`) for lazy loading.

Imperative API via ref: `play()`, `pause()`, `seekTo()`, `requestFullscreen()`, `getScale()`.

## Rendering

### CLI Rendering

```bash
npx remotion render src/index.ts MyComposition out/video.mp4
```

Options:
- `--codec` — `h264` (default), `h265`, `vp8`, `vp9`, `prores`, `png`, `gif`
- `--crf` — quality (lower = better, default varies by codec)
- `--scale` — scale factor (e.g., `0.5` for half resolution)
- `--image-format` — `jpeg` or `png` (png for transparency)

Render a still image:

```bash
npx remotion still src/index.ts MyStill out/thumbnail.png
```

### Programmatic Rendering

```bash
npm i @remotion/renderer
```

```tsx
import { bundle } from "@remotion/bundler";
import { renderMedia, selectComposition } from "@remotion/renderer";

const bundled = await bundle({ entryPoint: "src/index.ts" });
const composition = await selectComposition({ serveUrl: bundled, id: "MyVideo" });
await renderMedia({ composition, serveUrl: bundled, codec: "h264", outputLocation: "out/video.mp4" });
```

### Cloud Rendering

- AWS Lambda: `@remotion/lambda`
- Google Cloud Run: `@remotion/cloudrun`

See Remotion docs for cloud setup guides.

## Workflow

Make a todo list for all the tasks in this workflow and work on them one after another.

### 1. Clarify the Goal

Ask the user what kind of video they want to create:
- Video type (social media clip, data visualization, animated explainer, promotional, etc.)
- Target dimensions (1920x1080 for YouTube/landscape, 1080x1080 for Instagram, 1080x1920 for TikTok/Reels)
- Duration and fps (typically 30 fps)
- Content requirements (text, images, video clips, audio, data sources)

### 2. Set Up the Project

If no Remotion project exists, scaffold with `npx create-video@latest`. If adding to an existing project, install packages and create the entry point structure. Verify `remotion.config.ts` and `src/Root.tsx` exist.

### 3. Design the Composition Structure

Plan scenes and their timing. Define each scene as a separate React component. Decide on transitions between scenes. Register all compositions in `Root.tsx` with correct dimensions, fps, and duration.

### 4. Implement Scene Components

Build each scene using `useCurrentFrame()` and `useVideoConfig()`. Use `interpolate()` and `spring()` for animations. Use `<Img>`, `<Video>`, `<Audio>` for media. Style with inline styles or CSS modules.

### 5. Compose the Timeline

Use `<Sequence>`, `<Series>`, or `<TransitionSeries>` to arrange scenes in order. Add transitions between scenes. Verify total frame count matches the registered composition duration.

### 6. Add Parametrization (if needed)

Define a Zod schema for any props the user wants to customize. Pass the schema and defaults to `<Composition>`.

### 7. Preview in Studio

Run `npm start` to open Remotion Studio. Verify timing, animations, and visual output. Check for flickering or blank frames.

### 8. Render the Video

```bash
npx remotion render src/index.ts <CompositionId> out/video.mp4
```

Choose the appropriate codec and quality settings for the target platform.

## Critical Rules

- All animations MUST be driven by `useCurrentFrame()`. CSS transitions, CSS animations, `@keyframes`, and Tailwind animation classes will NOT render correctly.
- Always use `<Img>` from `remotion` for images. Never use `<img>`, Next.js `<Image>`, or CSS `background-image` — Remotion cannot track their loading state.
- Always use `<Video>` and `<Audio>` from `@remotion/media`. Never use native `<video>` or `<audio>` tags.
- Use `staticFile()` to reference assets in the `public/` folder. Never use relative imports or `require()` for media files.
- Keep all `remotion` and `@remotion/*` packages at the same version. Remove `^` from version numbers in `package.json` to prevent mismatches.
- Always add `extrapolateLeft: "clamp"` and/or `extrapolateRight: "clamp"` to `interpolate()` calls to prevent values from going out of range.
- Use `type` (not `interface`) for component props when using Zod schemas.
- `defaultProps` must be JSON-serializable. No functions, classes, or non-serializable values.
- Use `premountFor` on `<Sequence>` to preload components and avoid blank frames during loading.

## Wrap-Up

After completing the workflow, provide the user with:

- Summary of files created or modified
- Preview command: `npm start`
- Render command: `npx remotion render src/index.ts <CompositionId> out/video.mp4`
- Suggestions for next steps (adding more scenes, parametrization, cloud rendering, embedding with Player)
