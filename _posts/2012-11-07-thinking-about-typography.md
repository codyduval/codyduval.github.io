---
layout:     post
title:      "Thinking About Typography"
date:       2012-11-07 12:31:19
summary:    "The magical relationship between font size, weight, and line height."
categories: design
---
I just stumbled onto Mark Boulton's [five-part blog series](http://www.markboulton.co.uk/journal/five-simple-steps-to-better-typography) on typography. I came across his blog while Googling for information on the ideal relationship between a font's size and line height. More specifically, I was trying to figure out what CSS font settings are most appropriate for a blog where reading and legibility is paramount.

###Font Size
Interestingly, I learned that a six-point to 72-point font scale (and the sizes in-between) has been the standard for over four-hundred years. And until relatively recently, each size was referred to by name:

<p style="font-size: 6px">
6pt: nonpareil
</p>

<p style="font-size: 7px">
7pt: minion
</p>

<p style="font-size: 8px">
8pt: brevier or small text
</p>

<p style="font-size: 9px">
9pt: bourgeois or galliard
</p>

<p style="font-size: 10px">
10pt: long primer or garamond
</p>

<p style="font-size: 11px">
11pt: small pica or philosophy
</p>

<p style="font-size: 12px">
12pt: pica
</p>

<p style="font-size: 14px">
14pt: english or augustin
</p>

<p style="font-size: 18px">
18pt: great primer
</p>

<p style="font-size: 21px">
21pt: double small pica or double pica
</p>

<p style="font-size: 24px">
24pt: double pica or two-line pica
</p>

<p style="font-size: 36px">
36pt: double great primer
</p>

Point being, this family of sizes was chosen for its visual harmony - it's not an accident we see this list in nearly every web application since the dawn of word processors.

###Line Height
For body copy, there seems to be consensus that line height should be 1.5 times font size. So a 15px font should have a line height of 22.5px. But this is really only a starting point, because line height is directly related to...

###Line Length
Over at [The Ultimate Guide to Readable Web Typography](http://www.pearsonified.com/2011/12/golden-ratio-typography.php), the author argues that not only are the vertical values of line height and font size important, but the third dimension of line width must also be considered. For any font size, the line height must increase as the length of the line gets longer.

You can check out his math on [his blog](http://www.pearsonified.com/2011/12/golden-ratio-typography.php); in essence, he uses the Golden Ratio (1.61) as the basis for his equation.
