**Introduction**

**Play for free at https://solutionsandprojects.nl**

In a quiet hidden pool lives a small adventurous boat who spends its days gliding through shimmering water waves and drifting sound waves.  
Whenever you **left-click and move your mouse**, the water begins to ripple and the air vibrates, forming waves the little boat loves to dance upon. If the chorus of sound becomes too loud, you can silence it with a soft tap on the S-key.

Curious glowing balls appear whenever you **right-click on the water**. They float gently, waiting to be nudged. A normal bump from the boat gives them a simple push—but when timing truly matters, the boat can release its rare **Witchblitz spark by striking with the spacebar**, sending the balls shooting forward with magical power. Choose your angle well, because a perfectly placed Witchblitz shot can turn the tide.

To explore the pool, you can steer the boat with the arrow keys, weaving between waves, balls, and echoes. And if you ever wish to see life from the boat’s own point of view, **press C to slip into his perspective**—press it again to return to the peaceful view above the pool.

You can zoom and rotate the world with your mouse, letting you watch every swirl, splash, and rolling crest of the waves as they spread across the water.

With waves to ride, mysteries to uncover, and magic to control, the little boat is ready for adventure.  
**All he’s waiting for… is you. Play alone or in competition with your friends.**


**How the waves “texture” works**\
**What it is**\
The waves are driven by a dynamic heightfield stored in a GPU texture called heightmap.
Internally it’s an RGBA float texture where:
x = height at current frame.
y = height at previous frame.
z, w = unused.
**Where it’s created**
In initWater():
gpuCompute = new GPUComputationRenderer(WIDTH, WIDTH, renderer);
const heightmap0 = gpuCompute.createTexture();
fillTexture(heightmap0); initializes the texture with simplex noise.
heightmapVariable = gpuCompute.addVariable('heightmap', shaderChange.heightmap_frag, heightmap0);
Uniforms on the compute material are set (mousePos, mouseSize, viscosity, deep) and 
gpuCompute.init()
 prepares ping‑pong render targets.
**How it updates each frame (compute shader)**
Each animation frame:
js
gpuCompute.compute();
tmpHeightmap = gpuCompute.getCurrentRenderTarget(heightmapVariable).texture;
if (waterMesh) waterMesh.material.heightmap = tmpHeightmap;
The compute shader shaderChange.heightmap_frag evolves the heightfield (discrete wave equation with damping):
glsl
// neighbors
vec4 north = texture2D( heightmap, uv + vec2( 0.0,  cellSize.y ) );
vec4 south = texture2D( heightmap, uv + vec2( 0.0, -cellSize.y ) );
vec4 east  = texture2D( heightmap, uv + vec2(  cellSize.x, 0.0 ) );
vec4 west  = texture2D( heightmap, uv + vec2( -cellSize.x, 0.0 ) );

// current & previous heights
vec4 heightmapValue = texture2D( heightmap, uv );

// wave propagation + damping
float newHeight = ( (north.x + south.x + east.x + west.x) * 0.5 - heightmapValue.y ) * viscosity;

// mouse disturbance (injects a ripple)
float mousePhase = clamp(length((uv - vec2(0.5)) * BOUNDS - vec2(mousePos.x, -mousePos.y)) * PI / mouseSize, 0.0, PI);
newHeight -= (cos(mousePhase) + 1.0) * deep;

// ping-pong time integration
heightmapValue.y = heightmapValue.x; // previous = current
heightmapValue.x = newHeight;        // current = new
gl_FragColor = heightmapValue;
This uses a ping‑pong scheme (two render targets) to store current and previous height and compute the next height from neighbor samples.
**How the height “texture” is used to render**
WaterMaterial extends THREE.MeshStandardMaterial and injects the heightmap into the vertex shader via onBeforeCompile:
Adds uniform sampler2D heightmap;
Replaces chunks to:
Sample height for displacement:
glsl
float heightValue = texture2D( heightmap, uv ).x;
vec3 transformed = vec3( position.x, position.y, heightValue );
Approximate normals from height differences (central differences):
glsl
vec2 cellSize = vec2( 1.0 / WIDTH, 1.0 / WIDTH );
vec3 objectNormal = vec3(
  ( texture2D( heightmap, uv + vec2( -cellSize.x, 0 ) ).x - texture2D( heightmap, uv + vec2( cellSize.x, 0 ) ).x ) * WIDTH / BOUNDS,
  ( texture2D( heightmap, uv + vec2( 0, -cellSize.y ) ).x - texture2D( heightmap, uv + vec2( 0,  cellSize.y ) ).x ) * WIDTH / BOUNDS,
  1.0
);
The mesh is a plane rotated to lie flat; using heightValue as the vertical axis produces visible waves. The normal affects PBR lighting and reflections.
**Mouse interaction**
Pointer events set mousePos in raycast() when the mouse is down, injecting energy in the compute shader around the intersection point to create ripples.
**Optional smoothing**
A separate smoothShader is provided (5‑tap average). The helper smoothWater() ping‑pongs between the two targets to low‑pass filter the heightmap, but it’s not called by default.
**Parameters**
 WIDTH controls simulation resolution (also the plane segmentation).
 BOUNDS maps world units to UVs for sampling neighbors and mouse injection.
 viscosity, deep, mouseSize tune damping and disturbance strength/size.
In short: the “texture for waves” is a GPU-updated heightfield computed every frame with a wave equation, then sampled in the vertex shader to displace the water surface and derive normals for lighting.

**How the water interacts with objects (level + normals)**
**What it is**
Ships and balls read the water height and local slope to align/orient and to detect transitions between air and water.
This uses a tiny “readback” compute shader `readWaterLevelFragmentShader` that samples the current heightmap at a given UV and encodes:
- water level (x)
- surface normal components (nx in pixel 2, ny in pixel 3)
into 4 pixels of a 4×1 render target, which are read on CPU with `renderer.readRenderTargetPixels`.
**Where it’s used**
- `shipDynamics()` updates each ship: converts world (x,z) to UV, renders the readback shader, decodes height and normal, tilts the ship and applies buoyancy/drag.
- Balls do the same when bouncing/transitioning between `air` and `water` states to decide splash-in/out and resting height.

**How the sound “waves” work**
**What it is**
The project includes a small Web Audio synth + step sequencer. Sound is not analyzed to drive water; instead, both water and sound respond to the same user input (pointer over the pool).
**Components**
- `audioCtx` with a `masterGain` and a `reverbBus` + `Convolver` (impulse response is generated at startup).
- A 16‑step scheduler (“kick”, “snare”, “hat”, and a bell voice) that runs only while a chord is active.
- Per‑note voices are simple oscillators through a low‑pass filter into master + reverb sends.
**Interaction**
- Left‑click/hold over the water plays a chord chosen by `chordForPoint(x,z)` (maps pool position to scale degrees and octave). Moving the mouse morphs the chord; releasing stops it.
- Press `S` to toggle all audio on/off. Toggling off ramps gains down, stops scheduled events/voices, disconnects buses, and may suspend the context; toggling on reconnects and resumes.
**Relation to water**
- The water simulation continues regardless of audio state. There is no audio‑driven displacement; the heightfield is driven by the compute shader and user/ship inputs. Sound and water are synchronized only via shared pointer interactions.

