# MiniMax-H3-Image ncnn Vulkan

:exclamation: :exclamation: :exclamation: More features and optimizations in progress

![CI](https://github.com/nihui/h3image-ncnn-vulkan/workflows/CI/badge.svg)
![download](https://img.shields.io/github/downloads/nihui/h3image-ncnn-vulkan/total.svg)

ncnn implementation of [MiniMax-H3-Image](https://huggingface.co/MiniMaxAI/MiniMax-H3-Image), with text-to-image and multi-reference image editing.

h3image-ncnn-vulkan uses [ncnn project](https://github.com/Tencent/ncnn) as the universal neural network inference framework.

## Features

<table>
<tr>
<td>

- **BF16** text encoder and DiT, FP32 vision and VAE
- **Vulkan** GPU acceleration
- **Windows / Linux / macOS** builds
- **CPU** inference is also supported
- **No** CUDA / PyTorch / Python dependency for inference
- **Portable** standalone executable

</td>
<td>

- Text-to-image
- Image **editing**
- Up to **10** reference images
- **Dynamic** output resolution
- **Batch** generation
- Native **6-step** PDD checkpoint
- Automatic weight streaming and VAE tiling
- Exact feature caching and optional rough modes

</td>
</tr>
</table>

minimax-h3-image text-to-image 1024x1024
|dit transformer step(s/it) less is better|RX9060XT|RTX3060|
|:-:|:-:|:-:|
|stable-diffusion.cpp (vulkan) 20261116|9.24|11.83|
|h3image-ncnn-vulkan 20261116|4.57|10.03|

minimax-h3-image two reference image editting 1536x1024
|dit transformer step(s/it) less is better|RX9060XT with 24G RAM and 15.5G VRAM|
|:-:|:-:|
|stable-diffusion.cpp (vulkan) 20261118|26.02|
|h3image-ncnn-vulkan 20261118 rough-mode 0|20.25|
|h3image-ncnn-vulkan 20261118 rough-mode 1|17.33|
|h3image-ncnn-vulkan 20261118 rough-mode 2|15.82|
|h3image-ncnn-vulkan 20261118 rough-mode 3|11.37|

## [Download](https://github.com/nihui/h3image-ncnn-vulkan/releases)

The release workflow builds Windows/Linux/macOS standalone packages and publishes a release with an automatic date tag (`YYYYMMDD`) pointing to the built commit. See [Build from Source](#build-from-source) to build locally.

Sample images are available in the repository's [images directory](https://github.com/nihui/h3image-ncnn-vulkan/tree/master/images).

The executable is portable. No CUDA, PyTorch or Python runtime environment is needed for inference :)

### prepare model files

Download the [h3image model folder(huggingface)](https://huggingface.co/nihui-szyl/h3-image-ncnn/tree/main/h3image) or [h3image model folder(modelscope)](https://www.modelscope.cn/models/nihuisyzl/h3-image-ncnn) and place it under `models/` next to the executable, or select its location with `-m`.

The prepared model uses the reduced DiT core and the six-step BF16 conditioning bank. The four- and five-step banks are optional experimental presets.

```
h3image-ncnn-vulkan
models/
    h3image/
        processor/
            vocab.txt
            merges.txt
            added_tokens.txt
        text_encoder/
            text_encoder.ncnn.param
            text_encoder.ncnn.bin
            embedding.bin
        vision/
            vision_encoder.ncnn.param
            vision_encoder.ncnn.bin
            vision_pos_embed.f32
        transformer/
            transformer-core.ncnn.param
            transformer-core.ncnn.bin
            transformer-core.ncnn.bin.sha256
            fixed-modulations-bf16.bin
            fixed-modulations-bf16-steps4.bin
            fixed-modulations-bf16-steps5.bin
        vae/
            encoder.ncnn.param
            encoder.ncnn.bin
            decoder.ncnn.param
            decoder.ncnn.bin
```

With the optional Hugging Face downloader:

```shell
hf download nihui-szyl/h3-image-ncnn --local-dir models \
  "h3image/processor/*" "h3image/text_encoder/*" \
  "h3image/vision/*" "h3image/vae/*" \
  "h3image/transformer/transformer-core.ncnn.*" \
  "h3image/transformer/fixed-modulations-bf16*.bin"
```

This prepared download is approximately 93.46 GB (87.04 GiB). Keep the checksum sidecar and verify downloaded weights against the model repository's checksums. No JSON configuration or tokenizer conversion is needed.

For online conditioning, additionally download `h3image/transformer/fixed-build/*` (about 26.11 GB). The split core and this producer cover the full DiT; the original `transformer.ncnn.*` files are not required. The producer contains `modulations.ncnn.param`, `modulations.ncnn.bin` and their checksum sidecars. Published core and producer weights preserve the original DiT tensor bytes.

## About MiniMax-H3-Image

MiniMax-H3-Image is a unified text-to-image and image editing model released by MiniMax. This implementation uses its six-step PDD checkpoint.

https://huggingface.co/MiniMaxAI/MiniMax-H3-Image

Guidance is distilled into the checkpoint. Keep `-w 1` and omit the negative prompt. The default six steps use the six trained output heads. Other step counts and rough K/V reuse are experimental.

## Usages

### requirements

- A Vulkan-capable GPU and driver, or `-g -1` for CPU inference.
- About 94 GB of disk space for the prepared model files.
- Additional disk space for streamed weights. A fast SSD is recommended; temporary use can reach tens of gigabytes.
- RAM and VRAM sufficient for the requested prompt, image sizes and references. Weight placement and VAE execution adapt automatically to current capacity: whole-graph encoding is preferred when its buffers fit, with convolution tiling as the fallback.

Estimated text-to-image planning budgets with disk weight streaming:

| Generation / output resolution | RAM budget (MiB) | VRAM budget (MiB) |
|:---:|:---:|:---:|
| 512 × 512 | 1792 | 768 |
| 1024 × 1024 | 1792 | 1024 |
| 2048 × 2048 | 1792 | 1792 |
| 4096 × 4096 | 1792 | 4864 |

These planner estimates use batch 1, a 512-token prompt, six steps, the prepared BF16 conditioning bank, rough mode 0 and a feature-cache miss with caching enabled.

The calculation includes weight-loading peaks: mapped 16-bit source pages, FP32 expansion and layout conversion, storage casts, pending upload buffers and resident weight caches. Streamed inference also reserves the mapped source pages that coexist with upload staging. These numbers are planning budgets, not minimum installed RAM/VRAM or measured hard limits. Driver and library allocations and the OS need additional capacity. Larger budgets keep more weights in memory and improve speed. Longer prompts, references and larger batches need separate estimates.

The default minimum internal generation area is 512 × 512 pixels. Each square resolution in the table generates directly at the requested size; smaller-area requests use a larger internal canvas and are resized after decoding.

CPU inference requires sufficient RAM for resident model weights and an ncnn build with BF16 Gemm support.

Use a disk-backed temporary directory when `/tmp` is a small RAM disk:

```shell
mkdir -p /path/to/fast-disk/h3image-tmp
TMPDIR=/path/to/fast-disk/h3image-tmp h3image-ncnn-vulkan -p "A red ceramic teapot." -o output.jpg
```

### Example Command

Text to image

```shell
h3image-ncnn-vulkan -p "A turquoise alpine lake in the Dolomites at sunrise." -o output.jpg
```

Image editing

```shell
h3image-ncnn-vulkan -i images/portrait.jpg -p "Change the cardigan to a cute cherry-red jacket. Preserve her face, smile, hair ribbon, milk bottle and store background." -o edited.jpg
```

Multiple reference images

```shell
h3image-ncnn-vulkan -i images/portrait.jpg -i images/person-reference.jpg \
  -p "Replace the woman in <Picture 1> with the woman from <Picture 2>. Keep the store scene, pose and milk bottle from <Picture 1>." -o person-replacement.jpg
```

Set image size, denoise steps, seed and GPU

```shell
h3image-ncnn-vulkan -s 1536,864 -l 6 -r 2026 -g 0 \
  -p "A turquoise alpine lake in the Dolomites at sunrise." -o landscape.jpg
```

Batch generation

```shell
h3image-ncnn-vulkan -b 3 -r 42 -p "A small red kite over a quiet lake." -o output.jpg
```

This saves `output-0.jpg`, `output-1.jpg` and `output-2.jpg`. A batch shares encoding and model loading; samples use successive draws from the same seeded random stream.

Set memory budgets independently, in MiB

```shell
h3image-ncnn-vulkan --ram-budget 16384 --vram-budget 4096 \
  -p "A red ceramic teapot." -o output.jpg
```

A value of `0` selects an automatic budget of `max(available × 0.95, available − 300 MiB)` for RAM and VRAM independently, reserving the smaller of 5% and 300 MiB of the current available capacity. Explicit budgets are capped by availability without this automatic reserve. The runtime plans tensors for the actual prompt, image and reference shapes, tries resident weights first, then streams weights as needed. Whole-graph VAE encoding runs when its buffers fit; smaller convolution tiles provide the low-memory fallback. Decoder tiles retain the trained context and overlap. Budgets guide allocation planning; they are not operating-system memory reservations.

Compute conditioning online instead of loading a precomputed bank

```shell
h3image-ncnn-vulkan --fixed-modulations 0 \
  -p "A red ceramic teapot." -o output.jpg
```

This requires `transformer/fixed-build/modulations.ncnn.param` and `.bin` alongside the core. Conditioning is computed before sampling for the selected step count, then the producer weights are released. The default `--fixed-modulations 1` uses a compatible bank when available and otherwise prepares conditioning online. Both modes use the split core; neither needs the original full transformer.

Optional rough K/V reuse

```shell
h3image-ncnn-vulkan -i images/amber-perfume.jpg --rough-mode 2 \
  -p "Change the glass bottle to emerald green." -r 2718 -o edited.jpg
```

Rough modes trade quality for prefix reuse and may change details or composition. `0` recomputes attention at every step; `1` reuses the text prefix, `2` also reuses reference-image tokens, and `3` freezes that complete prefix after the first step. Modes `1` and `2` refresh at steps 1, 3 and 5. K/V storage uses GPU memory when it fits and RAM otherwise.

### Feature cache

Deterministic text features and reference VAE posteriors are cached under `MODEL/.cache/h3image-exact`, with a 512 MiB disk limit. Matching later invocations can skip encoder loading and computation. Cache keys include inputs, model identity, backend and build; damaged or unreadable entries are recomputed. Reference sampling still uses the requested seed on every invocation.

```shell
h3image-ncnn-vulkan -p "A red ceramic teapot." --cache-dir /fast-disk/h3-cache --cache-size 1024 -o output.jpg
# Disable persistent caching:
h3image-ncnn-vulkan -p "A red ceramic teapot." --cache-size 0 -o output.jpg
```

The exact Vulkan path keeps static text conditioning and the FP32 PDD sample on the GPU when memory allows, reducing transfers and command submissions. Online conditioning shares each time embedding across its output heads. CPU and lower-memory paths remain available.

### Full Usages

```console
Usage: h3image-ncnn-vulkan [options]...

  -h                   show this help
  -p prompt            prompt
  -o output-path       output image path (default=out.png)
  -i input-image       reference image for editing (repeat 1 to 10 times)
  -s image-size        image resolution (default=1024,1024; editing=input size)
  -r random-seed       random seed (default=rand)
  -m model-path        model directory (default=models/h3image)
  -g gpu-id            GPU device to use (-1=cpu, default=auto)
  --ram-budget MiB     process RAM planning budget (0=auto, default)
  --vram-budget MiB    GPU memory planning budget (0=auto, default)
  --rough-mode mode    0=exact, 1=text/2, 2=full/2, 3=freeze (default=0)
  --fixed-modulations 0|1  use precomputed conditioning banks (0=online, default=1)
  --cache-dir path     exact feature cache directory (default=model/.cache/h3image-exact)
  --cache-size MiB     disk cache limit (default=512, 0=disabled)
  -b batch-size        batched generation (default=1)
  -l steps             denoise steps (default=6; other counts experimental)
  -n negative-prompt   unsupported by the guidance-distilled H3 checkpoint
  -w guidance-scale    H3 has baked-in guidance (must be 1)
```

- Input and output support JPEG, PNG and WebP. The output filename extension selects the encoding.
- Repeat `-i` for up to ten references and refer to them as `<Picture 1>`, `<Picture 2>`, and so on. Editing keeps the first reference's output dimensions unless `-s` is set.
- Explicit output dimensions are 32–4096 pixels per side. H3 generates on a 32-pixel-aligned canvas with at least 512×512 pixels of area, then resizes to the requested dimensions.
- `-r` fixes the seed. Reproducibility also depends on the model, backend and runtime. A matching seed in another implementation need not produce the same image.
- `-l` accepts 1–256; use `6` for the released schedule. The prepared 4/5-step presets and custom schedules are experimental. Other counts require a matching bank or the optional online-conditioning producer.
- This checkpoint rejects a nonempty negative prompt and guidance scales other than `1`.

## Build from Source

1. Clone this project with all submodules

```shell
git clone https://github.com/nihui/h3image-ncnn-vulkan.git
cd h3image-ncnn-vulkan
git submodule update --init --recursive --depth 1
```

2. Build with CMake

```shell
mkdir build
cd build
cmake ../src
cmake --build . --config Release -j 4
```

On Linux, install a C++ compiler, CMake and the Vulkan development headers and loader. On Windows, use Visual Studio and the Vulkan SDK. On macOS, use Xcode command line tools and a Vulkan SDK with MoltenVK. The workflow files contain the platform build settings.

ncnn and the image codecs are built from the pinned submodules under `src/`. Source archives include these dependencies. RTTI and C++ exceptions are disabled by default (`NCNN_DISABLE_RTTI=ON`, `NCNN_DISABLE_EXCEPTION=ON`); loading and inference failures return an error while releasing their resources.

For a CPU-only executable, configure with `-DNCNN_VULKAN=OFF`. System libraries can be selected with `-DUSE_SYSTEM_NCNN=ON`, `-DUSE_SYSTEM_JPEG=ON`, `-DUSE_SYSTEM_PNG=ON`, `-DUSE_SYSTEM_ZLIB=ON` and `-DUSE_SYSTEM_WEBP=ON`. The ncnn version must include the paged-weight allocator from PR #7167.

### Windows command-line encoding (UTF-8)

The executable embeds a UTF-8 application manifest and sets the console output code page to UTF-8. This preserves Chinese and other non-ASCII prompts on Windows 10 version 1903 or newer.

## Sample Images

The examples below use this project's Vulkan pipeline and the six-step BF16 bank. The original JPEG files are available in the repository's [images directory](https://github.com/nihui/h3image-ncnn-vulkan/tree/master/images). Expand a command to see the complete prompt, output dimensions and seed. Rendering can vary across GPU backends.

These original scene descriptions follow the composition, material, lighting and camera detail used in the [official MiniMax-H3-Image showcase](https://github.com/MiniMax-AI/MiniMax-H3-Image/tree/main/showcase). The gallery covers photography, watercolor, science fiction, graphic design, surreal photography, fantasy environments and gongbi painting. Each image was generated at the dimensions in its command, approximately 4–5.3 megapixels. Detailed prompts and larger canvases are used here to describe and resolve scene structure; the files are original outputs from this implementation.

### Text to image

<table width="100%">
<tr>
<td width="33%" valign="top">

Convenience-store portrait · 1:1

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "A candid editorial photograph of a lively young East Asian woman holding a bottle of milk inside a quiet convenience store late at night. Square composition, framed from the waist upward at eye level, with both hands and the complete bottle visible above the lower edge. The atmosphere is cheerful and spontaneous, like a happy moment shared with a friend during a late walk home.

[Subject and gesture]
The woman stands slightly to the right of center, her body turned gently toward the refrigerator on the left while her face turns back toward the camera. She has bright expressive eyes and a natural, delighted smile, with a slight playful tilt of the head and relaxed shoulders. Her dark shoulder-length hair is loosely gathered half-up with one small ivory ribbon, and a few soft wisps frame her forehead. Keep fine natural skin texture and a fresh, lightly made-up appearance. She wears a blush-pink knitted cardigan over an ivory cotton blouse with a small rounded collar. Neat ribbed cuffs, small cream buttons, and one tiny embroidered cherry near the cardigan's left chest give the outfit a cute everyday character. Both hands hold one small opaque white milk bottle at waist level: one supports its base and the other rests around its middle. The bottle has a simple green screw cap, a cylindrical body, and one understated cream-and-green label with a small leaf emblem. The hands, cap, and bottle remain clearly separated.

[Store and spatial structure]
An upright glass-front refrigerator occupies the left third of the picture and recedes toward the rear wall. Show three straight horizontal shelves, with a modest number of upright milk bottles and juice cartons arranged in small groups. Each container has a complete silhouette, a distinct base and cap, and a narrow space separating it from its neighbors. The metal shelf edges follow one consistent perspective and remain visible between the rows. Reflections in the glass are faint and belong to this room. A warm wooden counter and the dark street beyond the front window form a quiet, uncluttered background on the right.

[Light, camera, and finish]
Warm ceiling light gives the face, pink cardigan, and ivory collar a gentle amber tone; the refrigerator contributes a soft cooler fill from the left. Preserve detail in the bright white bottles without turning the shelf into a white glow. Use a 50 mm photographic perspective at about f/2.8: the face and hands are sharp, the nearby shelf shapes remain understandable, and the distant shop softens naturally. Soft pastel pink, cream, and green form a fresh, inviting palette. Gentle highlight roll-off, subtle fine grain, credible glass, knit, and cotton textures. The picture fills the frame, with no added typography, watermark, beauty-filter finish, or collage elements." -s 2048,2048 -l 6 -r 42 -o portrait.jpg
```

</details>

![portrait](images/portrait.jpg)

</td>
<td width="33%" valign="top">

Macro wildlife · 1:1

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "A patient natural-history macro photograph of one small emerald tree frog resting on the curled tip of a fern in a humid tropical forest. Square frame, intimate scale, and a calm composition built around the frog's eye and the arc of the plant.

[Animal anatomy and pose]
The frog sits just left of center, facing toward the upper right in a clear three-quarter side view. Its rounded body is supported by folded hind legs, while both front feet rest securely on the upper surface of the frond. The visible toes terminate in small circular pads and wrap naturally around the narrow leaf stem. Its near eye is amber-gold with a dark horizontal pupil and one small reflection of the pale sky opening above. Fine pores and subtle changes from bright green on the back to pale yellow-green along the flank keep the skin tactile and alive. The mouth is closed and the expression remains that of a real animal.

[Plant and environment]
A single fern grows diagonally from the lower left toward the center, curling upward beneath the frog. Individual leaflets attach regularly to the central stem without merging into the animal. Several dew droplets sit at the edges; the largest droplet hangs below the curl and refracts a tiny inverted patch of foliage. Dark moss covers a small area of the lower-left corner. Behind the subject, broad rainforest leaves form a quiet arrangement of soft dark-green shapes, with one pale opening far in the upper right. Nothing competes with the eye or breaks the frog's outline.

[Optics and color]
Use diffused early-morning light from the upper right, softened by the forest canopy, with gentle shadows under the belly and feet. A 100 mm macro lens at about f/5.6 keeps the near eye, shoulder, and supporting toes in focus; the rear body gradually softens, and the background falls well outside the depth of field. Wet skin has restrained highlights rather than a plastic gloss. Retain a realistic emerald-and-olive palette, fine organic detail, and smooth photographic tonal transitions. No extra animals, duplicate limbs, oversized flowers, luminous particles, captions, logos, or watermark." -s 2048,2048 -l 6 -r 8864 -o emerald-frog.jpg
```

</details>

![emerald-frog](images/emerald-frog.jpg)

</td>
<td width="33%" valign="top">

Watercolor illustration · 4:3

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "A complete watercolor-and-ink illustration for a beautifully printed children's storybook, showing a red fox quietly reading inside the library of an ancient hollow tree. Landscape 4:3 composition. It should feel hand-painted on warm cream cotton paper, with visible pigment, delicate drawing, and an inviting narrative moment.

[Main figure]
Place the fox in the lower center-right, seated in a small moss-green armchair with its body in profile and its head bent toward an open book resting on its knees. Its russet ears point upward, its cream muzzle is softly lit, and its long white-tipped tail curls around the chair's base toward the lower center. Two forepaws support the book at its lower corners. The eyes follow the page, and the animal's proportions remain elegant and fox-like rather than becoming a human in a costume. The pages show faint irregular ink marks, not a readable title.

[Room and arrangement]
The tree trunk curves around the room in a broad oval. Three low wooden bookshelves follow the left wall, filled with small books of different thicknesses in dusty blue, ocher, muted brick, and cream. A simple brass reading lamp stands on a round side table near the fox's shoulder. Behind the chair, a round window reveals a pale misty forest with a few distant trunks. In the near foreground, one open notebook and a tiny ceramic cup lie on a braided rug. A narrow stair carved into the trunk curves upward at the far left and disappears into shadow. Keep furniture supported by the floor and leave clear paths between the objects.

[Painting technique and mood]
Golden lamplight meets cool blue-green daylight from the window. Build the atmosphere with transparent washes, soft blooms of pigment in the bark and forest, and controlled darker strokes around the fox's face, paws, and book. Use thin warm-brown ink lines that vary in pressure, allowing some contours to disappear into the wash. A few paper-white gaps catch light on the book and tail. The room is richly observed but not cluttered, with recognizable silhouettes before small decorative details. No glossy 3D rendering, photographic fur, heavy black outlines, speech bubbles, cover typography, signatures, or watermark." -s 2560,1920 -l 6 -r 31415 -o fox-library.jpg
```

</details>

![fox-library](images/fox-library.jpg)

</td>
</tr>
<tr>
<td width="33%" valign="top">

Alpine landscape · 16:9

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "A carefully observed landscape photograph of a turquoise alpine lake below the limestone peaks of the Dolomites just after sunrise. Wide 16:9 composition, with a sense of stillness and cool early-morning air rather than an exaggerated travel advertisement.

[Foreground and visual path]
Photograph from a low grassy bank on the southern shore. In the lower left, several rough pale stones emerge through shallow, transparent water; their submerged halves are darker and slightly distorted by the surface. A small cluster of yellow alpine flowers and three slender grass stems catch the first sunlight beside them. The shoreline curves from the lower left toward a small dark timber boathouse in the middle distance on the right. Keep the foreground sparse enough that the lake remains the main subject.

[Lake, forest, and mountains]
The lake fills the lower half of the image. Its near water is clear green-blue, becoming deeper teal toward the opposite shore. An almost unbroken reflection of the mountains is interrupted by a few gentle horizontal ripples near the bank. A band of dark spruce forest follows the far shoreline, with individual treetops readable against the foothills. Above it, three main limestone summits form an asymmetrical skyline: the tallest peak sits left of center, with smaller ridges stepping down to the right. Thin mist lies just above the water at the far shore and does not obscure the principal reflection. The boathouse rests firmly on its shore, with one simple doorway and a short wooden landing.

[Light and photographic treatment]
Low sunlight from the upper right touches only the upper rock faces with pale rose-gold light; the forest and near bank retain cool blue-green shadows. The sky is a light, desaturated blue with a few narrow cirrus clouds. Use a 28 mm lens, a level horizon, and deep but natural depth of field. Maintain believable relative scale, gentle atmospheric separation, detailed limestone strata, and restrained water saturation. Highlights stay soft, shadows retain texture, and the result feels like a real exposure made with patient attention. No people, boats, text, logos, excessive HDR, or invented glowing elements." -s 3072,1728 -l 6 -r 2026 -o alpine-lake.jpg
```

</details>

![alpine-lake](images/alpine-lake.jpg)

</td>
<td width="33%" valign="top">

Architecture · 3:2

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "A realistic architectural photograph of a narrow Kyoto street shortly after a light evening rain. Landscape 3:2 format, shot at pedestrian eye level with a normal wide lens. The picture captures the quiet interval when warm shop lights have come on but a little blue daylight remains in the sky.

[Composition and buildings]
The stone-paved lane enters from the lower center and bends gently to the left beyond the middle of the frame. Two wooden machiya shopfronts line the right side; the nearest facade occupies the right third and has dark vertical timber slats, a shallow tiled eave, a recessed entrance, and a low bench. Across the lane, a plaster wall and a smaller wooden doorway create a quieter left edge. Keep the buildings upright, with coherent roof lines and converging paving joints establishing one continuous perspective. A small red paper umbrella leans, closed, beside the nearest doorway rather than floating in the street.

[Objects and atmosphere]
Place one warm round paper lantern under the near eave and two smaller lanterns farther down the lane. Their paper surfaces show delicate ribs and softly glowing centers, with no large written characters. A blue ceramic pot with a modest green plant sits beside the bench. Rain has darkened the stones; shallow puddles collect in their uneven joints and reflect the lanterns as broken vertical amber streaks. The street is empty, but a faint curl of steam from a tea-shop doorway suggests recent human activity. The distant turn dissolves into a thin blue-gray haze.

[Light and material]
The scene balances cool blue-hour ambient light with localized amber pools from the shop interiors. Preserve the texture of wet wood, chipped plaster, fired roof tiles, and worn stone without making every surface equally shiny. Use a 35 mm lens at approximately f/5.6, with clear foreground architecture and gradually softer distance. The lanterns remain within a plausible exposure range; their reflections follow the wet ground. Muted indigo, dark brown, soft ivory, and a single red accent form the palette. Natural perspective, moderate contrast, subtle grain, and no crowds, fantasy buildings, decorative text, watermark, or artificial neon wash." -s 2688,1792 -l 6 -r 3421 -o kyoto-street.jpg
```

</details>

![kyoto-street](images/kyoto-street.jpg)

</td>
<td width="33%" valign="top">

Science fiction · 16:9

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "A finished environment concept image for an original science-fiction film, showing a large orbital research station above the blue curve of Earth at sunrise. Wide 16:9 composition, with physically credible scale, an immediately readable engineered silhouette, and restrained cinematic lighting.

[Station structure]
The station occupies the upper-left and central portions of the frame. Its principal form is one broad habitation ring seen obliquely as an ellipse, attached to a straight axial hub by four evenly spaced structural spokes. The near portion of the ring is larger and more detailed than the far portion. Pressurized modules, service hatches, and shallow radiator panels follow the ring's continuous curvature rather than breaking it into unrelated fragments. Two long rectangular solar-panel wings extend from the central hub toward the upper right and lower left, with fine, regular blue-black cell grids and clearly attached metal support frames.

[Depth and narrative scale]
Beneath the near side of the hub, a small cargo shuttle approaches one open docking bay. Its tiny white hull and two warm navigation lights provide a clear scale comparison without hiding the station. A second, much smaller service vehicle rests beside the far ring. Earth fills the lower-right half, with a luminous blue atmospheric rim, broad white cloud systems, and glimpses of muted land through the cloud gaps. Keep the planet's curvature smooth and its cloud patterns coherent. The remaining upper-right sky is almost black, with only a sparse scattering of faint stars.

[Light and surface language]
The rising sun sits outside the upper-left frame, illuminating the station's silver-white outward-facing surfaces with a thin warm edge. Unlit sides retain cool gray detail from reflected Earth light. Use panel seams and occasional amber service markings sparingly; prioritize large functional forms before small mechanical detail. Solar panels are matte and dark, metal has broad controlled reflections, and the docking bay glows softly without neon haze. A moderately long-lens view gives the scene weight and avoids toy-like wide-angle distortion. One complete cinematic artwork, no existing franchise designs, labels, logos, interface elements, giant lens flare, or watermark." -s 3072,1728 -l 6 -r 1618 -o orbital-station.jpg
```

</details>

![orbital-station](images/orbital-station.jpg)

</td>
</tr>
<tr>
<td width="33%" valign="top">

Product photography · 3:4

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "A refined studio still-life photograph for an independent fragrance campaign, portrait 3:4 format. One translucent amber perfume bottle stands on a pale travertine plinth, balanced by a white orchid and the quiet curve of a stone arch. The image relies on precise material behavior and a spacious composition.

[Bottle and nearby objects]
Position the bottle slightly left of center in the lower-middle part of the frame. Its body is a clean rectangular volume with softly rounded vertical corners, a thick transparent glass base, and a short neck supporting one rounded gold-metal cap. The amber liquid fills approximately three quarters of the bottle, with a level horizontal surface visible through the glass. The bottle is unbranded and has no label or lettering. A small oval highlight along the left edge and a darker refracted edge on the right reveal the wall thickness. The cap fits the neck directly and has a restrained satin-gold reflection rather than a mirror finish.

[Set and composition]
A low cylindrical travertine plinth occupies the lower third, with fine horizontal pores and a slightly irregular, naturally cut rim. At the right rear, one slender orchid stem enters from below the frame and bends inward with three white blossoms. Its petals overlap delicately, with pale yellow centers and softly translucent edges. A broad ivory stone arch rises behind the still life; its opening creates a quiet pool of slightly deeper beige. Leave the upper quarter mostly open and keep the main bottle entirely within the frame. The orchid does not cross the bottle's cap or obscure its silhouette.

[Lighting and finish]
A large warm window light from the upper left produces a broad gradient across the bottle and a long, soft-edged shadow toward the lower right. A faint amber caustic falls on the plinth immediately beside the base. Gentle reflected fill keeps the glass interior readable. Use an 85 mm lens and sufficient depth of field to resolve the bottle, cap, and near petals; the distant arch softens slightly. The palette stays within warm ivory, honey amber, pale gold, and a small touch of leaf green. Fine surface detail, controlled highlights, convincing glass refraction, no extra bottles, product text, floating props, watermark, or computer-rendered sheen." -s 1920,2560 -l 6 -r 2718 -o amber-perfume.jpg
```

</details>

![amber-perfume](images/amber-perfume.jpg)

</td>
<td width="33%" valign="top">

Chinese poster · 9:16

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "Design a complete contemporary Chinese ink-art exhibition poster, portrait 9:16, on warm ivory handmade paper. It is a flat graphic artwork intended for print, with a calm hierarchy, generous negative space, and only one short Chinese title.

[Layout and exact lettering]
The upper fifth remains mostly open paper. In the upper-middle area, place the exact four-character title 山海之间 as one large vertical column, reading from top to bottom: 山, 海, 之, 间. Use elegant black brush lettering with clear, well-formed character structures and slightly varying stroke weight. Each character is distinct, evenly spaced, and visually legible; no additional letters, dates, subtitles, or decorative pseudo-writing appear anywhere. The title column sits just to the right of the central axis, leaving open air around every character.

[Landscape and visual balance]
The lower half contains three layers of ink-wash mountains. A dark, textured near ridge rises from the lower left; a softer blue-gray ridge sits behind it; the farthest pale mountain dissolves into paper-white mist near the middle of the poster. A winding unpainted river creates a narrow bright path from the lower center into the distance. One small cinnabar-red sun rests above the far ridge on the left, balancing the title. Add one tiny rectangular red seal near the bottom right as a simple abstract carved mark, without readable seal-script text. The red sun and seal are the only strong color accents.

[Surface and print treatment]
Brushwork should reveal dry-brush breaks on the closest rocks, transparent blooms in the distant washes, and a few feathered edges where ink meets absorbent fibers. The title has dense ink with slight natural variation, not a digital outline effect. Preserve a broad open interval of bare paper between the lettering and mountains, with asymmetric but stable visual weight. The paper grain is delicate and evenly lit, like a high-resolution scan of a physical print. Refined charcoal black, pale gray-blue, ivory, and restrained red. No frame, mockup wall, heavy shadows, gradients behind the text, logos, signature, watermark, or extra typography." -s 1728,3072 -l 6 -r 20261116 -o mountain-poster.jpg
```

</details>

![mountain-poster](images/mountain-poster.jpg)

</td>
<td width="33%" valign="top">

Gongbi painting · 2:3

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "A Chinese gongbi painting in mineral colors on lightly aged silk, portrait 2:3. Depict one kingfisher perched on a lotus stem above a quiet pond. The work has the delicacy of a physical hanging-scroll painting, with precise linework, layered translucent color, and thoughtful open space.

[Bird and focal point]
The kingfisher sits just above the center on a slender bent lotus stalk entering from the lower left. Its small body faces right in profile, with a long dark beak angled gently downward toward the pond. Turquoise and deep-blue feathers cover the head and back; the breast is warm cinnamon orange, with a small clean white patch at the neck. Two tiny feet grasp the stalk securely. Paint fine feather divisions with controlled dark outlines and thin overlapping color layers, keeping the eye a single glossy black point with one minute highlight. The bird is anatomically compact and alert, not enlarged into a decorative fantasy creature.

[Botanical composition]
A broad lotus leaf rises on the lower left, partly folded so its pale underside is visible. A second leaf, darker and more weathered, curves across the lower-right corner without touching the bird. One pale pink lotus blossom appears below and to the right of the perch, with a yellow seed center and clearly separated petals. A distant seedpod and two faint reed stems sit near the lower edge. Leave the upper-left and central-right areas mostly empty silk. A few pale ink ripples indicate water at the bottom; there is no fully painted photographic pond or horizon.

[Line, pigment, and surface]
Use hair-fine warm-gray outlines for stems and petal edges, stronger ink accents around the bird's beak and feet, and many thin glazes of azurite blue, malachite green, pale pink, and ocher. Leaf veins follow the leaf surfaces and remain softer than the bird's feather details. Preserve the weave of the silk and slight variations where mineral pigment collects, with no uniform digital distress filter. The image is flat, decorative, and materially hand-painted, with quiet asymmetry and a clear contrast between dense detail and breathing space. No calligraphy, inscriptions, red seals, border, photographic blur, 3D lighting, signature, or watermark." -s 1792,2688 -l 6 -r 7003 -o kingfisher-gongbi.jpg
```

</details>

![kingfisher-gongbi](images/kingfisher-gongbi.jpg)

</td>
</tr>
<tr>
<td width="33%" valign="top">

Food photography · 4:3

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "An editorial food photograph of a lemon tart on a pale blue handmade plate, photographed at a sunlit kitchen table shortly before breakfast. Landscape 4:3 framing, with the natural, unhurried feel of a small food magazine rather than an overfilled advertising set.

[Tart and cut slice]
Place the round tart slightly right of center. Its golden pastry shell has a gently fluted rim, with a smooth, softly reflective lemon-yellow filling set just below the crust. One wedge has been removed from the near-left side and placed on the same plate a short distance in front of the tart. The two cut faces reveal a consistent thickness of filling over a thin crisp pastry base. Three blueberries and one small white edible blossom sit near the top edge of the tart, with a few tiny lemon-zest curls; the rest of the yellow surface remains visible. A few believable pastry crumbs collect beside the cut slice.

[Table and supporting objects]
The plate rests on a honey-colored oak table whose grain runs diagonally toward the upper right. At the left, a folded cream linen napkin supports a small silver dessert fork, its four tines pointing away from the camera. A cut lemon half sits near the back left, with a second small group of blueberries between it and the plate. The far edge of a plain clear water glass is visible in the upper right, soft and unobtrusive. Keep the props separated, with comfortable empty table areas and a stable, physically plausible arrangement.

[Light and optics]
Broad morning light enters from a window at the upper left, giving the filling a gentle sheen and casting soft shadows to the right of the plate and fork. The pastry looks crisp, the cut filling dense and silky, the blueberries matte with natural bloom, and the ceramic glaze subtly uneven. Shoot at a three-quarter overhead angle with a 70 mm lens at about f/5.6. The tart, removed slice, and near plate edge are sharp; background objects soften gradually. Use fresh but restrained yellow and blue, true white balance, fine texture, and moderate contrast. No melted geometry, excessive garnish, implausibly long forks, readable packaging, text, logos, or watermark." -s 2560,1920 -l 6 -r 10086 -o lemon-tart.jpg
```

</details>

![lemon-tart](images/lemon-tart.jpg)

</td>
<td width="33%" valign="top">

Surreal photography · 16:9

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "A surreal fashion-editorial photograph made to look like a carefully constructed physical set: an enormous open book rests on a quiet beach, its pages forming a shallow pavilion around a single reading chair. Wide 16:9 frame, poised between an ordinary seaside photograph and an impossible literary daydream.

[Main set and scale]
Place the book left of center, about three meters across, with its spine lying firmly in pale sand and both covers open at a broad angle. Thick ivory pages rise gently on either side of the central fold, curving like two low architectural wings. Their edges reveal many fine stacked paper layers, with a few softly irregular corners. The visible page surfaces remain almost blank, showing only faint indistinct gray printing. A dark cherry-red wooden chair stands in front of the fold, its four legs pressing into the sand. A woman in a simple cream linen dress sits in the chair, holding one normal-sized closed blue book on her lap and looking quietly toward the sea on the right. Her proportions and seated gesture are natural.

[Environment and spatial cues]
A short trail of footprints leads from the lower-right foreground toward the chair and ends there. Small ripples of damp sand reflect the sky near the waterline. The sea forms a calm horizontal band beyond the set, with one thin wave curling onto the shore. A pale full moon hangs low above the distant horizon on the right and is the only moon in the scene. Keep the beach otherwise open, allowing the giant book, figure, and red chair to read clearly at a glance.

[Photographic language]
Shoot at the last blue hour before sunrise, with soft cool ambient light and a gentle warm practical glow from a small brass lamp beside the chair. The lamp lights a small patch of the figure's sleeve and nearby page without turning the set into a glowing sculpture. Give paper, linen, painted wood, and wet sand distinct, tactile materials. Use a 50 mm lens, level horizon, and moderate depth of field. Muted blue, ivory, sand beige, and one restrained red accent, subtle film grain, soft highlights, and practical-prop realism. No readable title, surreal floating debris, additional people, brand marks, artificial frame, or watermark." -s 3072,1728 -l 6 -r 7001 -o moon-reading.jpg
```

</details>

![moon-reading](images/moon-reading.jpg)

</td>
<td width="33%" valign="top">

Fantasy environment · 16:9

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -p "A finished fantasy-game environment concept showing an abandoned stone observatory reclaimed by an ancient temperate forest. Wide 16:9 composition, rendered with credible architecture and materials, clear foreground-to-background depth, and a quiet sense of a place once used to study the stars.

[Architecture and terrain]
The observatory stands on a low rocky terrace slightly right of center. A circular two-story stone building supports a partly open weathered copper dome, its green patina broken by reddish seams and a few missing plates. Inside the dome opening, a large brass telescope points upward at a moderate angle, mounted on a recognizable pivot and stable pedestal. A short straight stair rises from the lower center to the building's open doorway; worn steps follow the slope of the rock and meet the threshold at the correct height. A low retaining wall curves around the terrace, with several displaced blocks resting on the ground below.

[Forest and narrative details]
Huge cedar trunks frame the left edge and the far right, their roots entering cracks in the terrace without passing through intact masonry. Moss, ferns, and small pale wildflowers occupy damp joints and shaded stones. In the lower-left foreground, a broken stone star chart lies tilted beside a shallow puddle, its carved radial pattern visible without readable words. One cloaked traveler, small relative to the architecture, stands on the upper stair looking toward the telescope. Keep this figure subordinate to the environment. Beyond the observatory, layered tree trunks and a distant ravine fade into fine morning mist.

[Light and concept-art finish]
Early sunlight enters from the upper left through one opening in the canopy. It lights the dome rim, a few steps, and selected fern edges, while the forest interior remains cool and subdued. Stone is rough and damp, copper softly oxidized, brass warm and slightly tarnished, and foliage varied without repetitive identical leaves. Use an eye-level 35 mm perspective with stable verticals and a strong, readable architectural silhouette. Fine details support the large forms rather than overwhelming them. Restrained forest green, cool stone gray, copper, and small warm highlights. Original setting, no existing game symbols, glowing runes, title, UI, signature, or watermark." -s 3072,1728 -l 6 -r 7002 -o forest-observatory.jpg
```

</details>

![forest-observatory](images/forest-observatory.jpg)

</td>
</tr>
</table>

### Image editing

<table width="100%">
<tr>
<td colspan="3">

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -i images/portrait.jpg -p "Edit <Picture 1> into the same cheerful convenience-store portrait, changing only the woman's blush-pink knitted cardigan into a cute cherry-red cotton jacket. This is a localized wardrobe change within the existing photograph, with the subject's identity and the complete scene preserved.

[Requested change]
The new jacket is a soft cherry red with a small rounded collar, subtle seams, small cream buttons, and natural cotton texture. It follows exactly the original shoulders, arms, torso outline, sleeve length, and folds imposed by the pose. Keep the ivory round-collar blouse visible at the neckline. The jacket receives the same warm overhead light and cool reflected refrigerator light as the original garment, with plausible red highlights and shadows. Its overall character stays youthful and cute without adding accessories, decorative lettering, oversized buttons, or a different pose.

[Preserve the person and interaction]
Retain the woman's recognizable facial proportions, bright cheerful smile, gaze direction, half-up hairstyle, small ivory hair ribbon, loose hair strands, skin texture, and body proportions. Keep the exact arrangement of both hands around the milk bottle, with the same visible fingers and overlap. The bottle's white body, green cap, simple label, size, tilt, and position stay unchanged. Preserve the natural occlusion of the sleeves behind the hands and bottle. The output remains a waist-up square photograph taken from the same camera position.

[Preserve the environment and finish]
Keep the refrigerator's glass door, three shelf edges, product groups, reflections, store counter, rear window, floor, and all background objects in their original locations. Preserve the original focus fall-off, exposure, warm-to-cool light balance, subtle grain, and photographic detail. No new merchandise, enlarged hands, replacement face, rearranged shelves, extra bottle, added text, border, collage seam, or watermark. The result should look like the same exposure with the wardrobe color and garment changed naturally." -s 2048,2048 -l 6 -r 42 -o portrait-edit.jpg
```

</details>

</td>
</tr>
<tr>
<td width="44%" valign="middle"><img src="images/portrait.jpg" width="100%" alt="Input reference"></td>
<td width="8%" align="center" valign="middle">➡️</td>
<td width="44%" valign="middle"><img src="images/portrait-edit.jpg" width="100%" alt="Edited result"></td>
</tr>
</table>

<table width="100%">
<tr>
<td colspan="3">

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -i images/amber-perfume.jpg -p "Edit the product photograph in <Picture 1> by changing the perfume bottle's amber glass and amber-toned liquid into deep emerald-green glass with a gently green-tinted liquid. Preserve the original composition and every structural feature of the still life.

[Glass and optical change]
The bottle retains the exact same rectangular body, softly rounded corners, wall thickness, thick transparent base, short neck, and level fill line. The green remains translucent: light can pass through the bottle, and the original interior shape is still visible. Keep the brightness and positions of the broad studio highlights, while their transmitted color and the small caustic beside the base shift naturally from amber to emerald. Darker glass edges become deep forest green rather than opaque black. The liquid occupies the same volume and has the same meniscus.

[Objects that remain unchanged]
Keep the satin-gold cap, its outline, fit, texture, and reflected lighting exactly as in the input. Preserve the travertine plinth's pores, rim, size, and position; the orchid's three blossoms, petals, stem, and leaves; and the broad ivory stone arch behind the arrangement. The bottle remains unbranded and has no added label. All object boundaries, contact points, occlusions, and cast shadows follow the original photograph.

[Photographic continuity]
Retain the warm ivory background, the large soft window light from the upper left, the same camera angle and depth of field, and the quiet empty space above the still life. The edited glass should have the same realistic material quality as the original, with the green becoming the sole new color emphasis. No extra bottle, shifted orchid, altered cap, floating object, new graphic design, written product name, watermark, or changes to the framing. Output one complete portrait-format product photograph." -s 1920,2560 -l 6 -r 2718 -o perfume-edit.jpg
```

</details>

</td>
</tr>
<tr>
<td width="44%" valign="middle"><img src="images/amber-perfume.jpg" width="100%" alt="Input reference"></td>
<td width="8%" align="center" valign="middle">➡️</td>
<td width="44%" valign="middle"><img src="images/perfume-edit.jpg" width="100%" alt="Edited result"></td>
</tr>
</table>

### Multiple reference images

Replace the person while keeping the original scene. Picture 1 supplies the store, framing, pose and milk bottle; Picture 2 supplies the new face, short bob, glasses and yellow outfit.

<table width="100%">
<tr>
<td colspan="4">

<details>
<summary>expand for full prompt and command</summary>

```shell
h3image-ncnn-vulkan -i images/portrait.jpg -i images/person-reference.jpg -p "Edit <Picture 1> by replacing the woman with the different woman shown in <Picture 2>. Keep the convenience-store scene from <Picture 1> in place. The result is the same photograph with a new person standing in the original person's position, holding the original milk bottle in the original pose.

[Use each reference for its assigned role]
<Picture 1> supplies the complete scene, camera position, square framing, perspective, body placement, head angle, hand gesture, milk bottle, lighting, and background. <Picture 2> supplies only the new woman's identity, facial features, chin-length chestnut bob, thin round eyeglasses, butter-yellow cardigan, cream rounded-collar blouse, and tiny white daisy embroidery. Transfer the recognizable face and hairstyle from <Picture 2>, including her bright eyes and cheerful natural smile. The new person should clearly resemble <Picture 2> rather than retaining the original woman's face, longer tied-back hair, ivory hair ribbon, or pink cherry cardigan.

[Fit the new person into the original pose]
Place her head and body exactly where the original woman's head and body are in <Picture 1>, at the same scale and camera distance. Follow the first picture's torso angle, slight head tilt, gaze toward the camera, shoulder line, bent elbows, and hand positions around the bottle. Adapt the yellow cardigan sleeves and folds to this pose. Keep one white milk bottle with its green screw cap, label, size, tilt, and position unchanged, with both hands naturally supporting and wrapping around it. The new glasses follow the face perspective, and the short bob rests naturally above the shoulders. Relight the new person with the original warm overhead store light and cooler refrigerator fill, including credible contact shadows beneath the chin, glasses, sleeves, and fingers.

[Preserve the original scene]
Keep the left refrigerator frame and glass door, straight shelf rails, every visible group of bottles and cartons, green caps, reflections, dark rear window, warm wooden counter, background blur, and all empty spaces in their original locations. Preserve the original crop, focus fall-off, exposure, color balance, subtle grain, and photographic texture. The new woman's silhouette may naturally reveal or cover a small area around the changed hair and clothing, but the surrounding store remains the same continuous place. Do not import the studio backdrop or empty-handed pose from <Picture 2>. Do not move the woman into a new room, rearrange merchandise, change the bottle, add another person, add text, or create a collage. One coherent photograph of the new woman in the unchanged convenience-store scene." -s 2048,2048 -l 6 -r 42 -o person-replacement.jpg
```

</details>

</td>
</tr>
<tr>
<td width="31%" valign="middle">Picture 1 · original scene<br><img src="images/portrait.jpg" width="100%" alt="Picture 1: original scene and pose"></td>
<td width="31%" valign="middle">Picture 2 · new person<br><img src="images/person-reference.jpg" width="100%" alt="Picture 2: new person identity"></td>
<td width="7%" align="center" valign="middle">➡️</td>
<td width="31%" valign="middle">Result · same scene<br><img src="images/person-replacement.jpg" width="100%" alt="New person in the same store scene"></td>
</tr>
</table>

<details>
<summary>expand to generate Picture 2, the new person reference</summary>

```shell
h3image-ncnn-vulkan -p "A fresh, natural studio portrait of a lively young East Asian woman with a short dark chestnut bob and thin round glasses. Square photographic frame, shown from the waist upward against a plain warm ivory backdrop. This is one clear identity reference photograph, with a fully visible head, recognizable facial features, and a simple cute everyday outfit.

[Person and expression]
Her face is softly oval with a slightly pointed chin, straight gently lifted eyebrows, bright dark eyes, a small straight nose, and a broad spontaneous smile that reveals her upper teeth. She looks directly into the camera with a relaxed, cheerful expression. Her chin-length chestnut-brown bob has a neat curved outline, a light side part, and one side tucked behind her ear. The hair ends stay above the shoulders. Slim dark bronze round eyeglass frames sit naturally on the bridge of her nose, with transparent lenses and faint reflections that leave both eyes clearly visible. Preserve fine natural skin texture, a light touch of peach blush, and an approachable appearance.

[Clothing and pose]
She wears a soft butter-yellow cardigan with small ivory buttons over a cream blouse with a modest rounded collar. The knit has fine ribbing at the cuffs and hem, and one tiny white daisy embroidery near the left chest. The outfit feels playful, comfortable, and believable. Her shoulders are relaxed, her body is turned slightly toward the left, and her face turns back toward the camera. Her hands rest naturally together at waist level, empty and fully separated from the clothing. Keep her head, hair, glasses, upper torso, sleeves, and hands inside the image with comfortable space around the silhouette.

[Light and photographic finish]
Use a large soft window light from the front left and a gentle reflected fill on the right. An eye-level 70 mm lens at about f/4 gives natural facial proportions, sharp eyes and glasses, and enough depth of field for the complete hairstyle and outfit. Warm ivory, butter yellow, chestnut brown, and subtle peach form a clean palette. The backdrop is evenly lit with only a very soft tonal gradient and no furniture or props. Fine photographic grain, restrained highlights, realistic fabric and hair detail. One woman only, no bottle, shop, shelves, additional people, lettering, watermark, collage, or artificial beauty-filter skin." -s 2048,2048 -l 6 -r 1742 -o person-reference.jpg
```

</details>

## Original MiniMax-H3-Image Project

- https://github.com/MiniMax-AI/MiniMax-H3-Image
- https://huggingface.co/MiniMaxAI/MiniMax-H3-Image

## Other Open-Source Code Used

- https://github.com/Tencent/ncnn for fast neural network inference on all platforms
- https://github.com/futz12/ncnn_llm for BPE tokenizer
- https://github.com/webmproject/libwebp for WebP encoding and decoding
- https://github.com/libjpeg-turbo/libjpeg-turbo for JPEG encoding and decoding
- https://github.com/pnggroup/libpng for PNG encoding and decoding
- https://github.com/zlib-ng/zlib-ng for PNG compression

This software is based in part on the work of the Independent JPEG Group. Third-party license notices are included in [LICENSE](LICENSE).

## License

Project code is licensed under Apache-2.0. Model weights are distributed under the MiniMax-H3-Image Community License; see the original model repository.
