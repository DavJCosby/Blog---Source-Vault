---
author: David Cosby
published: 8.7.2023
tags: misc
--- 

> *`By: David Cosby`
> `8.7.2023`
> https://twitter.com/cosbdev*

A long-time dream of mine has been to make a game that uses shaders to display 3-dimensional worlds in a classy pixel-art style, much like what [T3ssell8r](https://www.youtube.com/@t3ssel8r) has been doing for a while now. The appeal was to introduce more advanced techniques from the world of 3D graphics into the pixel-art style (think lights and shadows, global illumination, refractive materials, etc.)

Unfortunately, this isn't the plan for the game I'm working on right now, but I just couldn't stop myself from playing with these ideas anyways. Today I'm going to share the approach I've come up with to create pixel art sprites and VFX for my game.

# Setting Up Blender
> [!warning]- Warning: Low-Effort Explanation Ahead!
> *To be honest, the blender setup is nothing fancy so I'm not going to go into too much detail in this section. If you have any questions beyond what's covered here, [feel free to reach out to me.](https://twitter.com/cosbdev)*

This post is going to be more about image processing than it is about the blender setup, but I'll quickly go over my steps for getting pixel art-ish looking renders out of blender.

1. First, I position an orthographic camera over the scene in a way I'd expect the camera to be positioned for my top-down game.
![[blender_setup1.png]]

2. I set the output resolution to `640x360` and the [pixel filter](https://docs.blender.org/manual/en/latest/render/cycles/render_settings/film.html#pixel-filter) to the lowest possible value to mitigate anti-aliasing.

3. Lastly, I turned on Freestyle line art to automatically apply outlines to objects in my scene. I'm not totally happy with the way these turn out sometimes, so I'll have to play with it more.

When we hit render, we get this:
> [!success]+ Output
> ![[demo1.png]]

Not bad for faux pixel art, but we can do much better.

# Matching to a Color Palette
Part of what gives pixel art its charm is the creative usage of a limited color palette. In the image above, there are hundreds of unique colors! Let's see if we can chop that down a bit.

Before we get into the nitty-gritty of image processing, lets set up everything we need to start reading and writing pixel values.

Ideally this would be a shader written in some sort of shading language, but since I'm trying to improve my rust skills and it's been a while since I've written shader code, I'm just going to do this all in rust. For image input and output I'm using the [Image](https://docs.rs/image/latest/image/) crate and for color manipulation I'm using [Palette](https://docs.rs/palette/latest/palette/).

```rust
use image::{io::Reader, ImageError, ImageBuffer};

fn main() -> Result<(), ImageError> {
	let img = Reader::open(IMAGE_PATH)?.decode()?;
	let pixels = img.as_rgba8().unwrap().enumerate_pixels();
	let mut output_buffer = ImageBuffer::new(img.width(), img.height());
	
	for pixel in pixels {
		// start doing witchcraft here!
	}
	output_buffer.save("img/output.png").unwrap();
	Ok(())
}
```

I've seen a lot of solutions to this limited color palette problem that just [quantize](https://en.wikipedia.org/wiki/Color_quantization)the image down until we've got the number of colors we're looking for. I feel like we could get so much character out of our final image if we used a proper pixel art color palette. Plus, for use in an actual video game, we need some promise of uniformity otherwise everything is just going to look like a mess.
## Bringing in the Color Palette
For this first demo, I'll be using this pretty little color palette I found on [lospec](https://lospec.com/), called [Archer48.](https://lospec.com/palette-list/archerer48)

![[archer_palette.png]]

To make this palette easy to swap in and out, I'll just be plugging the hex values into the code.
```rust
const PALETTE_HEX: [&str; 48] = [
	"1b112c", "413047", "543e54", "75596f", "91718b", "b391aa", "ccb3c6", "e3cfe3", "fff7ff", "fffbb5", "faf38e", "f7d076", "fa9c69", "eb7363", "e84545", "c22e53", "943054", "612147", "3d173c", "3f233c", "66334b", "8c4b63", "c16a7d", "e5959f", "ffccd0", "dd8d9f", "c8658d", "b63f82", "9e2083", "731f7a", "47195d", "2a143d", "183042", "1e5451", "2a6957", "3b804d", "5aa653", "86cf74", "caf095", "e0f0bd", "3f275e", "3f317a","3c548f", "456aa1", "4a84b0", "56aec4", "92d7d9", "c3ebe3",
];
```

Though if we actually want to use this data, we're going to have to convert the palette into RGB.

```rust
use palette::Srgb;

fn palette_as_rgb() -> Vec<Srgb> {
	let mut rgb_palette: Vec<Srgb> = vec![];
	for hex in PALETTE_HEX {
		let rgb = hex_to_rgb(hex).unwrap();
		rgb_palette.push(rgb);
	}
	rgb_palette
	}
	
fn hex_to_rgb(hex: &str) -> Result<Srgb, &'static str> {
	if hex.len() != 6 {
		return Err("Invalid hex color code");
	}
	
	let r = u8::from_str_radix(&hex[0..2], 16)
		.map_err(|_| "Invalid hex color code")?;
	let g = u8::from_str_radix(&hex[2..4], 16)
		.map_err(|_| "Invalid hex color code")?;
	let b = u8::from_str_radix(&hex[4..6], 16)
		.map_err(|_| "Invalid hex color code")?;
	
	Ok(Srgb::new(
		r as f32 / 255.0,
		g as f32 / 255.0,
		b as f32 / 255.0,
	))
}

fn main() -> Result<(), ImageError> {
	let img = Reader::open(IMAGE_PATH)?.decode()?;
	let pixels = img.as_rgba8().unwrap().enumerate_pixels();
	let mut output_buffer = ImageBuffer::new(img.width(), img.height());
	let palette_rgb = palette_as_rgb();
	
	for pixel in pixels {
		// is it witchcraft time yet?
	}
	output_buffer.save("img/output.png").unwrap();
	Ok(())
}
```

## Finding the Closest Palette Color

We're going to run a calculation on each pixel to help us figure out which color from our palette is the closest match. The easiest way to do this is to imagine our RGB color values as a 3D points, and then compare the distances between those points.

![[cartesian_colors_rgb.mkv]]

Whichever point is closest to our original pixel's RGB gets chosen to be the new color in the output image.

```rust
use palette::{Srgb, color_difference::EuclideanDistance};

fn find_closest(palette: &Vec<Srgb>, color: Srgb) -> Srgb {
	let mut dist_of_closest = std::f32::MAX;
	let mut closest = Srgb::new(0.0, 0.0, 0.0);
	
	for palette_color in palette {
		let d = color.distance_squared(*palette_color);
		if d < dist_of_closest {
			dist_of_closest = d;
			closest = *palette_color
		}
	}
	closest
}
```

Okay, now let's plug it into `main()` and see what kind of an output we get!

```rust

fn main() -> Result<(), ImageError> {
// --snip--
	for pixel in pixels {
		let (x, y) = (pixel.0, pixel.1);
		let [r, g, b, a] = pixel.2.0;
		let pixel_rgb = Srgb::new(
			r as f32 / 255.0,
			g as f32 / 255.0,
			b as f32 / 255.0
		);
		// find the closest color in the palette to the pixel at (x, y)
		let closest = find_closest(&palette_rgb, pixel_rgb);
		// output the new color to the buffer
		let output_pixel = output_buffer.get_pixel_mut(x, y);
		
		*output_pixel = image::Rgba([
			(closest.red * 255.0) as u8,
			(closest.green * 255.0) as u8,
			(closest.blue * 255.0) as u8,
			a,
		]);
	// save as an image
	output_buffer.save("img/output.png").unwrap();
	Ok(())
}
```

>[!success]+ Output
>![[output1-rgb-no-dither.png]]

Sweet, it works!

>[!info]- Compared to...
>![[demo1.png]]

# Better Color Matches with the Oklab Color Space
Before we get going on dithering, I want to try using [another color space](https://bottosson.github.io/posts/oklab/) to see if we can get more accurate palette mapping. The image above seems alright, but there are a few things with it that bug me. Notice the darker strand on the highlight on the cone? Also, what's up with the super vibrant reds on the ring? These issues stem from the fact that we're using the RGB Color space.

## What's Wrong with RGB?
As far as digital color models go, RGB is one of the most straight-forward. It's an additive color model, meaning its components more or less describe how much <mark style="background: #5c3f4f;">red</mark>, <mark style="background: #48584E;">green</mark>, and <mark style="background: #3F5865;">blue</mark> light needs to be added together to create a given color. For most use cases, it's totally fine.

The problem with it in *this* use case is the lack of perceptual uniformity.

Here's an example. Look at the difference between "true blue" and "true green" below.

>**Blue**
> `rgb(0, 0, 255)`: <mark style="background: #0000ff; color: #0000ff">Nice.</mark>
>**Green**
> `rgb(0, 255, 0)`: <mark style="background: #00ff00; color: #00ff00">Nice.</mark>

To the human eye, green is far brighter than blue. Yet numerically, both are equally far away from black.
> **Black**
> `rgb(0, 0, 0)`: <mark style="background: #000000; color: #000000">Nice.</mark>

**The RGB color space doesn't account for the inherent lightness or darkness of colors.** If we want our `find_closest()` function to return the color from our palette that really is the perceptually closest match to the reference color, we'll need to use a different color space.

## Introducing Oklab
Actually, I'll let Björn introduce it himself, since he made the thing.

> [!quote] Björn Ottosson
> A color in Oklab is represented with three coordinates, similar to how [CIELAB](https://en.wikipedia.org/wiki/CIELAB_color_space) works, but with better perceptual properties. Oklab uses a D65 whitepoint, since this is what sRGB and other common color spaces use. The three coordinates are:
> - **L** – perceived lightness
> - **a** – how green/red the color is
> - **b** – how blue/yellow the color is

Of course there's a lot more to it than that (in fact he goes through just about all of it on his [blog post](https://bottosson.github.io/posts/oklab/)) but the point is these three coordinates form a color model that is far more perceptually uniform than RGB. We can still treat the `L`, `a`, and `b` components the same way we did previously to imagine 3d points, expect now the distance between those points more closely resemble their perceptual distance.

![[cartesian_colors_lab.mkv]]

Here's another example to illustrate this idea of perceptual uniformity. In HSV (Hue/Saturation/Value, a color space built on top of RGB) the `V` is *supposed* to represent how bright or dark a color is, but it doesn't do a great job. If you were to lock `V` and spin the hue to create a rainbow, here's what you'd get: 

![bad rainbow](https://bottosson.github.io/img/oklab/hue_hsv.png)
Compare that to gigachad Oklab, when you lock `L` and manipulate[^1] `a` and `b` to create a rainbow, here's what we get.
![good rainbow](https://bottosson.github.io/img/oklab/hue_oklab.png)

It's a bit less intuitive of a color model to use, but I think the results speak for themselves.
## Okay We Get it, Just Program the Thing Already
Alright, alright. First, let's update `palette_as_rgb()` to give us our color palette in the OkLab color space.

```rust
use palette::{color_difference::EuclideanDistance, IntoColor, Srgb, Oklab};

fn palette_as_oklab() -> Vec<Oklab> {
    let mut oklab_palette: Vec<Oklab> = vec![];
    for hex in PALETTE_HEX {
        let rgb = hex_to_rgb(hex).unwrap();
        oklab_palette.push(rgb.into_color());
    }
    oklab_palette
}
```

`find_closest()` needs to know we're working with a different color space now, so let's update that.

```rust
fn find_closest(palette: &Vec<Oklab>, color: Oklab) -> Oklab {
	let mut dist_of_closest = std::f32::MAX;
	let mut closest = Oklab::new(0.0, 0.0, 0.0);
	
	for palette_color in palette {
		let d = color.distance_squared(*palette_color);
		if d < dist_of_closest {
			dist_of_closest = d;
			closest = *palette_color
		}
	}
	closest
}
```

And then `main()` needs some tweaks to read get our pixel values in that color space too.

```rust
fn main() -> Result<(), ImageError> {
	// --snip--
	let palette_rgb = palette_as_oklab();
	for pixel in pixels {
		let (x, y) = (pixel.0, pixel.1);
		let [r, g, b, a] = pixel.2 .0;
		
		let pixel_rgb = Srgb::new(
			r as f32 / 255.0,
			g as f32 / 255.0,
			b as f32 / 255.0
		);
		let pixel_oklab: Oklab = pixel_rgb.into_color();
		let closest_oklab = find_closest(&palette_oklab, pixel_oklab);
		let closest_rgb: Srgb = closest_oklab.into_color();
		
		let output_pixel = output_buffer.get_pixel_mut(x, y);
		*output_pixel = image::Rgba([
			(closest_rgb.red * 255.0) as u8,
			(closest_rgb.green * 255.0) as u8,
			(closest_rgb.blue * 255.0) as u8,
			a,
		]);
	}
	// --snip--
}
```

Okay, let's see what we get now.
>[!success]+ Output (Oklab)
>![[output2_oklab_no_dither.png]]

> [!failure]- Old Output (RGB)
> ![[output1-rgb-no-dither.png]]

> [!info]- Original
> ![[demo1.png]]

I'm biased, but I think that matches the original much better than the RGB-based solution. It's not perfect, but *a lot* better.

Those out-of-place vibrant red shades on the ring that I mentioned earlier are gone, and the highlights on the cone look much better. There was also a deep purple hue that snuck its way into the shadow of the monkey on the old output that's been better handled here. The shadows on the monkey look a bit better and the brightness of the floor seems much more true to the original scene.

I think we'll see even more benefits from this color model when we introduce dithering.
# Dithering Close Color Matches
[Dithering](https://en.wikipedia.org/wiki/Ordered_dithering) lets us smoothly transition between two colors without stepping outside the bounds of a color palette. It's a technique that's been used historically for pixel art and can add a lot of character to a design when used right.
![dithering example](https://upload.wikimedia.org/wikipedia/commons/e/e5/Ordered_4x4_Bayer_matrix_dithering.png)

This is particularly useful for us because sometimes the pixel we're reading falls pretty evenly between two colors in our palette. To get even better results, we can checkerboard pattern between the two colors.

I spent a long time trying to make this work on my own and found out that dithering is a lot more complicated than I thought. Eventually I stumbled across [this fantastic post](https://bisqwit.iki.fi/story/howto/dither/jy/) by Joel Yliluoma that dives deep into the problem and introduced me to a patented algorithm called Pattern Dithering.

That patent has since expired, so uh...
*yoink*

This implementation departs a little bit from the patented technique, so let me break down what all happens before I dump a wall of code in front of you.
1. First, we create an $n\times n$ [Bayer Matrix](https://en.wikipedia.org/wiki/Ordered_dithering#Threshold_map) which serves as our threshold map. Later, we'll create a list of candidates that can be our final pixel color. This map helps us know which one candidate to use, depending on the `(x, y)` location of the pixel.
   Higher resolution threshold maps will yield more dithering patterns and therefore smoother transitions, but increase the computation time. I personally think the ${2}\times {2}$ gives the best looking results for this application, but there's no reason you couldn't do ${4}\times {4}$ or higher.
```rust
const THRESHOLD_MAP: [[usize; 2]; 2] = [[0, 2], [3, 1]];
```
2. Next, we need to create an $n^{2}$ sized list of potential candidates for our final pixel color. When finding a color candidate, we use the same `find_closest()` function as before, but on subsequent iterations we apply a little bit of an offset to that color in the direction of the original pixel color before searching for a match. This offset, or "error" is scaled down by a value of our choice. I think multiplying it by around `0.04` looks best with this color palette, but each scene is different.
3. We then take that list of candidates and sort it by ascending lightness, or `L` to make sure our results are consistent.
4. The final pixel color is resolved by pulling from that sorted list of candidates like so:
```rust
const MAP_SIZE: usize = THRESHOLD_MAP.len();
let index = THRESHOLD_MAP[x % MAP_SIZE][y % MAP_SIZE];
let chosen_color: Srgb = candidates[index].into_color();
```

Put all together, here's what our new pixel iteration loop looks like.

```rust
const THRESHOLD_MAP: [[usize; 2]; 2] = [[0, 2], [3, 1]];
const MAP_SIZE: usize = THRESHOLD_MAP.len();
const DITHER: f32 = 0.04;

for pixel in pixels {
	let (x, y) = (pixel.0, pixel.1);
	let [r, g, b, a] = pixel.2 .0;
	// figure out the color of the pixel we're trying to match
	let pixel_rgb = Srgb::new(r as f32 / 255.0, g as f32 / 255.0, b as f32 / 255.0);
	// convert it to Oklab
	let pixel_oklab: Oklab = pixel_rgb.into_color();
	
	// create a list of potential color candidates from our palette
	let mut candidates: Vec<Oklab> = vec![];
	let mut error = Oklab::new(0.0, 0.0, 0.0);
	for _ in 0..MAP_SIZE.pow(2) {
		let sample = pixel_oklab + error * DITHER;
		let candidate = find_closest(&palette_oklab, sample);
		candidates.push(candidate);
		error += pixel_oklab - candidate;
	}

	candidates.sort_by(
		|Oklab { l: l1, .. }, Oklab { l: l2, .. }| 
			l1.partial_cmp(l2).unwrap()
	);

	let index = THRESHOLD_MAP[x as usize % MAP_SIZE][y as usize % MAP_SIZE];
	let chosen_color: Srgb = candidates[index].into_color();

	let output_pixel = output_buffer.get_pixel_mut(x, y);
	*output_pixel = image::Rgba([
		(chosen_color.red * 255.0) as u8,
		(chosen_color.green * 255.0) as u8,
		(chosen_color.blue * 255.0) as u8,
		a,
	]);
}
```

Okay, let's see what we get!
> [!success]+ Output
> ![[output3_oklab_dithered.png]]

>[!info]- Previous (Undithered, Oklab)
>![[output2_oklab_no_dither.png]]

>[!info]- Original
>![[demo1.png]]

# Dithering Transparency
As fun as it is to turn entire 3D scenes into pixel-art, I don't think the example we've been using is a very realistic use case. Instead, lets try making actual sprites with transparent background. Something we could realistically plop into a game.

The use case that comes to mind for me is to use this to create animated VFX, so I quickly threw together some flame simulations and rendered them at a low resolution.

>[!tip]+ Flame Simulation
>![[demo2.gif]]

While we're shaking things up, let's swap out the color palette too. I really like [this one by Luis Miguel Maldonado](https://lospec.com/palette-list/slso8), so let's give that a try!

![[SLS08_palette.png]]

>[!question]+ Output
>![[output4_no_alpha_dither.gif]]

> [!tip]- Bonus: New Color Palette on the Old Scene
> Just adding because I thought it looked nice.
![[output5_new_palette.png]]

Looks pretty neat! But I think you can probably tell why this section is titled what it is from the image above. What's the point of snapping the result to a color palette, if we're just going to have wild color variations in the smoke from transparency? Let's dither it!

There might be a better way to do this, but my solution was to duplicate a lot of the code we used for color candidates and apply it to the principle of transparency. Rather than matching the alpha channel to a palette of possible transparencies, I just do `alpha.round()` to snap it to either `0.0` or `1.0`. I also set up a custom dithering constant for transparency for extra control.

This post is already super long, so if you want to see exactly how I implemented it, you'll find the full source code at the bottom of the page.

>[!success]+ Output
>![[output6_alpha_dither.mkv]]
>*in a video format instead of a gif so you can scrub through and pixel peep.*

# Conclusion
Wow, this took way longer to write than I expected. Here's some final thoughts before I go.
- [*] This technique seems to work best on individual sprites and animations. Don't just plug it into your viewport and call it pixel art :P
- [*] Smaller color palettes seem to work better. If you're using a large color palette for your game, maybe you could feed the script a more tailored, smaller slice of your color palette.
- [*] As cool as this is, this is far from a replacement for actual art talent in my project. We've got a really solid team of talented individuals coming together for this game (I'll share more in a future blog posts) but we're a bit thin when it comes to illustration skills. Though this will help alleviate some of the art burden, it's not meant to replace artists.
- [b] I've got a ton more ideas on how I want to use this system to expand what's visually possible for my game. If you're interested in seeing what's next, please bookmark this page (I'll try to get a newsletter or something set up soon) and check up on me in a little while. I'll post an update on my twitter account whenever I've got something new for you guys.

Thanks for reading! This is my first time writing a blog post so I'm sure it's full of all sorts of issues. If you have any questions or would like to make some suggestions, [please shoot me a message](https://twitter.com/cosbdev). If you'd like to support me, you can follow me on [twitter](https://twitter.com/cosbdev) or use the link below to buy me a boba.

<a href="https://www.buymeacoffee.com/davidcosby"><img src="https://img.buymeacoffee.com/button-api/?text=Buy me a boba&emoji=🧋&slug=davidcosby&button_colour=d699b6&font_colour=293136&font_family=Lato&outline_colour=293136&coffee_colour=d3c6aa" style="max-width: 16em"/></a>

## Final Source Code
```rust
use image::{io::Reader, ImageBuffer, ImageError};
use palette::{color_difference::EuclideanDistance, IntoColor, Oklab, Srgb};
// read/write filepaths
const INPUT_PATH: &str = "img/input.png";
const OUTPUT_PATH: &str = "img/output.png";
// Bayer matrix definition
const THRESHOLD_MAP: [[usize; 2]; 2] = [[0, 2], [3, 1]];
const MAP_SIZE: usize = THRESHOLD_MAP.len();
// Dithering controls for color and transparency
const COLOR_DITHER: f32 = 0.04;
const ALPHA_DITHER: f32 = 0.12;
// Color palette of choice
const PALETTE_HEX: [&str; 8] = [
    "0d2b45", "203c56", "544e68", "8d697a", "d08159", "ffaa5e", "ffd4a3", "ffecd6",
];

fn palette_as_oklab() -> Vec<Oklab> {
    let mut oklab_palette: Vec<Oklab> = vec![];
    for hex in PALETTE_HEX {
        let rgb = hex_to_rgb(hex).unwrap();
        oklab_palette.push(rgb.into_color());
    }
    oklab_palette
}

fn hex_to_rgb(hex: &str) -> Result<Srgb, &'static str> {
    if hex.len() != 6 {
        return Err("Invalid hex color code");
    }
	
    let r = u8::from_str_radix(&hex[0..2], 16).map_err(|_| "Invalid hex color code")?;
    let g = u8::from_str_radix(&hex[2..4], 16).map_err(|_| "Invalid hex color code")?;
    let b = u8::from_str_radix(&hex[4..6], 16).map_err(|_| "Invalid hex color code")?;
	
    Ok(Srgb::new(
        r as f32 / 255.0,
        g as f32 / 255.0,
        b as f32 / 255.0,
    ))
}

fn find_closest(palette: &Vec<Oklab>, color: Oklab) -> Oklab {
    let mut dist_of_closest = std::f32::MAX;
    let mut closest = Oklab::new(0.0, 0.0, 0.0);
	
    for palette_color in palette {
        let d = color.distance_squared(*palette_color);
        if d < dist_of_closest {
            dist_of_closest = d;
            closest = *palette_color
        }
    }
    closest
}

fn main() -> Result<(), ImageError> {
    // import color palette
    let palette_oklab = palette_as_oklab();
    // import source image
    let img = Reader::open(INPUT_PATH)?.decode()?;
    let pixels = img.as_rgba8().unwrap().enumerate_pixels();
    // create a place to store output pixels
    let mut output_buffer = ImageBuffer::new(img.width(), img.height());
	
    for pixel in pixels {
        let (x, y) = (pixel.0, pixel.1);
        let [r, g, b, a] = pixel.2 .0;
        let alpha_f32 = (a as f32) / 255.0;
        // cast pixel to oklab
        let pixel_rgb = Srgb::new(r as f32 / 255.0, g as f32 / 255.0, b as f32 / 255.0);
        let pixel_oklab: Oklab = pixel_rgb.into_color();
        // create a list of candidate color and alpha values
        let mut candidates_c: Vec<Oklab> = vec![];
        let mut candidates_a: Vec<f32> = vec![];
        let mut error_c = Oklab::new(0.0, 0.0, 0.0);
        let mut error_a = 0.0;
        for _ in 0..MAP_SIZE.pow(2) {
            // color
            let sample_c = pixel_oklab + error_c * COLOR_DITHER;
            let candidate_c = find_closest(&palette_oklab, sample_c);
            candidates_c.push(candidate_c);
            error_c += pixel_oklab - candidate_c;
			
            // alpha
            let sample_a = alpha_f32 + error_a * ALPHA_DITHER;
            let candidate_a = sample_a.round();
            candidates_a.push(candidate_a);
            error_a += alpha_f32 - candidate_a;
        }
		
        // sort candidates by brightness and alpha, respectively
        candidates_c
            .sort_by(|Oklab { l: l1, .. }, Oklab { l: l2, .. }| l1.partial_cmp(l2).unwrap());
        candidates_a.sort_by(|a1, a2| a1.partial_cmp(&a2).unwrap());

        // choose a candidate based on the pixel coordinates
        let index = THRESHOLD_MAP[x as usize % MAP_SIZE][y as usize % MAP_SIZE];
        let chosen_color: Srgb = candidates_c[index].into_color();
        let chosen_alpha = candidates_a[index];
		
        // write new pixel color to the output buffer
        let output_pixel = output_buffer.get_pixel_mut(x, y);
        *output_pixel = image::Rgba([
            (chosen_color.red * 255.0) as u8,
            (chosen_color.green * 255.0) as u8,
            (chosen_color.blue * 255.0) as u8,
            (chosen_alpha * 255.0) as u8,
        ]);
    }
	
    // save as a png
    output_buffer.save(OUTPUT_PATH).unwrap();
    Ok(())
}
```


[^1]: Just as RGB has color spaces like HSV and HSL to manipulate hue, OkLab has OkHsv and OkHsl. [Read more here.](https://bottosson.github.io/posts/colorpicker/)