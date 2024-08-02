# Seeds of Andromeda

This is an archived and adapted version of the seeds of andromeda aricle series called "Fast Flood Fill Lighting in a Blocky Voxel Game"

Retrieved from:
- [Part 1 - Breadth-First-Search](https://web.archive.org/web/20210622035752/https://www.seedofandromeda.com/blogs/29-fast-flood-fill-lighting-in-a-blocky-voxel-game-pt-1)
- [Part 2 - RGB & Sunlight](https://web.archive.org/web/20210416235853/https://www.seedofandromeda.com/blogs/30-fast-flood-fill-lighting-in-a-blocky-voxel-game-pt-2)

# Lighting

![ColouredCave](/voxel/lighting/soa/ColoredCave.jpg)

So you have a basic blocky voxel engine, and you want to implement lighting. The easy route would be to use standard deferred dynamic lighting like in most other games. However, dynamic lighting has a few issues.

1. You need to shadow map every light to prevent light from bleeding through your voxels.

2. Light will not radiate around corners without expensive global illumination, which can be a bit unrealistic in small undergound tunnels with many winding passages.

3. The more lights in the scene you have, the slower the game will run. Though this is greatly alleviated with deferred rendering.

Now these issues might not really be that relevant to your game, but they are in Seed Of Andromeda. Luckily, there is an alternative! We can store light as voxel data, and propagate it through the grid. This allows light to radiate around corners, and since the lighting is baked into the mesh, increasing the number of lights will minimally affect performance. We can also easily find the light level at a given point by simply sampling the voxel grid, which can be useful if we want voxel plants to be able to grow when exposed to sunlight. However, there are some issues with voxel lighting as well.

1. Voxel lighting requires you to manually propagate the light. I will show you how to do it quickly, but it is still more expensive than simply placing a dynamic light. This calculation only needs to be done when adding or removing a light, or when modifying blocks around a light.

2. The maximum range of your voxel lighting is limited by how much voxel data you allocate for it. If you allocate 5 bits for it, then the maximum range of any light is 25=32 blocks from the source.

3. The shape of the resulting light volume is not perfectly spherical, it is actually the shape of an octahedron.

If the cons of voxel lighting outweight the cons of dynamic lighting, you might want to go with dynamic lighting, or maybe even a combination of the two. Otherwise, I will show you the workings of voxel lighting.

## Constraints

For this tutorial I will be assuming a few things.
1. Your world is made up of cubic chunks of size 32^3. All voxel data is stored in flat arrays per chunk. If this is not the case, modify accordingly.
2. You want sunlight to propagate downward from the top of your voxel world, until it hits something.
3. You want lamps/torches that emit voxel light that is independent of the sunlight.

Any example code will be provided in C++.

## The Data

In order to correctly propagate and update the light, we need to store the light levels of each block.
To do that we will use a 3d lightmap, which is just a 3d array. Now the maximum light level
will depend entirely on your engine. Minecraft uses light levels of 0-15. I use 0-31 since my voxels are half
as large. So for now I will assume you will be using levels 0-15, which can be stored in 4 bits.

Since we need to store both sunlight and torchlight seperately, we will need 2 lightmaps. However, since
sunlight and torchlight only need 4 bits per voxel each, we can combine them into a single 8 bit byte.
Therefore we only need a single lightMap of bytes. (This is not true if we want to use colored light. See the colored light section in part 2 for more details)

```cpp
unsigned char lightMap[chunkWidth][chunkWidth][chunkWidth]; //chunkWidth == 32
```

Getting and setting light and sunlight are done using bitwise operations, since we need to mask out 4 bits
of the byte in each case. Lets say that the most significant 4 bits are sunlight, and the least significant
are voxel light. We can get and set the light values using these fast inline functions. Remember, 0xF is 15,
which is 00001111 in binary. 0xF0 is 11110000.

```cpp
// Get the bits XXXX0000
inline int Chunk::getSunlight(int x, int y, int z) {
    return (lightMap[y][z][x] >> 4) & 0xF;
}

// Set the bits XXXX0000
inline void Chunk::setSunlight(int x, int y, int z, int val) {
    lightMap[y][z][x] = (lightMap[x][y][z] & 0xF) | (val << 4);
}

// Get the bits 0000XXXX
inline int Chunk::getTorchlight(int x, int y, int z) {
    return lightMap[y][z][x] & 0xF;
}

// Set the bits 0000XXXX
inline void Chunk::setTorchlight(int x, int y, int z, int val) {
    lightMap[y][z][x] = (lightMap[x][y][z] & 0xF0) | val;
}
```

Notice that I am calling lightMap[y][z][x]. This is a matter of personal preference, you can store
the data in whatever order you wish. This is the cache friendly way to do it if you want to iterate your
chunk like this

```cpp
for (int y = 0; y < chunkWidth; y++) {
    for (int z = 0; z < chunkWidth; z++) {
        for (int x = 0; x < chunkWidth; x++) {
            //...
```

NOTE: If you would like to, you could just store two separate lightmaps. You will double your RAM usage,
but you could use the extra 4 bits for something else. If you want to also store light color, you could
use more data and store color channels.

When we first initialize our chunks, we should be sure to zero out the lightmap. In c++ this can be done
quite quickly with.

```cpp
memset(lightMap, 0, sizeof(lightMap));
```

## The Propagation Algorithm

Ok we have our storage! Now what? Well lets start with torchlight. If the player places a torch on a block,
we might call chunk.setSunlight(x, y, z, 14). But now we need to spread the light outwards in each direction,
reducing the light level by 1 at each block until we are at zero. Check out the image below, that I shamelessly stole from minecraft.gamepedia.

![TorchlightDistance](/voxel/lighting/soa/TorchlightDistance.jpg)

In this image, a yellow T box is a torch, and each torch has a light level of 14. As you can see, light level decreases by 1 for each cell that you move away from a torch. Notice that along the green diagonal, light level decreases by two instead of one. This is because distance from the light is calculated as X distance + Y distance, instead of a true linear distance calculation. Keep in mind this is happening in 3D, which the image does not illustrate.

So how do we design an algorithm that does this? Well a naive solution that I see people coming up with is
that you do 15 light passes (one for each level) to spread the light. Each pass, you iterate every block
in the chunk and when you hit a block with light you spread it to a neighbouring spot. After 15 passes
you are guaranteed to have spread out the light correctly! Sounds good right? NO!

That is WAY to much iteration. We only want to visit each voxel that the light touches ONE time! And we dont
want to visit any voxels that are unaffected! For this, we need Breadth-First Search (BFS).

The BFS algorithm is perfect for this job. And it is incredibly simple to implement! All we need is a FIFO queue
of nodes. For instance:

```cpp
std::queue <LightNode> lightBfsQueue.
```

Lets define a LightNode as follows.  

```cpp
struct LightNode {
    LightNode(short indx, Chunk* ch) : index(indx), chunk(ch) {}
    short index; //this is the x y z coordinate!
    Chunk* chunk; //pointer to the chunk that owns it!
}
```

<small>Constructor in a struct? I'm so bad...</small>

Notice that instead of storing x, y, and z, we store a single value index. This is to keep our node class
super tiny. As pointed out by /u/mysticreddit, this is called "linearizing". The smaller it is, the more
cache friendly it is, and the better our algorithm will perform.
To get an index from x, y, and z you do the following. ( Remember, I store them in (y,z,x)  order )

## The Algorithm

So lets look at the BFS algorithm for our lighting.

When placing a torch with light level L we do the following:

***Step 1:** Call setTorchlight to set the light.*

```cpp
chunk->setTorchlight(x, y, z, L);
```

***Step 2:** Add a new node for the torch to the bfsQueue.*

```cpp
short index = y * chunkWidth * chunkWidth + z * chunkWidth + x;
lightBfsQueue.emplace(index, chunk)
```

***Step 3:** Repeat the following steps until bfsQueue is empty.*

```cpp
while(lightBfsQueue.empty() == false) {
```

***Step 4:** Take the front node off of the queue.*

```cpp
    // Get a reference to the front node.
    LightNode &node = lightBfsQueue.front();

    int index = node.index;
    Chunk* chunk = node.chunk;

    // Pop the front node off the queue. We no longer need the node reference
    lightBfsQueue.pop();

    // Extract x, y, and z from our chunk.
    // Depending on how you access data in your chunk, this may be optional
    // could also be, x = index & (chunkWidth-1);
    int x = index % chunkWidth; 

    // could also be, y = (index >> 10) & (chunkWidth-1);
    int y = index / (chunkWidth * chunkWidth); 

    // could also be, z = (index >> 15) & (chunkWidth-1);
    int z = (index % (chunkWidth * chunkWidth) ) / chunkWidth;

    // Grab the light level of the current node       
    int lightLevel = chunk->getTorchlight(x, y, z);
```

***Step 5:** Look at all neighbouring voxels to that node. if their light level is 2 or more levels less than the current node, then set their light level to the current nodes light level - 1, and then add them to the queue.*

```cpp
    // NOTE: You will need to do bounds checking!
    // If you are on the edge of a chunk, then x - 1 will be -1. Instead
    // you need to look at your left neighboring chunk and check the
    // adjacent block there. When you do that, be sure to use the
    // neighbor chunk when emplacing the new node to lightBfsQueue;

    // Check negative X neighbor
    // Make sure you don't propagate light into opaque blocks like stone!
    if (chunk->getBlock(x - 1, y, z).opaque == false &&
            chunk->getTorchlight(x - 1, y, z) + 2 <= lightLevel) {

        // Set its light level
        chunk->setTorchlight(x - 1, y, z, lightLevel - 1);   

        // Construct index
        short index = y * chunkWidth * chunkWidth + z * chunkWidth + (x - 1);

        // Emplace new node to queue. (could use push as well)
        lightBfsQueue.emplace(index, chunk);
    }
    // Check other five neighbors
    ...
}  //End while loop
```

The end! Its only 5 steps, and the actual loop is only 2 steps! Not that bad at all!

## Light Removal

Well we have light propagation figured out, but what happens when we want to delete a light? We simple do BFS again! This just requires modification to the spreading algorithm.

We need a new node definition, and a new queue to prevent duplicate node updates with this algorithm

```cpp
std::queue <LightRemovalNode> lightRemovalBfsQueue.
struct LightRemovalNode {
    LightNode(short indx, short v, Chunk* ch) : index(indx), val(v), chunk(ch) {}
    short index; //this is the x y z coordinate!
    short val;
    Chunk* chunk; //pointer to the chunk that owns it!
}
```

***Step 1:** Add a new node for the torch to the bfsQueue, and set light to zero*

```cpp
short val = (short)chunk->getTorchlight(x, y, z);
short index = y * chunkWidth * chunkWidth + z * chunkWidth + x;
lightRemovalBfsQueue.emplace(index, val, chunk);
chunk->setTorchlight(x, y, z, 0);
```

***Step 2:** Repeat the following steps until bfsQueue is empty*

```cpp
while(lightRemovalBfsQueue.empty() == false) {
```

***Step 3:** Take the front node off of the queue*

```cpp
    // Get a reference to the front node
    LightRemovalNode &node = lightRemovalBfsQueue.front();
    int index = (int)node.index;
    int lightLevel = (int)node.val;
    Chunk* chunk = node.chunk;

    // Pop the front node off the queue.
    lightRemovalBfsQueue.pop();

    // Extract x, y, and z from our chunk. Same as before.
    ...
```

***Step 4:** Look at all neighbouring voxels to that node. if their light level is nonzero and is less than the current node, then add them to the queue and set their light level to zero. Else if it is >= current node, then add it to the light propagation queue.*


```cpp
    // NOTE: Don't forget chunk bounds checking! I didn't show it here.
    // Check negative X neighbor
    int neighborLevel = currentNode.chunk->getTorchlight(x - 1, y, z);

    if (neighborLevel != 0 && neighborLevel < lightLevel) {
        // Set its light level
        currentNode.chunk->setTorchlight(x - 1, y, z, 0);   

        // Construct index
        short index = y * chunkWidth * chunkWidth + z * chunkWidth + (x - 1);

        // Emplace new node to queue. (could use push as well)
        lightRemovalBfsQueue.emplace(index, neighborLevel, chunk);
    } else if (neighborLevel >= lightLevel) {
        short index = y * chunkWidth * chunkWidth + z * chunkWidth + (x - 1);

        // Add it to the update queue, so it can propagate to fill in the gaps
        // left behind by this removal. We should update the lightBfsQueue after
        // the lightRemovalBfsQueue is empty.
        lightBfsQueue.emplace(index, chunk);
    }

    // Check other five neighbors
    ...      
}  //End while loop
```

Notice that in step 4 we added an else if that checks to see if the neighbor light is >= the current nodes light. If it is, then we will propagate this light witht he first BFS algorithm, by pushing it onto lightBfsQueue and updating it after we are done with all the light removal updates. This is so that if you have two lights next to each other, removing one light will allow the other light to fill in the empty spaces left behind.

# Sunlight

![Sunlight](/voxel/lighting/soa/Sunlight.jpg)

We have learned the basics of voxel light propagation, but we have not yet covered the special case of sunlight. Assuming that you are using cubic chunks, how do we determine if a block is in sunlight or not? How do we deal with sunlight coming from unloaded chunks?

## Propogation

First lets worry about propagating the sunlight. Unlike torchlight, sunlight rays should propagate downward from the top of our voxel world. We should propagate sunlight even if it is night time (I will cover how to do Day/Night cycles below). All air blocks that have a direct line of sight to the sky should have the maximum sunlight level (16). Because we don't want sunlight to conflict with torchlight, sunlight should be stored separately. Luckily, we showed how to do this in part 1 using bitwise operations.

Like with torchlight, we will also need a BFS queue for calculating the basic propagation.

```cpp
std::queue <LightNode> sunlightBfsQueue.
```

After generating a chunk, we need to calculating sunlight. The sunlight generation algorithm for a single chunk goes as follows:

- **1** Check if the chunk above this chunk is loaded. Call this chunk TOP.
    - **1.1** If TOP is loaded, check the sunlightMap for TOP. For all nonzero sunlight values, add a node to the sunlightBfsQueue.
    - **1.2** If TOP is not loaded, then we need to guess if this chunk is above ground. This process varies based on your architecture. If your terrain is generated from a heightmap, then you can simply check the position of the chunk against the heightmap. If the chunk's location is at or above the heightmap, then we can assume there is sunlight above the chunk. Otherwise, we assume there is no sunlight.
        - **1.2.1** If we assume that there is no sunlight, i.e. we are underground, then we are done.
        - **1.2.2** If we assume that there is sunlight, then we iterate the voxels at the very top of the chunk.
        - **1.2.3** If a voxel is transparent to light, such as glass or air, then we set the sunlight value to the maximum and add a node to sunlightBfsQueue.

After generating a chunk, we may have a list of nodes in our sunklightBfsQueue. We should now do the standard BFS light propagation algorithm as highlighted in part 1, but with an additional constraint. Whenever we are propagating light downward in the -Y direction, we do NOT lower the light level by 1 if the light is currently at the max level (16). This simple constraint will cause light to propagate downard forever until it hits an opaque voxel, but of course it will still propagate sideways normally. I am not going to paste the example code here because it is very redundant. You should be able to figure out how to add the step to the pseudocode in part 1.

## Removal

When we place a block in the path of a sunlight ray, we should block the sun ray and darken all voxels below it. Whenever the user places an opaque block in a voxel that has nonzero sunlight, we should remove the sunlight at that voxel. To do this, we simply use the same light removal algorithm as torchlight, but again with an additional constraint.

When the light level of the current node is at the maximum (16), then when checking the voxel below the current node we ALWAYS remove it, even if its light level is also at the maximum. This simple rule will cause an entire ray of sunlight to be removed if an opaque block is placed above it.

## Day/Night Cycle

So we know how to set the light level, but how do we handle day/night transitions? This is also really easy! We first need to determine the brightness of sunlight based on the time of day. This can be done simply by checking how high the sun is in the sky. You can use any formula you want, for instance, if we have a vector called sunPosition which is the normalized vector to the sun, we could use:

```cpp
float sunlightIntensity = max(sunPosition.y * 0.96f + 0.6f, 0.02f);
```

Next, we simply use this sunlightIntensity value when calculating the brightness based on the voxel light level. This should be done in your shader program. For instance:

```cpp
float lightIntensity = torchlight + sunlight * sunlightIntensity;
color = fragmentColor * lightIntensity;
```

And voila! We have a proper day/night cycle!

# Colored Light

![RGB](/voxel/lighting/soa/RGB.jpg)

We know how to propagate torchlight of a single color, but what if we want users to be able to place lights with many different colors? We need to make sure these colors blend together correctly, since the player should be allowed to mix all of our colors. To do this, we are going to need to store more data in the torchlight.

Recall that we store torchlight in an array:

```cpp
unsigned char lightMap[chunkWidth][chunkWidth][chunkWidth];
// Bits = SSSS TTTT
```

Where 4 bits are used for torchlight and the other 4 bits are sunlight. To do colored light, we could expand this lightMap so that each voxel gets 2 bytes (16 bits). Then we can store an RGB color channel in this data. 4 bits for red, 4 bits for green, 4 bits for blue, and 4 bits sunlight. Our new lightmap would look like this:

```cpp
unsigned short lightMap[chunkWidth][chunkWidth][chunkWidth];
// Bits = SSSS RRRR GGGG BBBB
```

We will need helper functions like in part 1 that allow us to get and set the different color channels.

```cpp
inline int Chunk::getRedLight(int x, int y, int z) {
    return (lightMap[y][z][x] >> 8) & 0xF;
}

inline void Chunk::setRedLight(int x, int y, int z, int val) {
    lightMap[y][z][x] = (lightMap[x][y][z] & 0xF0FF) | (val << 8);
}

inline int Chunk::getGreenLight(int x, int y, int z) {
    return (lightMap[y][z][x] >> 4) & 0xF;
}

inline void Chunk::setGreenLight(int x, int y, int z, int val) {
    lightMap[y][z][x] = (lightMap[x][y][z] & 0xFF0F) | (val << 4);
}

inline int Chunk::getBlueLight(int x, int y, int z) {
    return lightMap[y][z][x] & 0xF;
}

inline void Chunk::setBlueLight(int x, int y, int z, int val) {
    lightMap[y][z][x] = (lightMap[x][y][z] & 0xFFF0) | (val);
}
```

## Propagating and Removing Colored Light

Propagating and removing the colored light is as simple as can be. All we need to do is propagate all three channels separately using the same algorithms as in part 1! So instead of doing a single BFS, we do three BFS passes, one for each color channel. This will cause different colored lights to automatically mix together correctly!

# A Word on Attenuation

Attenuation is the gradual loss of color as we get further from the light. Unfortunatly, this light color method is not perfect. It sufferes from an attenuation inaccuracy. Since each color channel attenuates independantly from the other, and each decreases at a linear rate of I - 1 per voxel traversed, we will get some strange behavior.

When all nonzero light color channels are equal, such as a magenta torch where (R,G,B) = (16,0,16) we get perfect attenuation. However, if we use a light color where the ratio of nonzero colors is not 1:1 it sort of breaks down. Take orange light for instance, (R,G,B) = (16,8,0). Since the ratio of red to green light is 2:1 we are going to get improper attenuation.

In reality, as a color attenuates the ratio of the color channels should remain equal as the brightness approaches zero. Unfortunatly, since each channel decreases by 1 at each voxel this is not true with our model. Lets pretend we are 8 voxels away from the original orange light source. At this point, our light color is (8,0,0). The green channel is completely gone! This is no longer orange light at all, it is now dark red light.

![Attenuate](/voxel/lighting/soa/Attenuate.jpg)

How big of an issue is this? Well it depends. The attenuation problem only occurs when the ratio of nonzero colors is not 1:1. If we constrain all the torches in our game to a select few colors this issue will never appear. Or, we could allow colors with non-zero ratio anyways as it isn't really that noticable in most cases. If we want to stick to only 1:1 colors, we can use the following 7 colors:

```
    White (16,16,16)
    Cyan (0,16,16)
    Magenta (16,0,16)
    Yellow (16,16,0)
    Red (16,0,0)
    Green (0,16,0)
    Blue (0,0,16)
```

# Color Filters

![Filter](/voxel/lighting/soa/Filter.jpg)

Because we are storing the RGB color channels, we can also do color filtering! For instance, if we have red stained glass and white light passes through it, it should become red light! Doing this is fairly easy, we just give red stained glass a light filter color value of (R,G,B) = (1.0f,0.0f,0.0f). We also need to add a step in light propagation that checks if the current block has a color filter value. If it does, we simply multiply the color by the color filter. For instance, if we have white light and want to pass it through a red color filter, we will end up with:

```
newColor = (16,16,16) * (1.0f,0.0f,0.0f) = (16,0,0)
```

# Gamma Correction

I am not going to go too into detail about gamma correction since there is an excellent [GPU Gems](https://web.archive.org/web/20210416235853/http://http.developer.nvidia.com/GPUGems3/gpugems3_ch24.html) post about it. But we definitely want to utilize gamma correction to make sure we correct the nonlinearity of our display. Adding some simple gamma correction to your shaders is easy:

```cpp
float gamma = 1.0 / 2.2; // This could be changeable in the options
color = pow(color, gamma);
```