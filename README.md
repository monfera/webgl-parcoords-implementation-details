# plotly.js parcoords implementation details

[Live example](https://codepen.io/monfera/full/dNBwOv/)

Parcoords are popular for visualizing and filtering multivariate datasets. Often, not only the dimensions are numerous, but also, the data points, each rendered as a polyline, can be numerous. If there's a requirement for a large number of lines, e.g. thousands to hundreds of thousands on 2014..2017 era laptops, then SVG rendering of the lines is not an option due to its slowness. The speed requirements don't just necessarily concern the initial rendering; there's also filtering and column reordering, possibly performed smoothly, rather than abruptly rerendering _after_ the user gesture ended.

While this writeup talks about optimizations, there may well be better ways for achieving the same ends. Suggestions welcome!

## Why not SVG

As mentioned, it's slow - which has multiple constituents:

- DOM manipulation is slow
- it's possible to use [just a few DOM elements for a large number of glyphs](https://codepen.io/monfera/pen/rLYqWR) (lines or sections) but it'll still be slow, so it's not just DOM scenegraph element count but also their complexity that matters
- interestingly, the mere presence of large DOM structures can slow down interactions even if the interactions impact very few, very simple elements
- SVG CSS properties such as opacity [aren't yet hardware accelerated](https://codepen.io/monfera/pen/JKOqyY), unlike [HTML](https://codepen.io/monfera/pen/qNVzBm) ones although it'll [improve](https://bugs.chromium.org/p/chromium/issues/detail?id=666244) in Chrome
- (not SVG specific but part of the slowness) rendering a large number of lines is inherently time consuming

## Why not 2D Canvas

Interestingly, there are constellations where a 2D canvas can draw lines faster than it's possible with WebGL lines or quads. Specifically, Chrome does a fantastic job if line thickness is exactly 1 pixel and there are few `ctx.stroke()` calls (in practical terms, there are relatively few distinct colors). I think the reason is either that

- the browser implementation uses a special [line drawing algorithm](https://en.wikipedia.org/wiki/Line_drawing_algorithm) in this case - lines of one pixel thickness can be better optimized
- WebGL performance seems to correlate with the area of the _bounding box_ of lines, ie. such line drawing optimization wasn't observed, not even with `GL.LINE` which more recently got constrained to a pixel width of 1
- and/or Chromium is free to use native DirectX / OpenGL facilities for 2D graphics, if present

While the color palette issue could be solved, as humans can't distinguish too many gradients, what killed this avenue was that *browser implementations varied wildly in Canvas2D line drawing speed even on the same hardware and OS*.

## Why not software based aggregation

While it's possible to pre-rasterize on the CPU side, lightening the GPU load, it just removes inherently parallel workload from the GPU - which is good at it - and burdens the CPU with it. For example, on filtering interaction, an arbitrary number of lines can be added or worse, removed, making re-rasterizations expensive. Also, even for an initial render, a software rasterizer will be slow. I stopped short of implementing a fast line drawing algorithm in JavaScript because it's a lot of pixels. Splatting and similar clumping approaches were also rejected.

## WebGL

Ruling out these other things, the natural candidate then is WebGL! It needs to be said that there's no magic, drawing a lot of lines, ie. a **lot** of pixels is expensive no matter what, so even the settled approach is fill rate limited on constrained platforms.

### Why not WebGL via an abstraction layer

There are parcoords implementations that use some library that maps Canvas2D-like operations to WebGL drawing operations. This approach results in WebGL rendering but doesn't bring much benefit over Canvas2D because Canvas2D itself is free to take advantage of present, native OpenGL/DirectX features. So this proves the experience that while WebGL _may_ perform way faster than anything else, it requires a careful working around of the platform capabilities and constraints. In other words, a JS-written Canvas2D-like implementation using WebGL is bound to be slower than a native, C++ implementation using the much faster OpenGL or DirectX backing.

### The plotly.js approach

#### Choice of library: [regl](http://regl.party/)

To remain close to the metal and have a direct relationship with what WebGL API function is called and when, the work is done at a low level. For example, `three.js` was not chosen because, while it handles higher levels of abstraction such as scenegraph and geometry representation, it also distances from the raw metal, ie. it may be harder to iterate on speed.

As direct WebGL work needs a lot of broilerplate for handling resources, we chose `regl` to avoid much of this error prone work. The `regl` API is shallow enough to have an understanding of what's going on at the WebGL API level, so it's easier to iterate on various approaches.

We at Plotly were also going to eventually switch to `regl` for various reasons, most importantly, its quality and suitability for these tasks.

#### Layering

There are the following notable layers, from far to near. 

Optimizations: 

- the parts that must be scalable, ie. that draw lines, are done with WebGL, but the annotations and affordances are rendered with SVG - this saves on development time and code size, and allow future alternative line renderers
- for this reason, the line renderer is fully separated in file source from the SVG renderer
- each layer is only redraw or modified as needed

##### 1. (WebGL) context layer 

It shows all lines in alpha blended black, ie. white->gray->black densities form that show distribution. 

Optimizations: 

- This layer doesn't respond to axis filters (doesn't redraw) because the lines are stationary
- While it still needs to be initially rendered, and rerendered upon column drag&drop, these are only done if at least some axis filters constrain the retained set, otherwise the context layer is fully occluded
- Optimizations that apply to the focus layer

##### 2. (WebGL) focus layer 

This shows the retained set, which is the entire set if there are no constraints applied (no axis domain filtering manually with the magenta bars or via the `constraintrange` attribute).

Concepts:

- Panel: a rectangular area between two adjacent axes
- Full render: clearing all panels and redrawing all lines initially, or upon axis brushing interaction
- Retina screen: for simplicity, a screen (area) where the `devicePixelRatio` is more than one
- Block: a unit of drawing activity that's done synchronously

Draw optimizations:

- Limited clearing and redrawing. It's often the case with interactive dataviz that a user interaction is best acted upon immediately, eg. moving the things during drag&drop gesture, but the impact is local, ie. it's enough to redraw smaller parts. For example, dragging a parcoords axis for reordering can change, at most, two _panels_. So dragging just clears and then redraws the one or two panels that sandwich the axis. This must work for the context layer too if filters apply.
- Per panel drawing: the above item needed this, but there are other benefits with this: the concept of what's rendered (parcoords lines) can be separated from the rendering substrate (a particular WebGL instance backed by a `<canvas>` element) - in other words, future new plots can render onto the same exact layer peacefully. This is important as the user would quickly run out of the 16 `gl` contexts that are commonly available.
- Incremental drawing of the lines on full redraw: fast GPUs can trivially render a moderate number of lines in one rAF loop. But the requirement called for non-blocking UI on intel-equipped laptops. On a _full_ redraw, the current code draws a maximum of 5000 lines in one `rAF`. I've experimented with adaptively changing it by measuring time, but haven't yet got a non-intrusive performance sampler, which would be important as eagerly measuring rendering throughput is itself costly. (Why? Reported `performance.now` or `rAF` function arg times are high resolution, but they measure 0 because WebGL rendering is done asynchronously, and changes are batched. Since pipeline flushing etc. don't generally work in WebGL, enforcing draw can be done via `readPixels` only, which, done often enough, degrades performance, in part because it breaks up batches. An adaptive sampler would only measure upfront, or rarely).
- Incremental drawing also implies that the screen buffer must be kept, ie. it's initialized with `preserveDrawingBuffer`
- Asynchronous block clearing / rendering: to reap the benefits from the above item, ie. ensure the UI is not blocking despite high throughput work going on, the blocks are queued and processed asynchronously. This means that by the time a block is rendered, maybe the user interacted with the system, spoiling the scheduled block. In this case, the block is canceled based on a composite key (eg. which panel).
- Fixed 1px line width: this allows the use of `gl.LINES`; the WebGL standard permits an implementation to support just one width (one pixel), which the recent browsers exploit (formerly, line thicknesses could go 4..16). A larger apparent width could be supported via drawing two elongated triangles, but it'd be a different vertex geometry, making the GPU based crossfiltering harder if not impossible (see separate section on crossfiltering). 
- Not exploiting retina screens: to exploit these would require blowing up the raster size by 2x2 times, ending up with 4 times that many pixels. Not only would it be much slower, the 1px width lines would be less salient, contrary to stated needs. [This example](https://codepen.io/monfera/pen/apWqgM) shows how the maximal resolution looks on Retina screens.

Geometry related optimizations:

- `gl.LINES` - the above mentioned optimization of using lines also saves on GPU memory, though it's not a bottleneck now
- All three layers draw from same exact geometry buffer. This has the desired future utility of sharing GPU-side data among plots on a dashboard, which is especially useful for crossfiltering
- Both Z-order and coloring work from the same vector; the actual color isn't in the arrayBuffer - instead, it's looked up from a texture that serves as the palette. The benefits: 1) it saves on attribs (a scarce resource as the requirements demanded a _lot_ of dimensions so attrib width is maxed out already), 2) recoloring, e.g. due to changing the color `domain` on the plot, does _not_ need geometry changes, only the palette needs to change and the focus layer is redrawn

GPU crossfiltering, an optimization:

The crossfiltering is done on the GPU, which is suitable for the task, due to how filtering can be achieved by applying linear transformations, which parallelize well. The alternative, CPU based filtering would require that the geometry, ie. a lot of data, be resent to the GPU continuosly, every frame. This might make the interactions CPU->GPU memory bandwidth and latency limited.

To minimize numerical imprecision issues, all dimension domains are normalized to to a range of [0, 1]. 100% precision isn't attainable, but the axis sliders are shown rasterized too so minor misselection is not a problem. Parcoords in general are for analyzing datasets in holistic manner rather than exact, tabular reporting. Still, precision can be increased similar to [simulated double floats](https://github.com/gl-vis/gl-scatter2d-fancy/compare/65f41ab5dd7eb17b033a63283b04f6b76dcb478c...3e41bb61b76e2c0125d8629b3de343cd4f7a084a)

Crossfiltering is done by clamping the minimum-maximum of the data hypercube with the current constraints and comparing the resulting hypercube with the original data hypercube. Where the two don't match, ie. a data point's value doesn't equal to its clamped value, the point is perturbed such that it's outside the frustrum and therefore won't burden the fragment shader at all.

In the future, it's desirable that all `gl` plots on a dashboard can share one crossfiltered dataset on the GPU.

##### 3. (WebGL) pick layer 

This uses the customary method where an otherwise human-invisible layer is rendered with a bijective mapping between the data point identity (e.g. index number) and the `rgb` or `rgba` value of a pixel. This requires that this layer be _not_ antialiased as antialiasing would tamper with the `rgba` channels.

Optimizations:

- This is a separate layer - ie. it can come and go independently of the visible layers, and the other way around (ie. each layer rendered just when needed)
- As drawing the pick layer is expensive, it's debounced - visible lines are redrawn on user interaction such as column drag&drop or brushing, which all make hover picking unneeded (the user can't brush and hover at the same time), and some dozens or even a couple of hundred milliseconds are not perceptible to the user, as it takes a bit of time to move the mouse from the axis brush to a line of interest

##### 4. (SVG) annotation layer 

Optimization:

- Development time: not worrying about forming a geometry from a Canvas2D rasterized text, then triangulating it for the few ticks rendered is a big saving in coding time
- Code size: this reduces code a lot; these text geom libraries are huge
- Runtime: text rasterization, readout to/from canvas and geometry calculation takes time, and loading the above libraries also add to payload size
- Robustness: Canvas2D rendering goes ahead without complaint even if the required font isn't yet loaded, resulting in default font types, something we otherwise need to protect against or rule out

Note: some of these also apply with other approaches eg. SDF based text rendering. In short, why render text with WebGL if a DOM layer is adequate? (3D annotations can be different if they live in data space and must be occluded in proper model Z.)

## Upshot

Most of the optimizations and approaches are not parcoords-specific, so it'd be useful to support across all 2D (maybe 3D too) `gl` plots:

- incremental and local clearing/rendering with non-blocking scheduling and invalidation
- one shared set of layers (eg. context+focus+pick+annotation would work for all plots)
- restyling large numbers of points without resending geometry (as with palette texture) - eg. crossfiltering and user initiated restyling needs it
- common model shared across multiple plots and trace types
- crossfiltering on the GPU: fast and shareable with all compatible `gl` plots on the dashboard

Examples:

- A [half-baked SPLOM PoC](https://codepen.io/monfera/full/pRwmwb/) on the basis of parcoords (a few lines of code change in `parcoords`) 
- Showing that glyph drawing can be separated from the concept of the model: 
