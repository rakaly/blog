---
title: "Thoughts on Converting PDS Clausewitz Syntax to JSON"
date: 2021-03-11
banner: storm.png
shortsummary: "A concise guide to the basic syntax that the Clausewitz parser handles as well as edge cases one will encounter"
summary: "Paradox Development Studio (PDS) develops a game engine called Clausewitz that consumes and produces files in a proprietary format. This format is undocumented. I decided it would be worthwhile for myself as well as future developers interested in writing parsers to not only know the basics of this format but also the edge cases that I've encountered along the way."
---

After one is familiar with the [PDS Clausewitz syntax]({{< relref a-tour-of-pds-clausewitz-syntax.md >}}), it becomes almost too tempting to try and convert it to JSON. There are a lot of nuances though so I figured I'd take the time and do a brain dump of my thoughts.

## The Easy Case

Let's start with the easy case -- the document below.

```plain
foo=bar
mynum=10
mybool=false
myobj={
  prop=erty
}
myarr={1 2}
```

And transform it into JSON

```json
{
  "foo": "bar",
  "mynum": 10.0,
  "mybool": false,
  "myobj": {
    "prop": "erty"
  },
  "myarr": [1.0, 2.0]
}
```

Text values extracted should not be written blindly to JSON, as [JSON has a default encoding of UTF-8](https://tools.ietf.org/html/rfc7158#section-8.1)