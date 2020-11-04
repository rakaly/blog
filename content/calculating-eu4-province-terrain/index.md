---
title: "Calculating EU4 Province Terrain"
date: 2020-11-04
banner: eu4-terrain.jpg
shortsummary: "Foundations laid for a user savefile query API, improved the range of game files rakaly supports, and imperator parser"
summary: |
  It's been a few weeks since the last update as my time has been taken up by moving a household to a new city, but I have some tidbits in store:

  - Foundations laid for a user savefile query API
  - Improved the range of game files rakaly supports
  - Imperator parser
---

One of the EU4 achievements left to implement is Eat your Greens where one needs to control all grassland provinces in Asia. While the terrain of a province is an easy concept to grasp, it's surprisingly difficult to calculate. 

The first step is to [start at the wiki](https://eu4.paradoxwikis.com/Map_modding) and work backwards.

The wiki mentions there an automatic algorithm that derives the terrain from `map/terrain.bmp` and this algorithm is overridden on an individual province level in `map/terrain.txt`. Before diving into the automatic algorithm, let's examine the manual override.

## Terrain Overrides

Simplified, `map/terrain.txt` contains categories that manually sets the terrain type of a province. In the below example, provinces with the ID of 1034, 1035, and 1104 have their terrain manually set to glacier.

```plain
categories = {
  # ...
  glacier = {
    # ...
    terrain_override = { 1034 1035 1104 }
  }
}
```

We can count how many provinces have their terrain explicitly set with this js script:

```js
const { readFileSync } = require("fs");
const { Jomini } = require("jomini");

const buffer = readFileSync("map/terrain.txt");
const parser = await Jomini.initialize();
const data = parser.parseText(buffer, { encoding: "windows1252" });

const overrideProvinces = Object.values(data.categories).flatMap(
  (x) => x.terrain_override || []
);

overrideProvinces.sort((a, b) => a - b);
console.log(`${overrideProvinces.length}: ${overrideProvinces}`);
```

There are 2275 entries and if we closely examine the output, we see something quite discerning: duplicates!

So let's find the duplicates:

```js
const dupes = overrideProvinces.filter(
  (x, i, arr) => arr.indexOf(x) !== i
);
console.log(dupes);
```

The above will print:

```js
[
     2,   63,   96,  318,
   798, 2074, 2128, 2129,
  2313, 2386, 2666, 2948,
  2970, 4105, 4176, 4298,
  4307
]
```

I checked a few by hand:

- The province id of 2 (Östergötland) is manually overridden as forest and woods. In the game the province is listed as forest.
- The province id of 63 (Erfurt) is manually overridden as hills and woods. In the game the province is listed as hills.
- The province id of 96 (Zeeland) is manually overridden as grasslands and marsh. In the game the province is listed as grasslands.

There's a pattern: the override that is listed first in the file is given preference.

For the other thousands of provinces not listed in `map/terrain.txt` we turn towards the automatic algorithm

## Automatic Algorithm

The algorithm to automatically derive the terrain for a province from `map/terrain.bmp` is not documented, so I've done my best to reverse engineer it.

We first need to find a province that does not have a terrain explicitly assigned (with 2000+ overrides this may be more difficult than first glance). Since I've been having fun with Oman in a playthrough of mine, I landed on Ibra.

If we overlay the terrain around Ibra with how provinces are defined in `map/provinces.bmp` we see that each province features multiple terrain types. 

{{< sfffig src="terrain.png" caption="Terrain around Ibra" >}}

{{< sfffig src="ibra.png" caption="Provinces around Ibra (reddish orange area)" >}}

It's intuitive that the predominant terrain in a province area would be the chosen terrain. To test this, one can open both images in an image editor and narrow them down to just a single province.

First we gather some info:

- Ibra has a province id of 4287 (you can verify this by enacting `debug_mode` in the console)
- EU4 reports Ibra as having a desert terrain
- Double check that Ibra does not appear in terrain.txt

Then count the pixels from `terrain.bmp` that appear inside Ibra's borders. Your image editor tools will come in handy here. Below are the counts with the RGB hex in parentheses:

- Light brown (CEA963): 236 pixels
- Brown (9E824D): 135 pixels

Ibra is predominately light brown and the next step is to resolve this color to a terrain type. This is done by looking at the BMP file at a very low level. BMP files carry with them a color table where the pixels are indexed into the color table which contains the RGB info. This saves on space as pixels can be stored in a single byte instead of 3 or 4 bytes.

Once we know the color table index, we go back to `map/terrain.txt` and lookup what type has the same color table index. Without getting too into the weeds on how to parse BMP files, the color `9E824D` is set by the 3rd color table index, so we resolve the terrain as desert:

```plain
terrain = {
  desert = {
    type = desert
    color = { 3 }
  }
  # ...
}
```

It is good to verify the theory:

- Bihar (157): is mostly covered by the terrain with a color index of 0 (grasslands) and the game renders Bihar as grasslands
- Aqabah (4268): is mostly covered by the terrain with a color index of 2 (mountains) and the game renders Aqabah as mountains
- Ayaviri (2829): is mostly covered by the terrain with a color index of 6 (mountains) and the game renders Ayaviri as mountains

Excellent so now we know roughly how terrain is automatically assigned -- it's by the most popular terrain

## Edge Cases

Before getting too excited, there are several edge cases that must be accounted for.

There are 3 provinces that currently have a tie for the most populous terrain and the province does not have a terrain override:

- Djerba (2954): has the same amount of coastal desert (19) as coastline (35) and the game represents it as coastal desert.
- Macau (668): has the same amount of grasslands (4) as coastline (35) and the game represents it as grasslands.
- Lima (809): has the same amount of desert mountain (2) as mountain (6) as desert (7) and the game represents it as mountains.

The tie is broken by which terrain has the lower color index (in parentheses).

Then there are indexes that resolve to the same terrain type. For instance, indices 12, 13, 14 all resolve to forest, so it's possible that a province could be dominated by forests but if the forest types are split evenly, a different terrain may appear to be dominant. This occurs surprisingly often with about 20 provinces afflicted (and these provinces don't have an explicit terrain type set). So far, I've found that the combined terrain outweighs the predominant terrain as seen in the below spreadsheet. I've outlined the outlier so far. 

{{< sfffig src="terrain-sheet.png" caption="Combining similar terrain indices to determine terrain" >}}

What's up with Qus? Good question. I'm not sure, but it's the only outlier province. I've even included a breakdown below for Qus:

{{< sfffig src="qus-breakdown.png" caption="Breaking down Qus terrain types" >}}

If I find out why, I'll update this post.

Fun fact: Macau and Lima have their ties also broken by combined terrains, so Djerba is the only province that is proof that ties are broken by terrain that has the lower color index.

## Conclusion

Here are the rules for determining a province's terrain.

- If the province id is listed in `terrain.txt`, choose the terrain that lists the province id first
- Else combine all similar terrains and choose the combined terrain that occurs most often in the province's `terrain.bmp` area, breaking any ties by preferring terrain with the smallest color index 
- Unless your Qus!

So back to the Eat your Greens achievement. The following files are necessary in order to determine if one accomplished it:

- The save file
- `map/continent.txt` to know what provinces are contained in which continents
- `map/definition.csv` to know what color constitute a province
- `map/provinces.bmp` to determine the coordinates of all pixels that are a part of a province
- `map/terrain.txt` for province terrain overrides and the dictionary of color index to terrain type name
- `map/terrain.bmp` to determine province terrain types for those without overrides

Exhausting, but hopefully this has proved informative.