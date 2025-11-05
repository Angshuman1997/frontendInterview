# How does the browser parse an HTML page (rendering pipeline)

1. Quick answer

Rendering pipeline: network fetch → parse HTML → construct DOM → parse CSS → construct CSSOM → create render tree → layout (reflow) → paint → composite. Scripts and styles can block parsing.

2. Steps explained

- Fetch: browser requests HTML and resources.
- Parse HTML: build the DOM tree.
- Parse CSS: build CSSOM; external CSS blocks rendering until loaded.
- Render tree: combine DOM + CSSOM to determine visible nodes and styles.
- Layout (reflow): compute geometry for each node.
- Paint: rasterize pixels for each layer.
- Composite: combine layers onto the screen.

3. Blocking behavior

- External CSS blocks rendering to avoid FOUC (flash of unstyled content).
- Scripts without `defer`/`async` block parsing.

4. Optimizations

- Inline critical CSS; defer non-critical CSS.
- Use `defer` for scripts that don’t need to run during parsing.
- Use `async` for independent scripts.

5. Interview-ready summary

The browser fetches and parses HTML/CSS, builds DOM and CSSOM, then creates a render tree. It computes layout, paints, and composites. Network, CSS, and synchronous scripts can block different stages of this pipeline.
