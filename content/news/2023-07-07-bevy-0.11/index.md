+++
title = "Bevy 0.11"
date = 2023-07-07
[extra]
author = "Bevy Contributors"
+++

Thanks to **X** contributors, **X** pull requests, community reviewers, and our [**generous sponsors**](/community/donate), we're happy to announce the **Bevy 0.11** release on [crates.io](https://crates.io/crates/bevy)!

For those who don't know, Bevy is a refreshingly simple data-driven game engine built in Rust. You can check out our [Quick Start Guide](/learn/book/getting-started/) to try it today. It's free and open source forever! You can grab the full [source code](https://github.com/bevyengine/bevy) on GitHub. Check out [Bevy Assets](https://bevyengine.org/assets) for a collection of community-developed plugins, games, and learning resources.

To update an existing Bevy App or Plugin to **Bevy 0.11**, check out our [0.10 to 0.11 Migration Guide](/learn/migration-guides/0.10-0.11/).

Since our last release a few months ago we've added a _ton_ of new features, bug fixes, and quality of life tweaks, but here are some of the highlights:

<!-- more -->

* **Morph targets**: Vertex-based animations
* **Parallax mapping**: Materials now support an optional depth map, giving
  flat surfaces a feel of depth through parallaxing the material's textures.

## Morph Targets

<div class="release-feature-authors">authors: @nicopap, @cart</div>

Bevy, since the 0.7 release, supports 3D animations.

But it only supported _skeletal_ animations. Leaving on the sidewalk a common
animation type called _morph targets_ (aka blendshapes, aka keyshapes, and a slew
of other name). This is the grandparent of all 3D character animation!
[Crash Bandicoot]'s run cycle used morph targets.

<video controls><source src="morph_targets_video.mp4" type="video/mp4"/></video>
<div style="font-size: 1.0rem" class="release-feature-authors">Character model by <a href="https://www.artstation.com/zambrah">Samuel Rosario</a> (© all rights reserved), used with permission. Modified by nicopap, using the <a href="https://studio.blender.org/characters/snow/v2/">Snow</a> character texture by Demeter Dzadik for Blender Studios <a href="https://creativecommons.org/licenses/by/4.0/">(🅯 CC-BY)</a>.
</div>
<!-- The previous paragraph requires the <a href> tags, since zola doesn't
process markdown markup within tags -->

Nowadays, an animation artist will typically use a skeleton rig for wide
moves and morph targets to clean up the detailed movements.

When it comes to game assets, however, the complex skeleton rigs used by
artists for faces and hands are too heavy. Usually, the poses are
"baked" into morph poses, and facial expression transitions are handled
in the engine through morph targets.

Morph targets is a very simple animation method. Take a model, have a base
vertex position, move the vertices around to create several poses:

<div style="flex-direction:row;display:flex;justify-content:space-evenly">
<div style="display:flex;flex-direction:column;align-items:center;width:20%"><p><b>Default</b></p><img alt="A wireframe rendering of a character's face with a neutral expression" src="default-pose-bw.png"></div>
<div style="display:flex;flex-direction:column;align-items:center;width:20%"><p><b>Frown</b></p><img alt="Wireframe rendering of a frowning character" src="frown-pose-bw.png"></div>
<div style="display:flex;flex-direction:column;align-items:center;width:20%"><p><b>Smirk</b></p><img alt="Wireframe rendering of a smirking character" src="smirk-pose-bw.png"></div>
</div>

Store those poses as a difference between the default base mesh and the variant
pose, then, at runtime, _mix_ each pose. Now that we have the difference with
the base mesh, we can get the variant pose by simply adding to the base
vertices positions.

That's it, the morph target shader looks like this:

```rust
fn morph_vertex(vertex: Vertex) {
    for (var i: u32 = 0u; i < pose_count(); i++) {
        let weight = weight_for_pose(i);
        vertex.position += weight * get_difference(vertex.index, position_offset, i);
        vertex.normal += weight * get_difference(vertex.index, normal_offset, i);
    }
}
```

In Bevy, we store the weights per pose in the `MorphWeights` component.

```rust
fn set_weights_system(mut morph_weights: Query<&mut MorphWeights>) {
    for mut entity_weights in &mut morph_weights {
        let weights = entity_weights.weights_mut();

        weights[0] = 0.5;
        weights[1] = 0.25;
    }
}
```

Now assuming that we have two morph targets, (1) the frown pose, (2)
the smirk pose:

<div style="flex-direction:row;display:flex;justify-content:space-evenly">
<div style="display:flex;flex-direction:column;align-items:center;width:12%">
  <p><b>[0.0, 0.0]</b></p>
  <p style="margin:0;font-size:75%">default pose</p>
  <img alt="Neutral face expression" src="morph_target_default-0.png">
</div>
<div style="display:flex;flex-direction:column;align-items:center;width:12%">
  <p><b>[1.0, 0.0]</b></p>
  <p style="margin:0;font-size:75%">frown only</p>
  <img alt="Frowning" src="morph_target_frown-0.png">
</div>
<div style="display:flex;flex-direction:column;align-items:center;width:12%">
  <p><b>[0.0, 1.0]</b></p>
  <p style="margin:0;font-size:75%">smirk only</p>
  <img alt="Smirking" src="morph_target_smirk.png">
</div>
<div style="display:flex;flex-direction:column;align-items:center;width:12%">
  <p><b>[0.5, 0.0]</b></p>
  <p style="margin:0;font-size:75%">half frown</p>
  <img alt="Slightly frowning" src="morph_target_frown-half-0.png">
</div>
<div style="display:flex;flex-direction:column;align-items:center;width:12%">
  <p><b>[1.0, 1.0]</b></p>
  <p style="margin:0;font-size:75%">both at max</p>
  <img alt="Making faces" src="morph_target_both-0.png">
</div>
<div style="display:flex;flex-direction:column;align-items:center;width:12%">
  <p><b>[0.5, 0.25]</b></p>
  <p style="margin:0;font-size:75%">bit of both</p>
  <img alt="Slightly frowning/smirking" src="morph_target_smirk-quarter-frown-half-0.png">
</div>
</div>

While conceptually simple, it requires communicating to the GPU a tremendous
amount of data. Thousand of vertices, each 288 bits, several model variations,
sometimes a hundred.

Bevy's morph target implementation is similar to BabyloneJS's. We store the
vertex data as pixels in a 3D texture. This allows morph targets to not only
run on WebGPU, but also on the WebGL2 wgpu backend.

This could be improved in a number of ways, but it is sufficient for an
initial implementation.

<video controls><source src="morph_target_smirk.mp4" type="video/mp4"/></video>

[Crash Bandicoot]: https://en.wikipedia.org/wiki/Crash_Bandicoot_(video_game)#Gameplay

## Parallax Mapping

<div class="release-feature-authors">author: @nicopap</div>

Bevy now supports parallax mapping and depth maps. Parallax mapping puts normal
maps to shame when it comes to giving "illusion of depth" to a material. The top half of this video uses parallax mapping plus a normal map, whereas the bottom half only uses a normal map:

<video controls loop><source alt="a rotating view of the earth, the top half of the screen uses parallax mapping, while the bottom half does not" src="earth-parallax.webm" type="video/webm"/></video>
<div style="font-size: 1.0rem" class="release-feature-authors">earth view, elevation & night view by NASA (public domain)</div>

Notice how it is not merely the shading of pixels that changes, but their
actual position on screen. The mountaintops hide mountain ridges behind
themselves. High mountains move faster than coastal areas.

Parallax mapping moves pixels according to the perspective and depth on the
surface of the geometry. Adding true 3D depth to flat surfaces.

All of that, without adding a single vertex to the geometry. The whole globe
has exactly 648 vertices. Unlike a more primitive shader, such as displacement
mapping, parallax mapping only requires an additional grayscale image, called
the `depth_map`.

Games often use parallax mapping for cobblestones or brick walls, so
let's make a brick wall in Bevy! First, we spawn a mesh:

```rust
commands.spawn(PbrBundle {
    mesh: meshes.add(shape::Box::new(30.0, 10.0, 1.0).into()),
    material: materials.add(StandardMaterial {
        base_color: Color::WHITE,
        ..default()
    }),
    ..default()
});
```

![A 3D desert scene with two flat white walls and a pebble path winding between them](parallax_mapping_none.jpg)

Of course, it's just a flat white box, we didn't add any texture.
So let's add a normal map:

```rust
normal_map_texture: Some(assets.load("normal_map.png")),
```

![The same scene with normal maps](parallax_mapping_normals.jpg)

This is much better. The shading changes according to the light direction too!
However, the specular highlights on the corner are overbearing, almost noisy.

Let's see how a depth map can help:

```rust
depth_map: Some(assets.load("depth_map.png")),
```

![The same scene with a depth texture](parallax_mapping_depth.jpg)

We eliminated the noise! There is also that sweet 3D feel reminiscent of
90's games pre-rendered cinematic sequences.

So what's going on, why does parallax mapping eliminate the ugly specular
lights on the wall?

This is because parallax mapping insets the ridges between bricks, so that they
are occluded by the bricks themselves.

![Illustration of the previous paragraph](ridge-light-view-1.svg)

Since normal maps do not "move" the shaded areas, merely shade them
differently, we get those awkward specular highlights. With parallax mapping,
they are gone.

![A montage of the three preceding images, contrasting each effect](parallax_mapping_compare.jpg)

Parallax mapping in Bevy is still very limited. The most painful aspect is that
it is not a standard glTF feature, meaning that the depth texture needs to be
programmatically added to materials if they came from a GLTF file.

Additionally, parallax mapping is incompatible with the temporal antialiasing
shader, doesn't work well on curved surfaces, and doesn't affect object's
silhouettes.

However, those are not fundamental limitations of parallax mapping, and may be
fixed in the future.

## WebGPU Support

## Deref Derive Attribute

<div class="release-feature-authors">authors: @MrGVSV</div>

Bevy code tends to make heavy use of the [newtype](https://doc.rust-lang.org/rust-by-example/generics/new_types.html) pattern,
which is why we have dedicated derives for [`Deref`](https://docs.rs/bevy/latest/bevy/prelude/derive.Deref.html) and [`DerefMut`](https://docs.rs/bevy/latest/bevy/prelude/derive.DerefMut.html).

This previously only worked for structs with a single field:

```rust
#[derive(Resource, Deref, DerefMut)]
struct Score(i32);
```

For 0.11, we've improved these derives by adding the `#[deref]` attribute, which allows them to be used on structs with multiple fields.
This makes working with generic newtypes much easier:

```rust
#[derive(Component, Deref, DerefMut)]
struct Health<T: Character> {
    #[deref] // <- use the `health` field as the `Deref` and `DerefMut` target
    health: u16,
    _character_type: PhantomData<T>,
}
```

<div class="release-feature-authors">authors: @mockersf, many others throughout Bevy's development</div>

![webgpu](webgpu.svg)

Bevy now supports WebGPU rendering on the web (in addition to WebGL 2). WebGPU support is still rolling out, but if you have [a supported web browser][webgpu-support] you can explore our new [live WebGPU examples](/examples-webgpu) page.

### What is WebGPU?

WebGPU is an [exciting new web standard](https://github.com/gpuweb/gpuweb) for doing modern GPU graphics and compute. It takes inspiration from Vulkan, Direct3D 12, and Metal. In fact, it is generally implemented on top of these APIs under the hood. WebGPU gives us access to more GPU features than WebGL2 (such as compute shaders) and also has the potential to be much faster. It means that more of Bevy's native renderer features are now also available on the web. It also uses the new [WGSL shader language](https://www.w3.org/TR/WGSL). We're very happy with how WGSL has evolved over time and Bevy uses it internally for our shaders. We also added usability features like imports! But with Bevy you still have the option to use GLSL if you prefer.

### How it Works

Bevy is built on top of the [wgpu] library, which is a modern low-level GPU API that can target pretty much every popular API: Vulkan, Direct3D 12, Metal, OpenGL, WebGL2, and WebGPU. The best backend API is selected for a given platform. It is a "native" rendering API, but it generally follows the WebGPU terminology and API design. Unlike WebGPU, it can provide direct access to the native APIs, which means Bevy [enjoys a "best of all worlds" situation](/news/bevy-webgpu/#how-it-works).

### WebGPU Examples

Click one of the images below to check out our live WebGPU examples (if your [browser supports it][webgpu-support]):

[![webgpu examples](webgpu_examples.png)](examples-webgpu)

[wgpu]: https://github.com/gfx-rs/wgpu
[webgpu-support]: https://caniuse.com/webgpu

## Improved Shader Imports

<div class="release-feature-authors">authors: @robtfm</div>

Bevy's rendering engine has a lot of great options and features. For example, the PBR `StandardMaterial` pipeline supports desktop/webgpu and webgl, 6 optional mesh attributes, 4 optional textures, and a plethora of optional features like fog, skinning, and alpha blending modes, with more coming in every release.

Many feature combos need specialized shader variants, and with over 3000 lines of shader code split over 50 files in total, the text-substitution-based shader processor was beginning to creak at the seams.

This release we've switched to using [naga_oil](https://github.com/bevyengine/naga_oil), which gives us a module-based shader framework. It compiles each file individually to naga's IR and then combines them into a final shader on demand. This doesn't have much visible impact yet, but it does give a few immediate benefits:

* The engine's shader code is easier to navigate and less magical. Previously there was only a single global scope, so items could be referenced even if they were only imported indirectly. This sometimes made it hard to locate the actual code behind the reference. Now items must be explicitly imported, so you can always tell where a variable or function originated just by looking at the current file: <br/>
![imported items](imported_items.png)
* Shaders now have codespan reporting, an error will point you to the shader file and line number, preventing a lot of hair pulling in complex shader codebases: <br/>
![codespan](codespan.png)
* naga_oil's preprocessor supports a few more conditional directives, you can use `#else if` and `#else ifndef` as well as `#else ifdef` which was previously supported
* Functions, variables and structs are all properly scoped so a shader file doesn't need to use globally unique names to avoid conflicts
* Shader defs can be added to modules directly. For example, any shader that imports `bevy_pbr::mesh_view_types` now has `MAX_DIRECTIONAL_LIGHTS` automatically defined, there's no longer a need to remember to add it for every new pipeline that uses the module.

The future possibilities are more exciting. Using naga IR opens the door to a bunch of nice features that we hope to bring in future releases:

* Automatic bind slot allocation will let plugins extend the core view bindgroup, which means self-contained plugins for features like lighting and shadow methods, common material properties, etc become viable. This will allow us to modularise the core pipelines to make growing the codebase - while keeping support for multiple targets - more sustainable
* "Virtual" shader functions will allow user modifications to core functions (like lighting), and potentially lead to a template-style material system, where users can provide "hooks" that will be called at the right point in the pipeline
* Language interop: mix and match glsl and wgsl, so bevy's pbr pipeline features could be accessed from your glsl material shader, or utils written for glsl could be used in wgsl code. We're hopeful that this can extend to spirv (and rust-gpu) as well
* More cool stuff we haven't thought of yet. Being able to inspect and modify shaders at runtime is very powerful and makes a lot of things possible!

## Schedule-First ECS APIs

<div class="release-feature-authors">authors: @cart</div>

In **Bevy 0.10** we introduced [ECS Schedule V3](/news/bevy-0-10/#ecs-schedule-v3), which _vastly_ improved the capabilities of Bevy ECS system scheduling: scheduler API ergonomics, system chaining, the ability to run exclusive systems and apply deferred system operations at any point in a schedule, a single unified schedule, configurable System Sets, run conditions, and a better State system.

However it pretty quickly became clear that the new system still had some areas to improve:

* **Base Sets were hard to understand and error prone**: What _is_ a Base Set? When do I use them? Why do they exist? Why is my ordering implicitly invalid due to incompatible Base Set ordering? Why do some schedules have a default Base Set while others don't? [Base Sets were confusing!](https://github.com/bevyengine/bevy/pull/8079#base-set-confusion)
* **There were too many ways to schedule a System**: We've accumulated too many scheduling APIs. As of Bevy **0.10**, we had [_SIX_ different ways to add a system to the "startup" schedule](https://github.com/bevyengine/bevy/pull/8079#unify-system-apis). Thats too many ways!
* **Too much implicit configuration**: There were both default Schedules and default Base Sets. In some cases systems had default schedules or default base sets, but in other cases they didn't! [A system's schedule and configuration should be explicit and clear](https://github.com/bevyengine/bevy/pull/8079#schedule-should-be-clear).
* **Adding Systems to Schedules wasn't ergonomic**: Things like `add_system(foo.in_schedule(CoreSchedule::Startup))` were not fun to type or read. We created special-case helpers, such as `add_startup_system(foo)`, but [this required more internal code, user-defined schedules didn't benefit from the special casing, and it completely hid the `CoreSchedule::Startup` symbol!](https://github.com/bevyengine/bevy/pull/8079#ergonomic-system-adding).

### Unraveling the Complexity

If your eyes started to glaze over as you tried to wrap your head around this, or phrases like "implicitly added to the `Update` Base Set" filled you with dread ... don't worry. After [a lot of careful thought](https://github.com/bevyengine/bevy/pull/8079) we've unraveled the complexity and built something clear and simple.

In **Bevy 0.11** the "scheduling mental model" is _much_ simpler thanks to **Schedule-First ECS APIs**:

```rust
app
    .add_systems(Startup, (a, b))
    .add_systems(Update, (c, d, e))
    .add_systems(FixedUpdate, (f, g))
    .add_systems(PostUpdate, h)
    .add_systems(OnEnter(AppState::Menu), enter_menu)
    .add_systems(OnExit(AppState::Menu), exit_menu)
```

* **There is _exactly_ one way to schedule systems**
  * Call `add_systems`, state the schedule name, and specify one or more systems
* **Base Sets have been entirely removed in favor of Schedules, which have friendly / short names**
  * Ex: The `CoreSet::Update` Base Set has become `Update`
* **There is no implicit or implied configuration**
  * Default Schedules and default Base Sets don't exist
* **The syntax is easy on the eyes and ergonomic**
  * Schedules are first so they "line up" when formatted

<details>
    <summary>To compare, expand this to see what it used to be!</summary>

```rust
app
    // Startup system variant 1.
    // Has an implied default StartupSet::Startup base set
    // Has an implied CoreSchedule::Startup schedule
    .add_startup_systems((a, b))
    // Startup system variant 2.
    // Has an implied default StartupSet::Startup base set
    // Has an implied CoreSchedule::Startup schedule
    .add_systems((a, b).on_startup())
    // Startup system variant 3.
    // Has an implied default StartupSet::Startup base set
    .add_systems((a, b).in_schedule(CoreSchedule::Startup))
    // Update system variant 1.
    // `CoreSet::Update` base set and `CoreSchedule::Main` are implied
    .add_system(c)
    // Update system variant 2 (note the add_system vs add_systems difference)
    // `CoreSet::Update` base set and `CoreSchedule::Main` are implied
    .add_systems((d, e))
    // No implied default base set because CoreSchedule::FixedUpdate doesn't have one
    .add_systems((f, g).in_schedule(CoreSchedule::FixedUpdate))
    // `CoreSchedule::Main` is implied, in_base_set overrides the default CoreSet::Update set
    .add_system(h.in_base_set(CoreSet::PostUpdate))
    // This has no implied default base set
    .add_systems(enter_menu.in_schedule(OnEnter(AppState::Menu)))
    // This has no implied default base set
    .add_systems(exit_menu.in_schedule(OnExit(AppState::Menu)))
```

</details>

Note that normal "system sets" still exist! You can still use sets to organize and order your systems:

```rust
app.add_systems(Update, (
    (walk, jump).in_set(Movement),
    collide.after(Movement),
))
```

The `configure_set` API has also been adjusted for parity:

```rust
// before
app.configure_set(Foo.after(Bar).in_schedule(PostUpdate))
// after
app.configure_set(PostUpdate, Foo.after(Bar))
```

## Nested System Tuples and Chaining

<div class="release-feature-authors">authors: @cart</div>

It is now possible to infinitely nest tuples of systems in a `.add_systems` call!

```rust
app.add_systems(Update, (
    (a, (b, c, d, e), f),
    (g, h),
    i
))
```

At first glance, this might not seem very useful. But in combination with per-tuple configuration, it allows you to easily and cleanly express schedules:

```rust
app.add_systems(Update, (
    (attack, defend).in_set(Combat).before(check_health)
    check_health,
    (handle_death, respawn).after(check_health)
))
```

`.chain()` has also been adapted to support arbitrary nesting! The ordering in the example above could be rephrased like this:

```rust
app.add_systems(Update,
    (
        (attack, defend).in_set(Combat)
        check_health,
        (handle_death, respawn)
    ).chain()
)
```

This will run `attack` and `defend` first (in parallel), then `check_health`, then `handle_death` and `respawn` (in parallel).

This allows for powerful and expressive "graph-like" ordering expressions:

```rust
app.add_systems(Update,
    (
        (a, (b, c, d).chain()),
        (e, f),
    ).chain()
)
```

This will run `a` in parallel with `b->c->d`, then after those have finished running it will run `e` and `f` in parallel.

## <a name="what-s-next"></a>What's Next?

* **X**: Y

Check out the [**Bevy 0.12 Milestone**](https://github.com/bevyengine/bevy/milestone/14) for an up-to-date list of current work being considered for **Bevy 0.12**.

## Support Bevy

Sponsorships help make our work on Bevy sustainable. If you believe in Bevy's mission, consider [sponsoring us](/community/donate) ... every bit helps!

<a class="button button--pink header__cta" href="/community/donate">Donate <img class="button__icon" src="/assets/heart.svg" alt="heart icon"></a>

## Contributors

Bevy is made by a [large group of people](/community/people/). A huge thanks to the X contributors that made this release (and associated docs) possible! In random order:

* @author