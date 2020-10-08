---
title: "A Tour of PDS Clausewitz Syntax"
date: 2020-10-08
banner: storm.png
shortsummary: "A concise guide to the basic syntax that the Clausewitz parser handles as well as edge cases one will encounter"
summary: "Paradox Development Studio (PDS) develops a game engine called Clausewitz that consumes and produces files in a proprietary format. This format is undocumented. I decided it would be worthwhile for myself as well as future developers interested in writing parsers to not only know the basics of this format but also the edge cases that I've encountered along the way."
---

[Paradox Development Studio (PDS) develops a game engine called Clausewitz](https://en.wikipedia.org/wiki/Paradox_Development_Studio#Game_engines) that consumes and produces files in a proprietary format. This format is undocumented so I decided that it would be a good idea to showcase the happy path, but more importantly, edge cases so that anyone who is interested in writing parsers (myself included) can plan accordingly because there are a lot of parsers: [#1](https://github.com/ParadoxGameConverters/commonItems), [#2](https://github.com/nickbabcock/jomini), [#3](https://github.com/cwtools/cwtools), [#4](https://github.com/rakaly/jomini), [#5](https://github.com/Shadark/ClauseWizard), [#6](https://github.com/nickbabcock/Pfarah), [#7](https://github.com/primislas/oikoumene), [#8](https://github.com/nickbabcock/Pdoxcl2Sharp), [#9](https://github.com/fuchsi/clausewitz_parser), [#10](https://github.com/soryy708/ClauParse), [#11](https://github.com/rikbrown/klausewitz-parser), [#12](https://github.com/ClauText/ClauParser), [#13](https://gitgud.io/nixx/paperman), [#14](https://github.com/cormac-obrien/pdx-txt), [#15](https://github.com/cloudwu/pdxparser), [#16](https://github.com/scorpdx/ck3json). And these are the only ones I've found after a quick open source search!

So if anyone wants to write a parser for Europa Universalis IV, Crusader Kings III, Stellaris, Hearts of Iron IV, Imperator -- this should be a good starting point. Before getting started with the tour, I see some try and describe the format formally with [EBNF](https://en.wikipedia.org/wiki/Extended_Backus%E2%80%93Naur_form) and while this may be possible, reality is a bit more messy. The data format is undocumented and any parser will need to be flexible enough to ingest whatever the engine produces or can also ingest. 

To keep the scope of this post limited. We'll only cover the plain text format. The binary format used predominantly for save files will be for another time. How to write scripted game files encoded in the Clausewitz format won't be covered as this layer above Clausewitz is called Jomini and has [at least some documentation](https://forum.paradoxplaza.com/forum/threads/grand-jomini-modding-information-manuscript.1170261). As an aside, it's a bit frustrating or just unfortunate to be [an author of a Clausewitz parser also called Jomini](https://github.com/nickbabcock/jomini) that predates Paradox's implementation by several years. I guess naming is hard or great minds think alike!

Two things to keep in mind before we begin:

- This is a knowledge dump from someone who has no relationship with Paradox or the engine, so mistakes are possible
- This tour will focus on creating a superset of syntax pulled from several games. When producing data, make sure it's compatible with the intended game. Our goal is to be robust: liberal in what we consider valid and conservative in what we produce.

## The Tour

The simplest of examples can use [TOML](https://github.com/toml-lang/toml) syntax highlighting

```toml
# This is a line comment
cid = 1 # This is an inline comment
name = "Rakaly Rulz"
```

The above depicts a nice 1-to-1 key-value mapping that any language worth its salt can store in an ergonomic data structure.

But before we turn off syntax highlighting, let's visit the first edge case: duplicate and unordered keys

```toml
# This is a line comment
cid = 1 # This is an inline comment
name = "Rakaly Rulz"
cid = 2
```

In this case, `cid` wouldn't map to a singular value but instead to a list of values. This format is commonly seen in EU4 saves

But that's about as far as we can take syntax highlighting so future examples will be plain.

## Scalars

A value in a key-value pair that contains the smallest unit of measurement is called a scalar. Shown below is an example demonstrating a smattering of scalars.

```plain
aaa=foo         # a plain scalar
bbb=-1          # an integer scalar
ccc=1.000       # a decimal scalar
ddd=yes         # a true scalar
eee=no          # a false scalar
fff="foo"       # a quoted scalar
ggg=1821.1.1    # a date scalar in Y.M.D format
hhh="a\"b"      # a quoted scalar with an escaped quote
```

Some notes:

- A quoted scalar can contain any of other scalar (except for another quoted scalar)
- A quoted scalar can contain any character including newlines. Everything until the next unescaped quote is valid
- A quoted scalar can contain non-ascii characters like "Jåhkåmåhkke". The encoding for quoted scalars will be either Windows-1252 (games like EU4) or UTF-8 (games like CK3)
- Decimal scalars vary in precision between games and context. Sometimes precision is recorded to thousandths, tens-thousandths, etc.
- Dates do not incorporate leap years, so don't try sticking it in your language's date representation.
- One should delay assigning a type to a scalar as it may be ambiguous if `yes` should be treated as a string or a boolean. This is more of a problem for the binary format as dates are encoded as integers so eagerly assigning a type could mean that the client sees dates when they expected integers.

Keys are scalars:

```plain
-1=aaa
"1821.1.1"=bbb
@my_var="ccc"    # define a variable
```

One can have multiple key values pairs per line as long as **boundary** character is separating them:

```plain
a=1 b=2 c=3
```

Whitespace is considered a boundary, but we'll see more.

## Arrays / Objects

Arrays and objects are values that contain either multiple values or multiple key-value pairs.

Below, `flags` is an object.

```plain
flags={
    schools_initiated=1444.11.11
    mol_polish_march=1444.12.4
}
```

And an array looks quite similar:

```plain
players_countries={
    "Player"
    "ENG"
}
```

And one can have arrays of objects

```plain
campaign_stats={ {
    id=0
    comparison=1
    key="game_country"
    selector="ENG"
    localization="England"
} {
    id=1
    comparison=2
    key="longest_reign"
    localization="Henry VI"
} }
```

## Operators

There are more operators than equality separating keys from values: 

```plain
intrigue >= high_skill_rating
age > 16
count < 2
scope:attacker.primary_title.tier <= tier_county
```

The other operators are typically reserved for game files (save files only use equals).

## Boundary Characters

Mentioned earlier, what separates values are boundary characters. Boundary characters are:

- Whitespace
- Open (`{`) and close (`{`) braces
- An operator

Thus, one can make some pretty condensed documents.

```plain
a={b=1 c=d}foo=bar
```

And though I don't have confirmation, I believe there are a couple more boundary characters:

- Quotes
- A Comment

So the below document could be possible:

```plain
key=value#comment
list={"a""b""c"} 
```

## The Weeds

An object / array value does not need to be prefixed with an operator:

```plain
foo{bar=qux}

# is equivalent to `foo={bar=qux}`
```

A value of `{}` could mean an empty array or empty object depending on the context. I like to leave it up to the caller to decide.

```plain
discovered_by={} 
```

Any number of empty objects / arrays can occur in an object and should be skipped.

```plain
history={{} {} 1629.11.10={core=AAA}}
```

An object can be both an array and an object at the same time:

```plain
brittany_area = { #5
    color = { 118  99  151 }
    169 170 171 172 4384
}
```

Scalars can have non-alphanumeric characters:

```plain
flavor_tur.8=yes
dashed-identifier=yes
province_id = event_target:agenda_province
@planet_standard_scale = 11
```

A scalar has at least one character:

```plain
# `=` is the key and `bar` is the value
=="bar"
```

*Unless* the empty string is quoted:

```plain
name=""
```

The type of an object or array can be externally tagged:

```plain
color = rgb { 100 200 150 }
color = hsv { 0.43 0.86 0.61 }
color = hsv360{ 25 75 63 }
color = hex { aabbccdd }
mild_winter = LIST { 3700 3701 }
```

An object can have a leading scalar (or my preference is looking at it as an array where the last element is an object):

```plain
levels={ 10 0=2 1=2 }
# I view it as equivalent to
# levels={ { 10 } { 0=2 1=2 } }
```

Objects can be infinitely nested. I've seen modded EU4 saves contain recursive events that reach several hundred deep.

```plain
a={b={c={a={b={c=1}}}}}
```

The first line of save files indicate the format of the save and shouldn't be considered part of the standard syntax.

```plain
EU4txt
date=1444.12.4
```

While I haven't encountered this edge case, there's been [a report](https://forum.paradoxplaza.com/forum/threads/all-save-files-have-an-unsightly-erroneous-trailing.706077/) about games generating save files with unbalanced braces. I'm crossing my fingers that all instances of this have been eradicated since the report 7 years ago. 

Save files can reach 100 MB in size and reach over 7 million lines long, so any parser must have performance as a focus. 

## Conclusion

With all these edge cases, a parser needs to be flexible. There are many ways to parse data, and I won't say which one is correct. In the list of open source parsers there's a good mix of regular expressions, parser generators, pull parsers, push parsers, dom parsers, and my favorite: [tape parsers](https://github.com/simdjson/simdjson/blob/4d2736ffa91c5ff072d1ab93241ee399892707d4/doc/tape.md). Which is the best approach may come down to the specific situation.

Good luck!
