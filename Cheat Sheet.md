> Note: Use Ctrl+] to indent selected code to the right. Use Ctrl+\[ to indent selected code to the left. 
# Basic Setup
Starting code:
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord;

    // Output to screen
    fragColor = vec4(uv, 0.0, 1.0);
}
```

Normalizing Coordinates: 
```Shadertoy
vec2 uv = fragCoord.xy / iResolution.xy;
```

Shifting origin to center, readjusting range of the axes:
```Shadertoy
uv = (uv - 0.5) * 2.0;
```

Removing stretching by accounting for aspect ratio.
```Shadertoy
uv.x *= iResolution.x / iResolution.y;
```

The final code should look like this:
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
	uv = (uv - 0.5) * 2.0;
	uv.x *= iResolution.x / iResolution.y;
	
    fragColor = vec4(uv, 0.0,1.0);
}
```
# Voronoi Noise Generator
Add `rand2()` before `mainImage()`:
```Shadertoy
vec2 rand2(in vec2 p)
{
	return fract(vec2(sin(p.x * 591.32 + p.y * 154.077), cos(p.x * 391.32 + p.y * 49.077)));
}
```

Add `voronoi()` after `rand2()`:
```Shadertoy
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
# Generating the Pattern
### Basic Pattern 
Generating the Voronoi Pattern:
```Shadertoy
float v = 0.0;
float a = 0.6, f = 1.0;

for(int i=0; i<1; i++){
	float v1 = voronoi(uv * f + 5.0);
	v = v1;
}
```

Setting the output color
```Shadertoy
fragColor = vec4(v, 0.0, 0.0, 1.0);
``` 

Filling the Voronoi Cells with solid color (before assigning to `v`):
```Shadertoy
v1 = smoothstep(0.0, 0.3, v1);
```

Inverting Color (remove before next step)
```Shadertoy
v1 = 1.0 - smoothstep(0.0, 0.3, v1);
```
### Creating the "Electron" Effect
Create an if statement and generate another layer of Voronoi Pattern moving with time:
```Shadertoy
float v2 = 0.0;
if(i > -1)
{
	// of course everything based on voronoi
	v2 = voronoi(uv * f * 0.5 + 50.0 + iTime);
}
```

Create two layers of line patterns in the `if` statement:
```Shadertoy
float va = 0.0, vb = 0.0;
va = 1.0 - smoothstep(0.0, 0.1, v1);
vb = 1.0 - smoothstep(0.0, 0.08, v2);
```

Multiplying `va` into `vb` gives us the basic electricity effect:
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

`for` loop should like the code below:
```Shadertoy
for(int i=0; i<1; i++){
	float v1 = voronoi(uv * f + 5.0);
	float v2 = 0.0;
	if(i > -1)
	{
		// of course everything based on voronoi
		v2 = voronoi(uv * f * 0.5 + 50.0 + iTime);
		
		float va = 0.0, vb = 0.0;
		va = 1.0 - smoothstep(0.0, 0.1, v1);
		vb = 1.0 - smoothstep(0.0, 0.08, v2);
		v += a * pow(va * (0.5 + vb), 2.0);
	}
}

```
### Completing the pattern for one iteration
Add the following function before `mainImage()`:
```Shadertoy
float rand(float n)
{
    return fract(sin(n) * 43758.5453123);
}
```
Add `noise1()` after `rand()`:
```Shadertoy
// 1D noise
float noise1(float p)
{
	float fl = floor(p);
	float fc = fract(p);
	return mix(rand(fl), rand(fl + 1.0), fc);
}
```

Create the variable `flicker` before the loop
```Shadertoy
//Generating the value
float flicker = noise1(iTime * 2.0) * 0.8 + 0.4;
```

Set loop condition to `i<3`
```Shadertoy
for(int i=0; i<3; i++)
```

Set `if` statement to `i>0`:
```Shadertoy
if(i>0)
```

Add the following after the `if` statement:
```Shadertoy
v1 = 1.0 - smoothstep(0.0, 0.3, v1);
```

Add more detail to the pattern with `noise1()`:
```Shadertoy
 v2 = a * (noise1(v1 * 5.5 + 0.1));
```

Apply the glow/flicker to the main layer only:
```Shadertoy
if(i == 0)
	v += v2 * flicker;
else
	v += v2;
```

Change the variables for the next iteration
```Shadertoy
f *= 3.0;
a *= 0.7;
```

Shader after generating the whole effect:
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
	uv = (uv - 0.5) * 2.0;
	uv.x *= iResolution.x / iResolution.y;
	
	float v = 0.0;
	float a = 0.6, f = 1.0;
	float flicker = noise1(iTime * 2.0) * 0.8 + 0.4;
	
	for(int i = 0; i < 3; i ++) 
	{	
		float v1 = voronoi(uv * f + 5.0);
		float v2 = 0.0;
		
		if(i > 0)
		{
			v2 = voronoi(uv * f * 0.5 + 50.0 + iTime);
			
			float va = 0.0, vb = 0.0;
			va = 1.0 - smoothstep(0.0, 0.1, v1);
			vb = 1.0 - smoothstep(0.0, 0.08, v2);
			v += a * pow(va * (0.5 + vb), 2.0);
		}
		
		v1 = 1.0 - smoothstep(0.0, 0.3, v1);
		
		v2 = a * (noise1(v1 * 5.5 + 0.1));
		
		if(i == 0)
			v += v2 * flicker;
		else
			v += v2;
		
		f *= 3.0;
		a *= 0.7;
	}
	
    fragColor = vec4(v, 0.0, 0.0,1.0);
}
```
# Camera Movement
Add the function `rotate()` before `mainImage()`:
```Shadertoy
// rotate position around axis
vec2 rotate(vec2 p, float a)
{
	return vec2(p.x * cos(a) - p.y * sin(a), p.x * sin(a) + p.y * cos(a));
}
```

First, we zoom in and out of the canvas. Add the code before the loop:
```Shadertoy
uv *= 0.6 + sin(iTime * 0.1) * 0.4;
```

Add rotation with angle controlled by `sin()`:
```Shadertoy
uv = rotate(uv, sin(iTime * 0.3) * 1.0);
```

This is a shift of the canvas in the positive XY direction.
```Shadertoy
uv += iTime * 0.4;
```
The number multiplied affects how fast the shift is. 

Shader after adding camera movement: 
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{
    vec2 uv = fragCoord.xy / iResolution.xy;
	uv = (uv - 0.5) * 2.0;
	uv.x *= iResolution.x / iResolution.y;
	
	uv *= 0.6 + sin(iTime * 0.1) * 0.4;
	uv = rotate(uv, sin(iTime * 0.3) * 1.0);
	uv += iTime * 0.4;
	
	float v = 0.0;
	float a = 0.6, f = 1.0;
	float flicker = noise1(iTime * 2.0) * 0.8 + 0.4;
	
	for(int i = 0; i < 3; i ++) 
	{	
		float v1 = voronoi(uv * f + 5.0);
		float v2 = 0.0;
		
		if(i > 0)
		{
			v2 = voronoi(uv * f * 0.5 + 50.0 + iTime);
			
			float va = 0.0, vb = 0.0;
			va = 1.0 - smoothstep(0.0, 0.1, v1);
			vb = 1.0 - smoothstep(0.0, 0.08, v2);
			v += a * pow(va * (0.5 + vb), 2.0);
		}
		
		v1 = 1.0 - smoothstep(0.0, 0.3, v1);
		
		v2 = a * (noise1(v1 * 5.5 + 0.1));
		
		if(i == 0)
			v += v2 * flicker;
		else
			v += v2;
		
		f *= 3.0;
		a *= 0.7;
	}
	
    fragColor = vec4(v, 0.0, 0.0,1.0);
}
```
# Applying Color
Create a copy of  `uv` before applying aspect ratio:
```Shadertoy
vec2 suv = uv;
```

Add the vignette after the loop
```Shadertoy
v *= exp(-0.6 * length(suv)) * 1.2;
```
The number multiplied to the end controls the brightness of the final image. We take the negative exponent because length alone will give us the opposite effect. 

We need to set Texture Channel 0. To do this, scroll down to iChannel0. Select it, go to Textures and choose RGBA noise small. An cog should appear next to iChannel0 ensure that VFlip is not checked. 

Texture Channel 0 for Color which we use to create an effect.
```Shadertoy
vec3 cexp = texture(iChannel0, uv * 0.001).xyz * 3.0 + texture(iChannel0, uv * 0.01).xyz;
cexp *= 1.4;
```

Final Color Assignment
```Shadertoy
vec3 col = vec3(pow(v, cexp.x), pow(v, cexp.y), pow(v, cexp.z)) * 2.0;

fragColor = vec4(col, 1.0);
```

# Completed Shader
The final code for with the color applied is as follows:
```Shadertoy
void mainImage( out vec4 fragColor, in vec2 fragCoord )
{

    vec2 uv = fragCoord.xy / iResolution.xy;
	uv = (uv - 0.5) * 2.0;
	vec2 suv = uv;
	uv.x *= iResolution.x / iResolution.y;
	
	uv *= 0.6 + sin(iTime * 0.1) * 0.4;
	uv = rotate(uv, sin(iTime * 0.3) * 1.0);
	uv += iTime * 0.4;
	
	float v = 0.0;
	float a = 0.6, f = 1.0;
	float flicker = noise1(iTime * 2.0) * 0.8 + 0.4;
	
	for(int i = 0; i < 3; i ++)
	{	
		float v1 = voronoi(uv * f + 5.0);
		float v2 = 0.0;
		
		if(i > 0)
		{
			v2 = voronoi(uv * f * 0.5 + 50.0 + iTime);
			
			float va = 0.0, vb = 0.0;
			va = 1.0 - smoothstep(0.0, 0.1, v1);
			vb = 1.0 - smoothstep(0.0, 0.08, v2);
			v += a * pow(va * (0.5 + vb), 2.0);
		}
		
		v1 = 1.0 - smoothstep(0.0, 0.3, v1);
		
		v2 = a * (noise1(v1 * 5.5 + 0.1));
		
		if(i == 0)
			v += v2 * flicker;
		else
			v += v2;
		
		f *= 3.0;
		a *= 0.7;
	}
	
	v *= exp(-0.6 * length(suv)) * 1.2;
	
	vec3 cexp = texture(iChannel0, uv * 0.001).xyz * 3.0 + texture(iChannel0, uv * 0.01).xyz;
	cexp *= 1.4;
	
	vec3 col = vec3(pow(v, cexp.x), pow(v, cexp.y), pow(v, cexp.z)) * 2.0;
	
	fragColor = vec4(col, 1.0);
}
```
