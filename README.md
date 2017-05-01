# AVdatafest-The-Quicksolver
Analytics Vidhya, The Quicksolver Hackathon

### Introduction
R script for team "mark12" to win The Quicksolver hackathon on Analytics Vidhya (https://datahack.analyticsvidhya.com/contest/avdatafest-the-quicksolver)

In short, features were created using Matt Dowle's data.table package and the modeling was done with three H2O models: random forest and two GBMs.

The progression of the models is actually represented in the way the features are laid out. The initial submission scored 1.94 on the public leaderboard (which scored very close to private) and was quickly tuned to about 1.82. This spanned the first six target encodings, no IDs, only the three Article features directly. Over time, I kept adding more (including a pair that surprisingly reduced model quality) all the way to the end. Light experimentation on the modeling hyperparameters, which are likely underfit. Random forest wound up my strongest model, most likely due to lack of internal CV of the GBMs that led to me staying fairly careful.

Feel free to add an Issue if you have a question.
~Mark

### Software
I used R (3.3.1) via RStudio (1.0.136) on a 2014 Macbook Pro (i7, 16GB RAM) and the following three libraries:
* library(data.table) ## data.table 1.10.0
* library(h2o)  ## I used version 3.11.0.3784, which is quite old
* library(bit64)

### Features
I kept the User and Article IDs out of the modeling from the start. At one point I used the most frequent User_ID values, but this did not help - the models were already picking up enough of the user qualities. The main feature style is target encoding, so in the code you will see sets of three lines that calculate the sum of response, count of records, and then performs an average that removes the impact of the record to which it is applying the "average". Data.table makes the syntax concise and thus easy to overlook what is happening.
The data.table[i,j,by] statement is similar to a (WHERE, SELECT, GROUP BY) of sql. And data.table applies the results direclty to the data frame, even while grouping, similar to a windowed function in SQL (but far less cumbersome!). The special .N is a count of records.

Exmple target encoding - little more than the average Rating per user:
```r
full[,nUser:=sum(Rating,na.rm=T),.(User_ID)]
full[,dUser:=sum(set=="train",na.rm=T),.(User_ID)]
full[,tgtUser:=ifelse(set=="train",(nUser-Rating)/(dUser-1),nUser/dUser)]
```

Similarly, from the comments of the code itself:
```
### The pattern is to sum up all the ratings per [feature(s)], and then count how many
###  then when creating the real feature we want, ensure that the record we are applying
###  the average (total ratings / total records) to has been removed from the value.
### There are more complex ways of doing this that also blend other values (e.g. overall mean)
###  but I just did the straightforward way. The downside of that is that it leaves you with
###  NAs when there is only one record (so that you don't overfit the actual record itself).
### For a more complex version of this, see fellow h2o.ai Branden Murray's catNWayAvgCV function 
###  in his "It is Lit" Kernel from Kaggle's recent 2-sigma Renthop problem (line 18)
###  https://www.kaggle.com/brandenkmurray/two-sigma-connect-rental-listing-inquiries/it-is-lit
### This was my first way of approaching the problem, with a few simple ones, then I kept adding
###  them deeper and deeper to keep improving my score
```

### H2O machine learning models
I compete with H2O out of choice because I really like the simplicity of not having to do anything to the features to get models off the ground, and see a rich output of the models - especially while training. Sure, I'm biased ;-) But with money/rankings on the line, I still use H2O though I am not "required" to do so :-)
The short duration and unlimited submissions for this competition kept me moving quickly, but at a disadvantage for model tuning. No doubt these parameters are not ideal, but that was just a choice I made to keep the leaderboard as my improvement focus and iterations extremely short, rather than internal validation. Had either the train/test split or public/private split not appeared random, I would have modeled things differently. But this was a fairly simple situation for getting away with such tactics.

Random Forest was my best individual model. I tried to get 1000 trees, but cut it short at 550 so I could submit it and have a few minutes left. We typically see a balanced view of the features, compared to GBM (often more pronounced than this):
<img width="547" alt="image" src="https://cloud.githubusercontent.com/assets/2976822/25575196/a7f5e9c4-2e1a-11e7-95dc-8fc51c67dcd5.png">

I used a pair of GBMs with slightly different parameters. These features are for the model with 200 trees, a 0.025 learning rate, depth of 5, row sampling of 60% and column sampling of 60%:
<img width="550" alt="image" src="https://cloud.githubusercontent.com/assets/2976822/25575206/bab3885a-2e1a-11e7-8da3-f6c627df4b39.png">

Similarly, from the comments of the code itself:
```
### In the modeling, I must confess that I used the leaderboard to check model progress
###  Using actual cross validation would have required a bit more work to ensure the 
###  target encodings were done in a way similar to the train/test split, and I started 
###  off not doing that, and resisted putting in the time to do it later. The downside
###  is that I couldn't tune the models without making a submission. So, these models can
###  probably be improved--especially by dropping the learning rate and training for longer
###  which I never really wanted to do.
```

### Ensembling
I typically prefer to use all my time working on features. But ensembling is an easy way to improve the score a little bit as well as make your single AV submission a bit more robust. With my (lack of) validation method, I could not have used stacking. So I just used simple fractional blends. Mostly I used two GBMs at 50/50 for slight improvements. After I added the final random forest model and it outscored my GBMs, I added it to the blend with a rough estimate of 60/20/20.

```r
fwrite(blend1[,.(ID,Rating=
                   (blend1$Rating*0.2
                    +blend2$Rating*0.2
                    +blend3$Rating*0.6
                   ))],"blend5.csv")
```
