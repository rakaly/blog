---
title: "Tracking Province Building Construction Dates"
date: 2020-08-24
banner: racing-province-values.png
shortsummary: "New racing bar chart for construction dates for province buildings"
summary: "Racing bar charts are seemingly all the rage now, so of course I must capitalize on this opportunity. I implemented a racing bar chart that tracks how popular a province building is over time. Hopefully one can glean useful insights from it, I know that I received confirmation that the AI is overly zealous when it comes to building coastal defences"
---

Back with some more graphs! [Racing bar charts](https://app.flourish.studio/@flourish/bar-chart-race) are seemingly all the rage now, so of course I must capitalize on this opportunity. I implemented a racing bar chart that tracks how popular a province building is over time. See the gif below for one of my saves!

{{< sfffig src="building-history.gif" caption="A racing bar chart of construction dates for province buildings. Click the gif" >}}

Hopefully one can glean useful insights from it, I know that I received confirmation of a certain AI bias shown below.

{{< sfffig src="coastal.png" caption="Building count in a passive game in the early 1700's. Coastal defense seems too heavily weighted" >}}

I feel AI is overly zealous when it comes to building coastal defences. The fact that there are nearly as many coastal defense buildings as marketplaces in a game where I'm playing a somewhat passive Switzerlake game is striking. Perhaps the AI doesn't weigh trade power as strongly as it should.

## Future

Some future improvements:

- Track building deletion (eg: when a player conquers a province and deletes all the coastal defenses, the count should go down)
- Provide an alternate view of the annual ledger details (monthly income, nation size, etc) so that one can visualize how nation size and income change over time. This should make those graphs more meaningful when it comes to world conquests (as it would be nice to see top rivals taken down visually)