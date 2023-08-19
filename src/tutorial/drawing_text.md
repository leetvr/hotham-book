# Drawing Text

Drawing text in a 3D world is not a simple matter.  There are a number of approaches, but the most common is to rasterize the text/font data into a texture of suitable resolution for display on something like a mesh plane, or a more complex mesh such as a curved screen.  This can then be parented to the display for example if you want an object which is always in front of the user's eyes, or it can be parented to a specific object to serve the purposes of a billboard.

Building on the example of dynamic texturing we provided in the last section, I'll be presenting an example of a reasonably performant implementation of updating a buffer with text.  I have tested this on an Oculus Quest 2 headset and found that the text updates so quickly, despite the frame by frame texture uploading and continuous polling of the input devices, that the changing text itself will blur into the data displayed the next frame.

For this example, we'll be using the `ab_glyph` library, which supports the loading of several different font formats.  It is also important to know a bit about how typography works.  Apple's developer website has a [good overview of typographical concepts](https://developer.apple.com/library/archive/documentation/TextFonts/Conceptual/CocoaTextArchitecture/TypoFeatures/TextSystemFeatures.html) in their discussion of the Cocoa texture architecture.  The concepts will serve us well for this tutorial.

Each glyph of a font has a baseline from which part of the character rises above the baseline, called the *ascent*, and part of the character dips below the baseline, by a distance called the *descent*.  As each character of a font may be of difference size, each glyph has its own *bounding box*.  Finally, there is a distance which the printer should move left or right, depending on the directionality of the font, known as the *advance*.

Each pixel of a glyph may be of a specific *coverage*.  This can be up to 64 separate levels of color, from the absence of a pixel ("black") to its complete presence ("white").  To correctly handle colored text, we need to be able to interpolate between the background color we are drawing onto, and the foreground color, based on the degree of coverage.

To begin with, lets create a representation of a color where the individual components are `f32` between 0 and 255, for use with the drawing library.

```rust,noplayground
#[derive(Debug,Clone,Copy)]
pub struct FontColor {
    r: f32,
    g: f32,
    b: f32,
    a: f32,
}

impl FontColor {
    #[inline]
    pub fn new(r: f32, g: f32, b: f32, a: f32) -> Self {
		Self {
			r,
			g,
			b,
			a,
		}
    }

    #[inline]
    pub fn from_slice(color_slice: &[u8]) -> Self {
		Self {
			r: color_slice[0] as f32,
			g: color_slice[1] as f32,
			b: color_slice[2] as f32,
			a: color_slice[3] as f32,
		}
    }

    #[inline]
    pub fn interpolate(&self, bgcolor: Self, factor: f32) -> Self {
		Self {
			r: (self.r-bgcolor.r)*factor+bgcolor.r,
			g: (self.g-bgcolor.g)*factor+bgcolor.g,
			b: (self.b-bgcolor.b)*factor+bgcolor.b,
			a: self.a,
		}
    }
}
```

To make this slightly more performant, because the functions we'll be using are simple expressions without any conditional logic, I've specified the `#[inline]` directive to suggest that the compiler optimise these functions to avoid a far call to external code every time they are used, as these functions will be used quite regularly.

The interpolate function will produce a color close to or equal to bgcolor as the coverage factor passed in approaches zero.  As coverage approaches 1.0, it will reach a maximum of the foreground color itself.

Here is the code for an example draw function which I will break down shortly.

```rust,noplayground
    pub fn draw_buffer(&self, text: String, pt_size: f32, x: u32, y: u32, buff: &mut [u8], extent: vk::Extent2D) {
		let fontref = self.fontvec.as_ref().unwrap();
		let scaledfont=fontref.as_scaled(pt_size);
		let ascent=scaledfont.ascent();
		let descent=scaledfont.descent();
		let glyph_height=ascent-descent;

	    let mut xposn=x;
		let mut yposn=y+(ascent as u32);
		// To place a dot directly to the left of the character, at the
		// baseline, uncomment the below two lines, so you can see the
		// relationship between the different parameters.
		//let tmp_posn=((xposn-1+yposn*extent.width)*4) as usize;
		//(&mut buff[tmp_posn..tmp_posn+4]).copy_from_slice(&[0,0,255,255]);
	   	for c in text.chars() {
			let gid = fontref.glyph_id(c);
			let g = gid.with_scale(pt_size);
			let advance=scaledfont.h_advance(gid);

	        if let Some(g_outline) = fontref.outline_glyph(g) {
				let glyph_bounds = g_outline.px_bounds();
	    
	            let mut glyph_top_x = xposn;
				let mut glyph_top_y = yposn;
				
				if glyph_bounds.min.x < 0. {
					glyph_top_x -= glyph_bounds.min.x.abs() as u32;
				} else {
					glyph_top_x += glyph_bounds.min.x as u32;
				}

	            if glyph_bounds.min.y < 0. {
					glyph_top_y -= glyph_bounds.min.y.abs() as u32;
				} else {
					glyph_top_y += glyph_bounds.min.y as u32;
				}

	            g_outline.draw(|x1, y1, cov| {
					let buf_position: usize = ((glyph_top_x+x1 + (glyph_top_y+y1) * extent.width)* 4) as usize;
					let buf_slice = &mut buff[buf_position..buf_position+4];
					if cov >= 0.0 {
						let bgcolor=FontColor::from_slice(&buf_slice);
						let interp = self.fgcolor.interpolate(bgcolor,cov);
						buf_slice.copy_from_slice(&[interp.r as u8,interp.g as u8,interp.b as u8,interp.a as u8]);
					};
				});
			} else {
				println!("Could not get outline for glyph!");
			}
			xposn+=(advance as u32);
		};
    }
```

In this function, &self is the Font object which we provided some initial code for in tutorial 1 [Getting Started](getting_started.md). We unwrap the font, scale it, and get the ascent and descent of the tallest characters in the font.  Because in this example we want the top of the characters to appear at the coordinates we specified, the baseline position is stored in `xposn`, `yposn`.  The `px_bounds()` function, which is different to the `glyph_bounds()` function, returns a conservative pixel bounding box whose coordinates are calculated relative to the baseline as the origin.

Because the bounding box coordinates can be positive or negative, and our `xposn` and `yposn` are `u32`, we ensure the `f32` is positive before casting to throw away the fractional part.

Note that this example code does not cover kerning, or advancing the cursor to the next line if the string passes the extent width.  It does nothing but lay out each character within the buffer in a linear fashion, interpolating the background color already stored in the buffer with the foreground color saved within the font struct.  It also does no error checking to ensure the string does not overflow the end of the buffer.  Calculating whether the text will overflow the buffer will depend on knowing your layout strategy as well as the width and height of each glyph.

I leave it as an exercise for the reader to sanitize this code to prevent the program from panicing due to an invalid index into the slice.  I also recommend removing the `println!()` call within the function, as it is incredibly common for a glyph (such as a space) to not appear in a font; in these cases this example code simply ignores the non existent character as if it did not exist.

As a final exercise we'll add a function that will be called each tick to test how performant this code is.

```rust,noplayground
fn update_dynscreen(engine: &mut Engine, state: &mut State) {
    let dyn_texture = state.dyn_texture.as_ref().unwrap();
    let mut vec_buff = state.texture_buffer.as_mut().unwrap(); 
    let mut text_slice=vec_buff.as_mut_slice();
    let mut n: u32 = 0;
    let mut m: u32 = 0;

    while m < 1024 {
		while n < 1024 {
			let c = ((255*n/1024) & 255) as u8;
			let slice_index = (n*4+m*4096) as usize;
			text_slice[slice_index..slice_index+4].copy_from_slice(&[c,0,0,255]);
			n += 1;
		}
		n = 0;
		m += 1;
    }

    state.fontref.draw_buffer("Hello Hotham 1234".to_string(), 18.0, 10, 10, text_slice, dyn_texture.image.extent);
    let affine1=engine.input_context.right.stage_from_grip();
    let (_, rotation, translation) = affine1.to_scale_rotation_translation();
    let (axis, angle) = rotation.to_axis_angle();
    let fmt1 = format!("Right Grip Rotation: {:.2} deg around {:.2?}", angle.to_degrees(), axis );
    let fmt2 = format!("Right Grip Translation: {:.2?}", translation);
    let affine2=engine.input_context.left.stage_from_grip();
    let (_, rotation, translation) = affine2.to_scale_rotation_translation();
    let (axis, angle) = rotation.to_axis_angle();
    let fmt3 = format!("Left Grip Rotation: {:.2} deg around {:.2?}", angle.to_degrees(), axis );
    let fmt4 = format!("Left Grip Translation: {:.2?}", translation);
    let affine3 = engine.world.get::<&LocalTransform>(engine.hmd_entity).unwrap();
    let (rotation, translation) = (affine3.rotation, affine3.translation);
    let (axis, angle) = rotation.to_axis_angle();
    let fmt5 = format!("HMD Rotation: {:.2} deg around {:.2?}", angle.to_degrees(), axis );
    let fmt6 = format!("HMD Translation: {:.2?}", translation);
    
    state.fontref.draw_buffer(fmt1, 18.0, 10, 30, text_slice, dyn_texture.image.extent);
    state.fontref.draw_buffer(fmt2, 18.0, 10, 50, text_slice, dyn_texture.image.extent);
    state.fontref.draw_buffer(fmt3, 18.0, 500, 30, text_slice, dyn_texture.image.extent);
    state.fontref.draw_buffer(fmt4, 18.0, 500, 50, text_slice, dyn_texture.image.extent);
    state.fontref.draw_buffer(fmt5, 18.0, 10, 70, text_slice, dyn_texture.image.extent);
    state.fontref.draw_buffer(fmt6, 18.0, 10, 90, text_slice, dyn_texture.image.extent);
    state.fontref.draw_buffer(format!("A {} B {} X {} Y {}", 
		engine.input_context.right.a_button(),
		engine.input_context.right.b_button(),
		engine.input_context.left.x_button(),
		engine.input_context.left.y_button()), 
		18.0, 10, 110, text_slice, dyn_texture.image.extent);
    state.fontref.draw_buffer(format!("Grips: Left {} Right {}", engine.input_context.left.grip_button(),
				      engine.input_context.right.grip_button()), 18.0, 10, 130, text_slice, dyn_texture.image.extent);
    state.fontref.draw_buffer(format!("Triggers: Left {} Right {}", engine.input_context.left.trigger_button(),
				      engine.input_context.right.trigger_button()), 18.0, 10, 150, text_slice, dyn_texture.image.extent);
    engine.vulkan_context.upload_image(text_slice, 1, vec![0], &(dyn_texture.image));
}
```
