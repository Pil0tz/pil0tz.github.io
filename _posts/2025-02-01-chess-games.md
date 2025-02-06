---
layout: article
title: Analyzing 200.000 Online Chess Games
tags: Chess
article_header:
  type: cover
  theme: dark
  background-image: 
    src: /assets/images/covers/chess-games-cover.jpg
  image: 
    src: /assets/images/covers/chess-games-cover.jpg
#   background_color: '#203028'
#   background_image: false
---

Openings are divided into many subvariations, of which there are too many to analyse them all now. Because of that I wanted to condense most of the subvariations into their main openings, and then look at the win rates of the 20 that were most commonly played in the dataset.
<!--more-->

Most of this was done by splitting each opening name on the characters ' : ' ' , ' and  ' # ', then only keeping the first part of the opening name, and finally stripping any leading or trailing spaces.
