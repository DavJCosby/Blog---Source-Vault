If you'd like what I do, you can support me by following me on [Twitter](https://twitter.com/cosbdev) or use the link below to buy me a boba.

<a href="https://www.buymeacoffee.com/davidcosby"><img src="https://img.buymeacoffee.com/button-api/?text=Buy me a boba&emoji=🧋&slug=davidcosby&button_colour=d699b6&font_colour=293136&font_family=Lato&outline_colour=293136&coffee_colour=d3c6aa" style="max-width: 16em"/></a>

# Miscellaneous Tech
```dataview
TABLE WITHOUT ID file.link as "Posts", file.mday as "Date", max(round(file.size / 7500)*5, 2)+" minutes" as "Reading Time"
FROM #misc
```
# Devlogs
```dataview
TABLE WITHOUT ID file.link as "Posts", file.mday as "Date", max(round(file.size / 7500)*5, 2)+" minutes" as "Reading Time"
FROM #devlog
```
