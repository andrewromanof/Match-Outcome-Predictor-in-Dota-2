# Match Outcome Predictor in Dota 2

## Important Files

All_Heroes_ID.csv - Contains the unique id linked to the hero

Fetching dota 2 data.ipynb - Notebook that was used to collect and generate all data

Final_project.ipynb - Notebook that was used to experiment on the data and collect and visualize results

recent_matches.json - A list of all matches that was collected from Opendota API

filtered_matches.json - A list of all matches from recent_matches.json that met a criteria

# Video presentation

https://www.youtube.com/watch?v=5LEm_U2f7Zg

## Overview

This project attempts to create a match predictor to predict the outcome of any given match in the game of Dota 2. This project explores a custom made dataset that was made recent from fresh matches and explores different features in the dataset to see not only their impact on the match predictor but also ways to improve the match predictor by providing the data in different ways. This project utilizes a decision tree to visualize the model and data.

## Background

Dota 2 is a strategic multiplayer online video game where players control virtual characters called heroes. It works as a match system where you can queue up and find a match and get put on a team and play until you win or lose the match. There are different types of gamemodes, but this project will focus only on Ranked All Pick. Each match contains two teams, radiant and dire, and each team will consist of 5 players. There is a draft phase where players may select their hero out of 124 heroes (currently) and no hero may be picked twice. After the draft is over, the match begins and the first team to destroy a building called the throne, wins the match.

## Data collection

I created my own dataset by fetching fresh and recent matches from the opendota API (link in references). I first fetched 5000 unique matches from their first endpoint /publicMatches using HTTP requests. Then with those 5000 unique matches, I filtered out the matches whose gamemode was not Ranked All Pick and was left with 4875 matches. Using the remaining 4875 matches, I did another API call on each of these 4875 matches individually to collect all the possible data I could get on them by calling their endpoint /matches/{match_id} using HTTP requests. I saved all this data to a .json file.

_The code that collected the data can be found in Fetching_data.ipynb_

## Data preprocessing

There was a lot of preprocessing to be done, it included:

- Removing features: There were a lot of features included in the data that didn’t have any contribution towards the match (i.e. match_id, chat, cosmetics, etc), a lot of features that are constant (i.e. pre_game_duration, patch, etc) and just some features that were returned NaN by the API but could have been useful (i.e. draft_timings, teamfights, etc)
- Extracting features from array: One of the features, picks_bans, was given as an array of an array that contains four variables, is_pick, hero_id, team, and order, and the decision tree doesn’t take the data in that format so it had to be extracted into more columns. The variable is_pick also only returned True regardless of whether it was a pick or ban so this variable was ignored. The other three variables were transformed into 20 columns where it was pick_{order} and pick_{order}_team.
- Merging columns: There are only a total of 5 features but 27 columns. To help reduce the amount of columns, the features that contained scores (kill score, tower score, barracks score) for both teams were combined into one as the difference (radiant - dire). The 20 pick columns that were formed from extracting features was also shortened to 10 by including the side in the column header and breaking it into phases so it would be in the format of {side}_P{phase}P{order}
- Removing bad data: As mentioned earlier, the is_pick variable returned True regardless if it was a pick or ban so this caused some bad data to form. In order to compensate, any row with bad data was ignored. A row with bad data was classified if the pick_ban array:
  - Didn’t have a length of exactly 10
  - Didn’t have an even balance of teams (5 heroes per team)

After all the preprocessing, there are 1820 rows of data left.

*The code containing all the preprocessing can be found in Final_project.ipynb*

## Creating the model

The model chosen to train and test on the data was a decision tree. The DecisionTreeClassifier and export_graphviz libraries from sklearn.tree are used for the model and to visualize the model. The training and testing process consists of:
- Split the data into 80% training, 20% testing
- Split into labels, train_data, test_data is data without the class and train_labels_mc, test_labels_mc is the correct classification for the data
- Compile the tree model on max depth being set as the amount of features
- Fit the tree model on the training data
- Use graphviz to export the model as a .dot file
- Convert the .dot file into a .png file and export it
- Evaluate the model using the testing data and based on the accuracy


## Experimenting with the model

Given all the features, I had categorized the features into two categories: After the match features (score, tower, barracks, duration) and before the match features (picks). I had a hypothesis that with after the match features, the model would easily be able to produce good results but as they get removed, the accuracy would plummet. To prove my hypothesis, I would use all the features on the model and remove the root feature of the decision tree until:
1. There were no more after the match features left
2. A before the match feature is the root node while there are still after the match features

My hypothesis proved correct as all the after the match features got removed and the accuracy hovered ~50%. Below is a table to show the timeline as the features got removed and the accuracy of the model.

| Features      | Accuracy      |
| :-----------: |:-------------:|
| [score, tower, barracks, duration, picks]      | 99.7% |
| [score, barracks, duration, picks]     | 92.3%     |
| [score, duration, picks] | 86.9%      |
| [duration, picks] | 49.7%      |
| [picks] | 51.4%      |


The three key features that kept the model accuracy high were the score, tower, barracks. Unfortunately, these are not available until a match is over and the goal is to create a match predictor that can predict the outcome of a match before it begins.

## Improving accuracy using phases

Right now, the model only produces a 51.4% accuracy rate using the picks data. This is not great considering the classification only has two options, a coin flip could produce the same accuracy. The first step is, instead of looking at picks as a whole, let’s start looking at it in phases. The actual draft stage in dota has 3 phases:
- Phase 1: Each team picks two heroes blind
- Phase 2: Each team picks two heroes, able to see the picks from phase 1
- Phase 3: Each team picks one hero, able to see the picks from phase 1 and 2

So instead of feeding the model the full dataset, it is fed in phases to see if it can draw better conclusions looking at individual phases. Here are the results:

| Phase      | Accuracy      |
| :-----------: |:-------------:|
| 3      | 50.3% |
| 2     | 52.7%     |
| 1 | 54.7%     |


Looking at phase 3, there is already a slight improvement. This result makes sense as phase 3 generally has the most information and can make the most informed decision to pick the best possible hero for the game which would lead to heroes picked in phase 3 having higher win rates than heroes picked in the other phases.

## Math and statistics on the data

The accuracy already showed improvement using phases, so the data will be kept in phases. Mathematically, there will probably never be enough data for the model to see every single combination of heroes. Here is some math:
- Total of 124 different heroes to choose from
- 5 possible hero combinations: (124! / (5! (124-5)!))
- 5 possible hero combinations from the remaining heroes: (119! / (5! (119-5)!))
- Then side side doesn't matter, so divide by 2:
  - ((124! / (5! (124-5)!)) * (119! / (5! (119-5)!))) / 2
  - = 20,560,393,199,622,276

That is about 20 quadrillion unique hero combinations there can be, the 1820 rows of data that is available doesn’t even compare.

## Using meta to improve accuracy

As it can be seen from the math above, it is near impossible to have enough data to have every combination of heroes checked off in our model. However, in a game like dota 2, there is a meta where certain heroes show up very frequently and certain heroes show up very infrequently. The meta can be used to adjust our dataset which will reduce the hero pool the model sees drastically. Here is an example of meta:


| Side_PhasePick      | Top 3 Picks      |
| :-----------: |:-------------:|
| Radiant_P1P1      | [123, 14, 86] |
| Radiant_P1P2     | [123, 14, 26]    |
| Dire_P1P1 | [123, 14, 53]     |
| Dire_P1P2 | [123, 86, 53]   |


In the table above, the top 3 most frequent heroes in each column are shown. The hero with id 123 is the most frequent hero in all four columns showing it is the most contested hero in the first phase.  The win rate is harder to use as for most heroes it usually hovers around 50% so it doesn’t add more meaning to the data.

For the next part of the attempt to improve the accuracy, three copies of the data frame will be created and rows will be filtered out. There will be a set (list with no duplicates) of the top 3 heroes from each column. If a row does not contain at least 1, 2, or 3 heroes from this set then it will be removed respectively from the copies of the data frame.

Below are the results of the model on the copies of the dataframes. It is shown that our accuracy does improve and now, looking at only games with 2 meta heroes and specifically at phase 3, our model is able to accurately predict almost 60% of the matches correctly before the match begins.


| Dataframe      | Phase [1, 2, 3] Accuracy     |
| :-----------: |:-------------:|
| 1      | [53.6%, 53.3%, 55.6%] |
| 2    | [51.2%, 49.1%, 59.8%]    |
| 3 | [46.1%, 49.4%, 47.8%]     |


## Limitations

One of the biggest issues with using only meta data is underfitting. It can be seen from the table above that by filtering for at least 3 meta heroes, the accuracy drops hard and it is due to a lack of data. The meta also shifts as new patches come out, so the model would have to be regularly updated.

## Ethics

For what I have done in my project so far, there are currently no ethical repercussions as my model isn’t accurate enough to predict with great certainty. In the future, there may be a cause for concern of misuse for gambling. Currently, there is no bias as the data only involves features within the game, but if the data were to start including features about the players in the game as well, there might be some bias that comes up.

## Conclusion and future directions

In the future, I would include a lot more data, to try and aim for at least 50,000 rows of valid data to have more leniency in the data manipulation as well as for the model to have more data to learn off of. I would also look into collecting more features that could provide the model more information such as player information (i.e. win rate, games played, etc) to take that into account as well. In conclusion, I didn’t achieve the same accuracy as the model had when it had the three key features, score, tower score, barracks score. However, I did manage to improve the model slightly using only data manipulation techniques from ~50% to ~60%.

## References

https://www.statisticshowto.com/probability-and-statistics/probability-main-index/permutation-combination-formula/

https://www.wolframalpha.com/input?i=%28%28124%21+%2F+%285%21+%28124+-+5%29%21%29%29+*+%28119%21+%2F+%285%21+%28119+-+5%29%21%29%29%29%2F2

https://www.opendota.com/api-keys

