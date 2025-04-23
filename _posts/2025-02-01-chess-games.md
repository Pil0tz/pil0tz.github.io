---
layout: post
title: Does spending more time on a chess move increase its quality? Analyzing 200k online games
categories: Chess
permalink: /chess/analyzing-online-games
cover: /images/covers/chess-games-cover.jpg
exclude: false
---
Chess is as much about finding the best move as it is about managing your available time, but does going 'into the tank' actually result in a better move? In this article I try to make sense of hundreds of thousands of moves, and learn a thing or two about continuous variables in the process.

----
<!--more-->
## Dataset
The data I used comes from [Chessdigits.com](https://web.chessdigits.com/data){:target="_blank"}, who mined and converted 200.000 online games. They were played on the [Lichess](https://lichess.org){:target="_blank"} website, the second most popular place to play chess on the internet, in 2019. Each row represents one game. The two most important columns for this analysis are:
- **Eval_ply_x:** the computer evaluation of the position _after_ the last played move
- **Clock_ply_x:** the player's remaining time _after_ the last played move

> _'ply'_ in chess means half of a move: so when white starts, their move is ply 1, black's response is ply 2, white's response is ply 3, and so on.

Both of these columns go up to 200 ply, but most games will have a lot of missing values towards the end of that range.
The computer evaluation is based on the best move according to a chess playing computer like [Stockfish](https://www.chess.com/terms/stockfish-chess-engine){:target="_blank"}, and as we'll see later on, it's not always the perfect way to judge which player has the better or easier to play position.


## Exploratory Data Analysis
The first step in understanding the dataset is to calculate some descriptive statistics. Below are histograms of the most important features that describe the games, such as the rating of the players, the length of the game, and more. 


<img src="\images\chess\EDA_pre.png" alt="EDA subgraphs" title="EDA subgraphs" style="max-width: 50%;">

From this, we see there's some anomalies that can be removed to get a clearer picture. Rules infractions and abandoned games can go, as well as games where the result seems inconclusive ('*' in the result column). I considered removing draws as well since they only comprise of a small amount of games but their inclusion doesn't hurt our results so I decided to leave them in. I also removed bullet and ultra bullet games since they are so short that no meaningful fluctuation in time spent can be observed.

Increment (the amount of bonus time you get after every move) has some unusual values which are not commonly used. This is probably due to people creating custom game challenges with less used or obscure time controls. 
We can remove the odd ones out to get a clearer picture of the increments that are commonly found in online chess games.

99% of games are a total of 75 moves or shorter, which indicates that we can remove the last 50 ply without losing much data, significantly speeding up the analysis, as well as reducing the skewness of the distribution. We cut off some games before their conclusion, but since we're analyzing on a per-move basis, this should pose too much of a problem .

Finally, to reduce the skew of the rating distribution and make the results more applicable to the general population, we will only look at games with a maximum rating of 2500 for either player, and a rating difference of no more than 200 between them. This is to remove any confounding factors such as the stress / complacency of playing against a higher or lower rated player, respectively.

It's possible that Blitz games prove to be too short as well, so we might revisit this at a later moment.
But for now, _let's clean some data!_


## Data cleaning
```python
df_clean = df[~df['Termination'].isin(["Abandoned", "Rules infraction"])]
df_clean = df_clean[~df_clean['GameMode'].isin(['Bullet', 'UltraBullet'])]
df_clean = df_clean[(df_clean['WhiteElo'] <= 2500) & (df_clean['BlackElo'] <= 2500)]
df_clean = df_clean[df_clean['increment'].isin([0, 15, 3, 2, 1, 5, 10])]
df_clean = df_clean[(df_clean['WhiteRatingDiff'] <= 200) & (df_clean['WhiteRatingDiff'] >= -200)]
```

These relatively simple filters make the dataset more normalized. Now we can plot the descriptive stats again to see if things improved:
{% raw %}
<div style="display: flex; justify-content: space-between;">
<img src="\images\chess\EDA_post.png" alt="EDA subgraphs" title="EDA subgraphs" style="max-width: 50%; margin-right: 20px;">
<table style="border-collapse: collapse; text-align: left;">
    <thead>
        <tr>
            <th style="border-bottom: 1px solid #ddd; padding: 8px;">Feature</th>
            <th style="border-bottom: 1px solid #ddd; padding: 8px;">Skewness</th>
            <th style="border-bottom: 1px solid #ddd; padding: 8px;">Kurtosis</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">Player Ratings</td>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">0.45</td>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">2.67</td>
        </tr>
        <tr>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">Game Length (ply)</td>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">1.12</td>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">3.45</td>
        </tr>
        <tr>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">Time Spent (sec)</td>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">0.89</td>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">2.98</td>
        </tr>
        <tr>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">Increment (sec)</td>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">0.31</td>
            <td style="border-bottom: 1px solid #ddd; padding: 8px;">1.85</td>
        </tr>
    </tbody>
</table>
</div>
{% endraw %}
These values indicate that while some features are relatively symmetric (e.g., Increment), others like Game Length and Time Spent exhibit positive skewness, suggesting a longer tail on the right side of their distributions. Kurtosis values above 3 indicate heavier tails compared to a normal distribution.
## Results per time control

## Results

## Limitations














<!-- Openings are divided into many subvariations, of which there are too many to analyse them all now. Because of that I wanted to condense most of the subvariations into their main openings, and then look at the win rates of the 20 that were most commonly played in the dataset.

Most of this was done by splitting each opening name on the characters ' : ' ' , ' and  ' # ', then only keeping the first part of the opening name, and finally stripping any leading or trailing spaces. -->

