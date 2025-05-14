---
layout: post
title: Does spending more time on a chess move increase its quality? Analyzing 200k online games
categories: Chess
permalink: /chess/analyzing-online-games
cover: /images/covers/chess-games-cover.jpg
exclude: false
upcoming: false
---
Chess is as much about finding the best move as it is about managing your available time, but does going 'into the tank' actually result in a better move? In this article I try to make sense of hundreds of thousands of moves, and learn a thing or two about continuous variables in the process.

----
<!--more-->
## Dataset & Domain Understanding

Analysis will be done using Python3.12. Specifically, the Pandas and NumPy packages for analysis and data cleaning, along with Matplotlib and Seaborn for visualizations.
These are the tools I started with when learning data analysis, although I'm very interested in trying out alternatives like [Polars](https://pola.rs/) or [Plotly](https://plotly.com/product-tour/), which I plan to write an article about in the near future.

The dataset comes from [Chessdigits.com](https://web.chessdigits.com/data){:target="_blank"}, who mined and converted 200.000 online games. They were played on the [Lichess](https://lichess.org){:target="_blank"} website, the second most popular place to play chess on the internet, in 2019. Each row represents one game. The two most important columns for this analysis are:

- **Eval_ply_x:** the computer evaluation of the position _after_ the last played move
- **Clock_ply_x:** the player's remaining time _after_ the last played move

> _'ply'_ in chess means half of a move: so when white starts, their move is ply 1, black's response is ply 2, white's next move is ply 3, and so on.

Both of these columns go up to 200 ply, but most games will have a lot of missing values towards the end of that range.
The computer evaluation is based on the best move according to a chess playing computer like [Stockfish](https://www.chess.com/terms/stockfish-chess-engine){:target="_blank"}, and as we'll see later on, it's not always the perfect way to judge which player has the better or easier to play position.

> When computer evaluations go below zero, this means *black has the better position*. If they are positive, then *white has the better position*.

**Other important columns include:**

- 
- **\[White/Black\]Rating:** the [ELO Rating](https://www.chess.com/terms/elo-rating-chess) of each of the players. It is a  historically tried and true measure of their relative strength.
- **TimeControl:** in the form **'N+K'**, where **N** is the number of minutes each player starts with, and **K** is their _increment_
- **Event:** a categorical grouping of the time control, into _Ultrabullet, Bullet, Blitz, Rapid_ and _Classical_. 

> Increment is the amount of time a player gets **after** making a move

## Exploratory Data Analysis

The first step in understanding the dataset is to calculate some descriptive statistics. Below are histograms of the most important features that describe the games, such as the rating of the players, the length of the game, and more. Originally the code block below contained 5 lines of brute-force 'code', where I wrote out the lists containing the rating ranges and labels manually after looking through the data, before realising that this approach kind of defeats the purpose of programming. So I wrote a function that will create the rating ranges before plotting, ensuring the graph will still look correct even after the data cleaning process. The result is a quite a couple of lines longer, and took more time to write, but is also a lot more reusable and taught me a lesson; a tradeoff I am more than willing to make.

{% include /code-blocks/chess/descriptive_stats.html %}

<img src="\images\chess\EDA_before.png" alt="EDA subgraphs" title="EDA subgraphs" style="width: 100%;"><br>

From this, we see there's some anomalies that can be removed to get a clearer picture. Rules infractions and abandoned games can go, as well as games where the result seems inconclusive ('*' in the result column). I considered removing draws as well since they only comprise of a small amount of games but their inclusion doesn't hurt our results so I decided to leave them in. I also removed bullet and ultra bullet games since they are so short that no meaningful fluctuation in time spent can be observed.

Increment has a lot of unusual values which are not commonly used. This is probably due to people creating custom game challenges with less used or obscure time controls. 
We can remove the odd ones out to get a clearer picture of the s that are commonly found in online chess games.

99% of games are a total of 75 moves or shorter, which indicates that we can remove the last 50 ply without losing much data, significantly speeding up the analysis, as well as reducing the skewness of those distributions. We cut off some games before their conclusion, but since we're analyzing on a per-move basis, this should pose too much of a problem .

Finally, to reduce the skew of the rating distribution and make the results more applicable to the general population, we will only look at games with a maximum rating of 2500 for either player, and a rating difference of no more than 200 between them. This is to remove any confounding factors such as the stress / complacency of playing against a higher or lower rated player, respectively.

It's possible that Blitz games prove to be too short as well, so we might revisit this at a later moment.
But for now, _let's clean some data!_

## Data cleaning

```python
df_clean = df[~df['Termination'].isin(["Abandoned", "Rules infraction"])]
df_clean = df_clean[~df_clean['GameMode'].isin(['Bullet', 'UltraBullet'])]
df_clean = df_clean[(df_clean['WhiteElo'] <= 2500) & (df_clean['BlackElo'] <= 2500)]
df_clean = df_clean[df_clean['Increment'].isin([0, 15, 3, 2, 1, 5, 10])]
df_clean = df_clean[(df_clean['WhiteRatingDiff'] <= 200) & (df_clean['WhiteRatingDiff'] >= -200)]

for col_prefix in ['Eval_ply_', 'Move_ply_', 'Clock_ply_']:
    columns_to_drop = [f'{col_prefix}{i}' for i in range(151, 201)]
    df_clean = df_clean.drop(columns=columns_to_drop)
```

### Data cleaning results

```markdown
Removed    17 Games with irregular Termination
Removed 43253 Bullet & UltraBullet games
Removed   310 Games with 2500 or higher ELO players
Removed  3503 Games with non-standard increments
Removed  4417 Games with RatingDiff of 200 or more
Removed columns for the last 50 ply columns

Removed 47997 games in total
Remaining games: 152003 from 200000 (76.00%)
Old shape: (200000, 628)
New shape: (152003, 478)
```
<br>
These relatively simple filters make the dataset more normalized and remove outliers on key features. 
Now we can plot the descriptive stats again to see if things improved:

<img src="\images\chess\EDA_after.png" title="EDA subgraphs" style="display: flex; max-width: 100%;"><br>

Those graphs already look a lot cleaner. 
Rating distribution skewness went from 0.24 to 0.14, and game length skewness from 1.17 to 0.68. This makes the distribution measurably more symmetrical.
Removing the odd increments also decreases the influence that rare custom time controls have on the data.

## Extracting Time and Evaluation Difference

To be able to answer our research question, we need to make a dataframe containing the time spent on each move, together with the evaluation difference between that move and the next one. An important factor to include is the increment the player gets, as it will be added to the clock after each move and thus isn't actually time spent on making the move itself. 

First, we make filtered dataframes containing only the relevant columns. Then we combine those, before finally starting on the bulk of the data processing for this project. The final pipeline is the result of a lot of trial and error, pushing my Python skills to it's limit. 

I've tried to explain my thought process as much as possible, but if you just want to go to the results, you can click the button below to collapse the code.

{% include /code-blocks/chess/time_eval-func.html %}

## Results

Now that we have our DataFrame on a per move basis, a second round of data examination and cleaning begins. 
First, let's look at the boxplots for evaluation change and time spent to check for any outliers. The scatterplot also helps show how the data is distributed.

<img src="\images\chess\move_eval_boxplot.png" title="EDA subgraphs" style="display: flex; max-width: 100%;">
<img src="\images\chess\eval_time_scatter.png" title="EDA subgraphs" style="display: flex; max-width: 80%;">

There are some unexpected things happening here:

<u>Some moves seem to take <i>negative time</i> to make</u>, even after taking the increment into account.
While it could potentially be an error in the code, this is likely due to a feature on lichess where your opponent can give you extra time on your clock,
which will show up as negative time spent on that move.
Some of the extremely large time spent values can also be due to this feature, as it allows a player to have much more time on their clock than the format should allow. With some exceptions, most of the large time spent comes from classical games, which can last for hours, so that makes sense.

This is also where we see a problem arise with the **interpretability of computer evaluations**.
During the processing, we ommitted any move where the current or previous move contained a forced mate evaluation, as those cannot be easily converted into a numerical value. Mate in one, two or three moves can be conceptualized as being relatively easy to find, but if the computer sees forced mate in 20 moves, and _every other sequence of moves_ leads to a much worse position, the position will be **much worse in practical terms**. A chess computer has no idea on how to differentiate those, as the evaluation of a position is based on the best move in that position.
Similarly, when a position is really good but the computer can't find a sequence leading to forced mate, it can produce some crazy high evaluations.

**Good to know:** chess evaluation is supposed to be mapped back to the value of the pieces. So if the evaluation is +9, this should roughly mean white is up the equivalent of a full queen.
> **This leads to some questions about interpretability**:

- If the evaluation is already +20 for your opponent, and your move takes it to +50, did the position get worse in human terms?
- If the inverse happens, and you manage to 'only' be down the equivalent of two queens instead of four, did your chances to win really improve?

In other words, **computer evaluation cannot be seen as a continuous scale on which to evaluate a position**. Go past a threshold of, say, 10 and subsequent increases start to matter less and less, in practical human terms.
Comparison with the second best computer move would provide some insight into how critical finding a specific move is, but that isn't available in this dataset.
For more analysis about the relationship between computer evaluation and winning chances, check out [this article](https://web.chessdigits.com/articles/when-should-you-resign) by the creator of the dataset.

## What's next?

Originally, and perhaps naively, I was planning to just do regression and correlation analyses on the extracted data.
But after delving into the results, it seems that it is very hard to obtain useful insights from computer evaluation change **as an absolute numerical value**.
Its value and relevancy fluctuates too drastically based on how close to 0 it is.

A more reasonable approach would be to turn evaluation changes into **categories**:

- If a move brings the evaluation from around 0 to going drastically one way or the other, we call it a _blunder_
- If a move changes the evaluation down from a completely winning advantage to just a favored one, we call it a _mistake_

Analzing data like this removes the scale problem of evaluations, and focuses on the question at hand; if spending more time leads to fewer _mistakes_.

<br>

----

<br>

For those of you that have read this post all the way through, thank you so much! Your attention and support means a lot to me.
As a parting gift, I made this nice graph of the top 20 most commonly played openings in the dataset. Check it out below!

<br>

<img src="\images\chess\win_percentages_transparent.png" title="EDA subgraphs" style="display: flex; max-width: 80%;">

<!-- Openings are divided into many subvariations, of which there are too many to analyse them all now. Because of that I wanted to condense most of the subvariations into their main openings, and then look at the win rates of the 20 that were most commonly played in the dataset.

Most of this was done by splitting each opening name on the characters ' : ' ' , ' and  ' # ', then only keeping the first part of the opening name, and finally stripping any leading or trailing spaces. -->