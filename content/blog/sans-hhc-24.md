---
title: "SANS Holiday Hacks '24 Write-Up"
date: 2024-12-21T21:06:30-06:00
draft: false
author: Jared Baron Panares
tags:
    - WriteUp
image: /images/sans-hhc-image.png
description:
toc:
center: true
---

## Elf Connect (Hard):
#### Hint Summary: The high score is 50k, but that's above the maximum amount of points. Try looking into the browser dev tools and for a variable named score.

From the hint, we assume that we start at the source of the page. When we look into the HTML for the game pop-up, we can find a script tag with some JavaScript. Based on the hint, we should try looking for the variable **score**.

Now that we've spotted score, we should check if it can be accessed via the console in our dev tools. Make sure to switch to switch to the game window in the dropdown within the console.

Yes, we can access it! When we look back at the JavaScript, we see a mission completion for a score > 50000. So we should set score to an arbitrary value > 50000. Lets say score = 60000. Ok now we see that for the mission to complete, we must also have the value 

```
isCorrectSet = true;
```

isCorrectSet is a local variable set to false, so we can't acccess it directly from the console. Let's check for when it's set to true. Aha! We now see that isCorrectSet is set to true in a for loop when the condition 
```
JSON.stringify(selectedIndices) === JSON.stringify(correctSets[i])
```
is filled. Since this is the case, we must alter correctSets[i] or selectedIndices from the console.

```
for (let i = 0; i < correctSets.length; i++) {
    correctSets[i] = selectedIndices.map(box => box.index);
}
```

We see that correctSets is now set to what is currently in our game, which makes the if condition we need to fulfill true.
Now let's run the function 
```
checkSelectedSets();
```
There you have it!