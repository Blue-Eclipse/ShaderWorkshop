The shader we will be replicating is the shader [Digital Brain](https://www.shadertoy.com/view/4sl3Dr) by srtuss in Shadertoy. 
# What is Shadertoy?
Shadertoy is a website for coding and publishing pixel shaders to be showcased online. Shadertoy uses its own coding language instead of GLSL. Much of the syntax is identical but is still different as Shadertoy is for pixel shaders. It only has one function for the output unlike the two in GLSL (Vertex and Fragment). This is because Shadertoy is only concerned with the final output of the screen, not with any geometry. Shadertoy additionally uses its own set of input variables for shaders. 
You can access their website at [Shadertoy BETA](https://www.shadertoy.com/) 

`mainImage()` is the only output function in Shadertoy. It takes `fragCoord` as input and provides `fragColor` as output. `fragCoord` is an internal variable provided by Shadertoy, providing the coordinates of each pixel in the screen. `fragColor` is the color that a pixel needs to display. Shadertoy uses the RGBA system, but normalizes the vector to 1. As such, all values are between 0-1.0 instead of 0-255. The Alpha channel does nothing in Shadertoy, so it can be kept as 1.0. 

The basic Shadertoy code to get an output is as follows:
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    fragColor = vec4(0.0, 0.0, 0.0, 1.0);
}
```
The above code will simply display a black screen. Try adjusting the first three values of `fragColor` to get different colors in the output.
# Basic Setup
To begin, select "New" in Shadertoy. Shadertoy provides the following code to start:
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = fragCoord/iResolution.xy;

    // Time varying pixel color
    vec3 col = 0.5 + 0.5*cos(iTime+uv.xyx+vec3(0,2,4));

    // Output to screen
    fragColor = vec4(col,1.0);
}
```

We'll start with the following code as our foundation:
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = fragCoord/iResolution.xy;

    // Output to screen
    fragColor = vec4(uv, 0.0, 1.0);
}
```

>Note: We use swizzling to directly use the variable `uv` to provide two scalar values.

We will modify the code to make some basic adjustments to our canvas:
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = fragCoord.xy / iResolution.xy;
	//Shifting origin to center of Canvas
	uv = (uv - 0.5) * 2.0;
	//Adjusting for aspect ratio
	uv.x *= iResolution.x / iResolution.y;

    // Output to screen
    fragColor = vec4(uv, 0.0,1.0);
}
```
### Code Breakdown
The coordinates are first normalized. This makes the coordinate values range from 0-1.  This allows us to keep effects more consistent regardless of actual resolution. 
```Shadertoy
vec2 uv = fragCoord.xy / iResolution.xy;
```

We will shift the origin to the center of the canvas and make the axes range from -1 to 1 with the following code:
```Shadertoy
uv = (uv - 0.5) * 2.0;
```

The canvas can still appear stretched due to the aspect ratio. We eliminate the stretching with the following code:
```Shadertoy
uv.x *= iResolution.x / iResolution.y;
```

# Voronoi Noise Generator
We need a noise generator to generate our pattern. We will be using Voronoi Noise (which is based on Voronoi Diagrams) for our purpose. 

More info on Voronoi Noise [here](https://builtin.com/data-science/voronoi-diagram)

For Voronoi Noise, we first need a function to generate 2D random numbers.
```Shadertoy
// 2D random numbers
vec2 rand2(in vec2 p)
{
	return fract(vec2(sin(p.x * 591.32 + p.y * 154.077), cos(p.x * 391.32 + p.y * 49.077)));
}
```
We then use it to generate Voronoi Noise. 
```Shadertoy
// voronoi distance noise
float voronoi(in vec2 x)
{
	vec2 p = floor(x);
	vec2 f = fract(x);
	
	vec2 res = vec2(8.0);
	for(int j = -1; j <= 1; j ++)
	{
		for(int i = -1; i <= 1; i ++)
		{
			vec2 b = vec2(i, j);
			vec2 r = vec2(b) - f + rand2(p + b);
			
			// chebyshev distance, one of many ways to do this
			float d = max(abs(r.x), abs(r.y));
			
			if(d < res.x)
			{
				res.y = res.x;
				res.x = d;
			}
			else if(d < res.y)
			{
				res.y = d;
			}
		}
	}
	return res.y - res.x;
}
```
The Voronoi Noise can be calculated with any kind of distance function. In our case, we will be using the Chebyshev distance formula.  
More information on the various distance formulas that exist can be found [here](https://medium.com/@eskandar.sahel/exploring-common-distance-measures-for-machine-learning-and-data-science-a-comparative-analysis-ea0216c93ba3)
# Generating the Pattern
We will modify our shader as follows to generate the pattern we need.
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = fragCoord.xy / iResolution.xy;
	//Shifting origin to center of Canvas
	uv = (uv - 0.5) * 2.0;
	//Adjusting for aspect ratio
	uv.x *= iResolution.x / iResolution.y;
	
	//Generating the Pattern
	float v = 0.0;
	// add some noise octaves
	float a = 0.6, f = 1.0;
	float flicker = noise1(iTime * 2.0) * 0.8 + 0.4;
	
	for(int i = 0; i < 3; i ++) // 4 octaves also look nice, its getting a bit slow though
	{	
		float v1 = voronoi(uv * f + 5.0);
		float v2 = 0.0;
		
		// make the moving electrons-effect for higher octaves
		if(i > 0)
		{
			// of course everything based on voronoi
			v2 = voronoi(uv * f * 0.5 + 50.0 + iTime);
			
			float va = 0.0, vb = 0.0;
			va = 1.0 - smoothstep(0.0, 0.1, v1);
			vb = 1.0 - smoothstep(0.0, 0.08, v2);
			v += a * pow(va * (0.5 + vb), 2.0);
		}
		
		// make sharp edges
		v1 = 1.0 - smoothstep(0.0, 0.3, v1);
		
		// noise is used as intensity map
		v2 = a * (noise1(v1 * 5.5 + 0.1));
		
		// octave 0's intensity changes a bit
		if(i == 0)
			v += v2 * flicker;
		else
			v += v2;
		
		f *= 3.0;
		a *= 0.7;
	}
	
    // Output to screen
    fragColor = vec4(v, 0.0, 0.0,1.0);
}
```
## Code Breakdown
### Basic Pattern 
We first generate the Voronoi Noise pattern. We modify the canvas with a zoom (by multiplying `f`) and an offset (`0.5`) before applying the Voronoi noise.
```Shadertoy
float v1 = voronoi(uv * f + 5.0);
```

The above code gives us colored cells, but we need the outline of the cells for our pattern, not the cells themselves. Highlighting the border of the cells from Voronoi Noise. 
```Shadertoy
v1 = 1.0 - smoothstep(0.0, 0.3, v1);
```
We use the Smooth Step function to fill the cells of the Voronoi Noise with color.
```Shadertoy
v1 = smoothstep(0.0, 0.3, v1);
```
We then subtract it from 1 to invert where the colors are. This puts the color from the cell body to the cell wall.  
```Shadertoy
v1 = 1.0 - smoothstep(0.0, 0.3, v1);
```
### Creating the "Electron" Effect
The following code is responsible for creating the flow of "electrons" along the pattern lines. It is only applied to the patterns generated after the first iteration.
```Shadertoy
if(i > 0)
{
	// of course everything based on voronoi
	v2 = voronoi(uv * f * 0.5 + 50.0 + iTime);
	
	float va = 0.0, vb = 0.0;
	va = 1.0 - smoothstep(0.0, 0.1, v1);
	vb = 1.0 - smoothstep(0.0, 0.08, v2);
	v += a * pow(va * (0.5 + vb), 2.0);
}
```

We need two different copies of the Voronoi Pattern to create this effect. We assign the first Voronoi noise pattern we made to `va`.
```Shadertoy
va = 1.0 - smoothstep(0.0, 0.1, v1);
```
We create a second pattern and assign it to `vb`.
```Shadertoy
v2 = voronoi(uv * f * 0.5 + 50.0 + iTime);
vb =  1.0 - smoothstep(0.0, 0.08, v2);
```
Multiplying `va` into `vb` gives us the basic electricity effect.
```Shadertoy
v += va * vb;
```
By adding 0.5 to `vb`, we can make sure that all parts of `va` are visible when multiplied. 
```Shadertoy
v += va * (vb+0.5);
```
Taking the power basically reduces any value below 1 and does nothing to any value above (because color is between 0 and 1).
```Shadertoy
v += pow(va * (vb+0.5), 2.0);
```
Multiplying a number less than one (here `a`) does the same thing as well.
```Shadertoy
v += a * pow(va * (vb+0.5), 2.0);
```
### Completing the pattern for one iteration
We complete the pattern for one iteration with the following code:
```Shadertoy
// make sharp edges
v1 = 1.0 - smoothstep(0.0, 0.3, v1);

// noise is used as intensity map
v2 = a * (noise1(v1 * 5.5 + 0.1));

// octave 0's intensity changes a bit
if(i == 0)
	v += v2 * flicker;
else
	v += v2;
```
We make `v1` into the line pattern. 
```Shadertoy
v1 = 1.0 - smoothstep(0.0, 0.3, v1);
```
Using noise as an intensity map, we add more detail to the lines. 
```Shadertoy
 v2 = a * (noise1(v1 * 5.5 + 0.1));
```
Here `a` controls the final brightness of the image. We amplify `v1` with the multiplication and addition. Then its fed through the `noise1()` function. 

I am unable to properly decipher the noise function. Particularly why it generates a layered affect that is consistent. It is indeed noise, but it seems to be in discreet pockets. Maybe the one being added in the `rand` function is the key to the consistency. 
```Shadertoy
// 1D noise
float noise1(float p)
{
	float fl = floor(p);
	float fc = fract(p);
	return mix(rand(fl), rand(fl + 1.0), fc);
}
```
The reason for the consistency seems lies in how the noise is calculated with the `mix()` function. This function is used to interpolate between two values.
```Shadertoy
return mix(rand(fl), rand(fl + 1.0), fc);
```
Also, `rand` is not actually truly random, its pseudo-random and it gives the same result if the same value is used as input. This can be seen by setting the interpolation amount to 0 or 1 and running the result as the color. The solid unit must be the result of the `floor()` and `ceil()` we call in `noise1()` or it could be the result of the `rand()` function. 
```Shadertoy
// 1D random numbers
float rand(float n)
{
    return fract(sin(n) * 43758.5453123);
}
```
The variable `flicker` does exactly like it says. The noise function makes it so that if assigned to a value, it makes an object twinkle.  We apply the flicker to the first completed iteration only. 
```Shadertoy
//Generating the value
float flicker = noise1(iTime * 2.0) * 0.8 + 0.4;

//Applying the flicker
if(i == 0)
	v += v2 * flicker;
else
	v += v2;
```
### Iterating the effect
Artistic shaders can be made to appear more complicated by modifying and iterating the effect. This is why we were modifying `uv` when calling the Voronoi Function. As we iterating, shifting the canvas each time will add variation. In our case, we modify `a` and `f` at the end of the loop. `a` will become smaller each loop, making the new layer less bright. `f` will zoom into the canvas each time. This will give an almost fractal like effect when the effects are added together. 
```Shadertoy
f *= 3.0;
a *= 0.7;
```
# Camera Movement
We need the following function declared first to assist with rotating the canvas.
```Shadertoy
// rotate position around axis
vec2 rotate(vec2 p, float a)
{
	return vec2(p.x * cos(a) - p.y * sin(a), p.x * sin(a) + p.y * cos(a));
}
```
Next we will add the camera movement with the following code: 
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    // Normalized pixel coordinates (from 0 to 1)
    vec2 uv = fragCoord.xy / iResolution.xy;
	//Shifting origin to center of Canvas
	uv = (uv - 0.5) * 2.0;
	//Adjusting for aspect ratio
	uv.x *= iResolution.x / iResolution.y;
	
	// Camera movement
	uv *= 0.6 + sin(iTime * 0.1) * 0.4;
	uv = rotate(uv, sin(iTime * 0.3) * 1.0);
	uv += iTime * 0.4;
	
	//Generating the Pattern
	float v = 0.0;
	// add some noise octaves
	float a = 0.6, f = 1.0;
	float flicker = noise1(iTime * 2.0) * 0.8 + 0.4;
	
	for(int i = 0; i < 3; i ++) // 4 octaves also look nice, its getting a bit slow though
	{	
		float v1 = voronoi(uv * f + 5.0);
		float v2 = 0.0;
		
		// make the moving electrons-effect for higher octaves
		if(i > 0)
		{
			// of course everything based on voronoi
			v2 = voronoi(uv * f * 0.5 + 50.0 + iTime);
			
			float va = 0.0, vb = 0.0;
			va = 1.0 - smoothstep(0.0, 0.1, v1);
			vb = 1.0 - smoothstep(0.0, 0.08, v2);
			v += a * pow(va * (0.5 + vb), 2.0);
		}
		
		// make sharp edges
		v1 = 1.0 - smoothstep(0.0, 0.3, v1);
		
		// noise is used as intensity map
		v2 = a * (noise1(v1 * 5.5 + 0.1));
		
		// octave 0's intensity changes a bit
		if(i == 0)
			v += v2 * flicker;
		else
			v += v2;
		
		f *= 3.0;
		a *= 0.7;
	}
	
    // Output to screen
    fragColor = vec4(v, 0.0, 0.0,1.0);
}
```
### Code Breakdown
First, we zoom in and out of the canvas.
```Shadertoy
uv *= 0.6 + sin(iTime * 0.1) * 0.4;
```
The number multiplied to the time controls how fast we zoom in and out. 
The number multiplied to sine is the magnitude of the zoom. 
The number we add before the multiplication is an offset to make sure the sin value never goes into the negatives (Zoom shouldn't be negative). In our case, 0.4 is the minimum we need to add. 

This rotates the canvas along the origin. The equations for x and y are equations to apply the transform for rotating an x-y point about the origin. 
Source: [math.stackexchange.com/questions/17246/is-there-a-way-to-rotate-the-graph-of-a-function](https://math.stackexchange.com/questions/17246/is-there-a-way-to-rotate-the-graph-of-a-function)
[khanacademy.org/computing/pixar/sets/rotation/v/set-7](https://www.khanacademy.org/computing/pixar/sets/rotation/v/set-7)
[khanacademy.org/computing/pixar/sets/rotation/v/sets-8](https://www.khanacademy.org/computing/pixar/sets/rotation/v/sets-8)
[khanacademy.org/computing/pixar/sets/rotation/v/sets-9](https://www.khanacademy.org/computing/pixar/sets/rotation/v/sets-9)

> Note: We can also find the rotation equation using a linear transformation. It might be shorter to use that than the brute force trigonometric way. But it might also be more complex in terms of understanding and need a foundation in linear algebra.
> Source: [khanacademy.org/math/linear-algebra/matrix-transformations/lin-trans-examples/v/linear-transformation-examples-rotations-in-r2](https://www.khanacademy.org/math/linear-algebra/matrix-transformations/lin-trans-examples/v/linear-transformation-examples-rotations-in-r2) 

```Shadertoy
// rotate position around axis
vec2 rotate(vec2 p, float a)
{
	return vec2(p.x * cos(a) - p.y * sin(a), p.x * sin(a) + p.y * cos(a));
}
```
The float value is the amount by which the function should rotate the canvas. Shadertoy (and GLSL) uses radians for trigonometric functions.  

We call rotate with a sine function providing the angle. This allows the canvas to oscillate both clockwise and counterclockwise. The number multiplied to the time will determine how fast the effect is. the number multiplied to the sin will determine how much the canvas will rotate in an instant.
```Shadertoy
uv = rotate(uv, sin(iTime * 0.3) * 1.0);
```

This is a shift of the canvas in the positive XY direction.
```Shadertoy
uv += iTime * 0.4;
```
The number multiplied affects how fast the shift is. 
# Applying Color
The final code for with the color applied is as follows:
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    float flicker = noise1(iTime * 2.0) * 0.8 + 0.4;

    vec2 uv = fragCoord.xy / iResolution.xy;
	uv = (uv - 0.5) * 2.0;
	vec2 suv = uv;
	uv.x *= iResolution.x / iResolution.y;
	
	
	float v = 0.0;
	
	// that looks highly interesting:
	//v = 1.0 - length(uv) * 1.3;
	
	
	// a bit of camera movement
	uv *= 0.6 + sin(iTime * 0.1) * 0.4;
	uv = rotate(uv, sin(iTime * 0.3) * 1.0);
	uv += iTime * 0.4;
	
	
	// add some noise octaves
	float a = 0.6, f = 1.0;
	
	for(int i = 0; i < 3; i ++) // 4 octaves also look nice, its getting a bit slow though
	{	
		float v1 = voronoi(uv * f + 5.0);
		float v2 = 0.0;
		
		// make the moving electrons-effect for higher octaves
		if(i > 0)
		{
			// of course everything based on voronoi
			v2 = voronoi(uv * f * 0.5 + 50.0 + iTime);
			
			float va = 0.0, vb = 0.0;
			va = 1.0 - smoothstep(0.0, 0.1, v1);
			vb = 1.0 - smoothstep(0.0, 0.08, v2);
			v += a * pow(va * (0.5 + vb), 2.0);
		}
		
		// make sharp edges
		v1 = 1.0 - smoothstep(0.0, 0.3, v1);
		
		// noise is used as intensity map
		v2 = a * (noise1(v1 * 5.5 + 0.1));
		
		// octave 0's intensity changes a bit
		if(i == 0)
			v += v2 * flicker;
		else
			v += v2;
		
		f *= 3.0;
		a *= 0.7;
	}

	// slight vignetting
	v *= exp(-0.6 * length(suv)) * 1.2;
	
	// use texture channel0 for color? why not.
	vec3 cexp = texture(iChannel0, uv * 0.001).xyz * 3.0 + texture(iChannel0, uv * 0.01).xyz;//vec3(1.0, 2.0, 4.0);
	cexp *= 1.4;
	
	// old blueish color set
	//vec3 cexp = vec3(6.0, 4.0, 2.0);
	
	vec3 col = vec3(pow(v, cexp.x), pow(v, cexp.y), pow(v, cexp.z)) * 2.0;
	
	fragColor = vec4(col, 1.0);
}
```
We need to set Texture Channel 0. To do this, scroll down to iChannel0. Select it, go to Textures and choose RGBA noise small. An cog should appear next to iChannel0 ensure that VFlip is not checked. 
## Code Breakdown
Vignetting is the slight fading that happens as we move towards the frame of an image. 
```Shadertoy
    // slight vignetting
	v *= exp(-0.6 * length(suv)) * 1.2;
```
The number multiplied to the end controls the brightness of the final image. We take the negative exponent because length alone will give us the opposite effect. 

Texture Channel 0 for Color which we use to create an effect.
```Shadertoy
vec3 cexp = texture(iChannel0, uv * 0.001).xyz * 3.0 + texture(iChannel0, uv * 0.01).xyz;//vec3(1.0, 2.0, 4.0);
cexp *= 1.4;
```

Final Color Assignment
```Shadertoy
vec3 col = vec3(pow(v, cexp.x), pow(v, cexp.y), pow(v, cexp.z)) * 2.0;

fragColor = vec4(col, 1.0);
```