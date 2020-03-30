# Basketball
NCAA Basketball Analysis using Python, Data Science, and Analytics

![basketball](https://github.com/elizkhan/Basketball/blob/master/basketball.png)

# Project Description:
I want to see how accurate the AP 25 Preseason Rankings for Men’s basketball are compared to how well teams perform in the Regular Season and where they end up in the NCAA tournament. To get started, I am utilizing the NCAA Men’s Basketball data accessible through Google Big Query as a public dataset. To learn more about how to connect to this for free check out my other tutorial [here.](http://github.com/elizkhan/CloudSetup)

1. Import libraries

```javascript
#Import Google BigQuery Library
from google.cloud import bigquery

#Import Standard Libraries
import pandas as pd
import math
import numpy as np

#Measure program's execution time
import time

#start script
script_start = time.time()


#Import seaborn and matplot lib

import seaborn as sn
import matplotlib.pyplot as plt
```

2. Read Local Files and use Google Big Query to pull down NCAA Regular Season and NCAA Tournament datasets

```javascript
# Read Local AP Preasons Rankings CSV

AP_Rank = pd.read_csv(r'C:\Users\Liz\Documents\AP_Mens_NCAA_Preseason_Ranks.csv')
Coaches_Rank = pd.read_csv(r'C:\Users\Liz\Documents\Coaches Poll Post Season Rank.csv')
#Connect to Google Big Query to Pull Down NCAA Mens Tournament Data
client = bigquery.Client()

#Pull Down Game Level Winning and Losing Teams 
query1 = """
select distinct hist.season,hist.tournament_type, hist.game_id, hist.scheduled_date, case when hist.win=True then name.school_ncaa else name2.school_ncaa end as WTeamName,  case when hist.win=True then hist.team_id else hist.opp_id end as WTeamID,case when hist.win=False then name.school_ncaa else name2.school_ncaa end as LTeamName, case when hist.win=False then hist.team_id else hist.opp_id end as LTeamID 
from `bigquery-public-data.ncaa_basketball.mbb_teams_games_sr` hist
left join `bigquery-public-data.ncaa_basketball.mbb_teams` name on hist.team_id=name.id 
left join `bigquery-public-data.ncaa_basketball.mbb_teams` name2 on hist.opp_id=name2.id
order by hist.scheduled_date asc
"""
query2 = """
select season, round, game_date,win_school_ncaa as WTeamName, win_team_id as WTeamID, lose_school_ncaa as LTeamName, lose_team_id as LTeamID
from `bigquery-public-data.ncaa_basketball.mbb_historical_tournament_games`
where season between 2013 and 2018
order by season, game_date asc
"""

query3 = """
select team_id as TeamID, season, wins,losses, wins-losses as net_wins, rank() over(partition by season order by season, wins desc) as regular_season_rank
from `bigquery-public-data.ncaa_basketball.mbb_historical_teams_seasons`
where season between 2013 and 2018
order by season, wins desc
"""

#Pull down as dataframe in memory
ncaa_games = client.query(query1).to_dataframe()
print(ncaa_games.head())

ncaa_tourney = client.query(query2).to_dataframe()
print(ncaa_tourney.head())

reg_season_all = client.query(query3).to_dataframe()
print(reg_season_all.head())

```

# Regular Season Games:
To see how teams are performing during the regular season I will use an Elo Ranking score which will update a team’s ranking based on each win and loss during the regular season. This will factor in the probability of a team beating another team to update the Elo Ranking. I have assigned K to be 20 which will be used to update the scores. All teams will start at a score of 1200. 

```javascript
team_rank['score'] = 1200

#Define Elo Function to update scores after a win/ loss

def Probability(rating1, rating2): 
  
    return 1.0 * 1.0 / (1 + 1.0 * math.pow(10, 1.0 * (rating1 - rating2) / 400)) 


def EloRwting_Win(Rw, Rl, K): 
   
    # Probability of Losing Team
    Pl = Probability(Rw, Rl) 
  
    # Probability of Winning Team 
    Pw = Probability(Rl, Rw) 
  
    #Winning Team Id is winner
    Win = Rw + K * (1 - Pw) 
    Loss = Rl + K * (0 - Pl) 
    
    Win = round(Win, 6)
    Loss = round(Loss, 6)
    print('Winning Team is now: '+str(round(Win, 6)), 'Losing Team is now: '+str(round(Loss, 6)))
    return Win

def EloRwting_Loss(Rw, Rl, K): 
   
    # Probability of Losing Team
    Pl = Probability(Rw, Rl) 
  
    # Probability of Winning Team 
    Pw = Probability(Rl, Rw) 
  
    #Winning Team Id is winner
    Win = Rw + K * (1 - Pw) 
    Loss = Rl + K * (0 - Pl) 
    
    Win = round(Win, 6)
    Loss = round(Loss, 6)
    print('Winning Team is now: '+str(round(Win, 6)), 'Losing Team is now: '+str(round(Loss, 6)))
    return Loss
    
    elo_start = time.time()

for team in range(len(ncaa_games)):
    Rw = team_rank[['index1','score']][(team_rank['TeamID']==ncaa_games['WTeamID'][team])&(team_rank['season']==ncaa_games['season'][team])].values[0]
    Rl = team_rank[['index1','score']][(team_rank['TeamID']==ncaa_games['LTeamID'][team])&(team_rank['season']==ncaa_games['season'][team])].values[0]
    team_rank.loc[Rw[0],'score'] =  EloRwting_Win(Rw[1], Rl[1], K=20)
    team_rank.loc[Rl[0],'score'] = EloRwting_Loss(Rw[1], Rl[1], K=20)
    
    elo_end = time.time()
print('ELO Calculation Time in Minutes:'+str((elo_end - elo_start)/60))


```


# Comparison:
How accurate is the Preseason Top 25 Ranking really? I will evaluate performance of a top 25 ranking by comparing the Regular Season Elo Ranking and Coaches Post season rank to the AP preseason ranking.  To do this comparison, I will be using a correlation and confusion matrix to assess the accuracy of Pre-season rankings.

1. Use a correlation matrix to see if there are any relationships between rankings. 

```javascript

features = ['AP_Preseason_Rank','elo_rank','Post_Season_Coach_Poll_Rank']

corr = all_ranks[features].corr()

sn.heatmap(corr,annot=True)
plt.show()

```

### Results:
There appears to be a moderate correlation between the AP Preseason Rankings and the Post Season Coaches Polls.

![correlationmatrix](https://github.com/elizkhan/Basketball/blob/master/CorrelationMatrix.PNG)



2. Use a Confusion Matrix to evaluation the accuracy of the Preseason Preditions compared to the Regular Season (Elo Score Rankings) and Post Season Results (Coaches Poll Rankings).
To calculate the Accuracy we will combine the True Positive and True Negative divided by the total. `True Positive + True Negative / Total`


```javascript
# Create Confusion Matrix to Understand Accuracy of Preseason compared to Actual Regular Season Elo rank
confusion_matrix = pd.crosstab(top25['AP_Preseason_top25'],top25['elo_rank_top25'],rownames=['Actual-Regular Season'],colnames=['Predicted- AP Preseason'])
sn.heatmap(confusion_matrix, annot=True)
plt.show()

# Create Confusion Matrix to Understand Accuracy of Preseason compared to Actual Post Season Coaches Poll 
confusion_matrix = pd.crosstab(top25['AP_Preseason_top25'],top25['coaches_poll_top25'],rownames=['Actual-Coaches Final Season Poll'],colnames=['Predicted- AP Preseason'])
sn.heatmap(confusion_matrix, annot=True)
plt.show()
```

### Results:
The Preaseason Rankings have **65%** accuracy for predicting which Teams will still be in the Top 25 by Postseason. 

#### Confusion Matrix between Preaseason AP Top 25 and Post Season Coaches Poll

![confusionmatrixcoachpoll](https://github.com/elizkhan/Basketball/blob/master/confusionmatrixcoachespoll.PNG)

#### Confusion Matrix between Preseason AP Top 25 and Regular Season Elo Ranking

![confusionmatrixelo](https://github.com/elizkhan/Basketball/blob/master/confusionmatrixeloranking.PNG)
