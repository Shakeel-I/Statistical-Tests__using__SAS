# STATISTICAL TESTS USING SAS SQL

## 1. Calibration Plot
<img width="640" height="480" alt="image" src="https://github.com/user-attachments/assets/2d395069-45a8-41ff-b9c1-1c6c1fd7a19f" />

A calibration plot helps assess how well predicted probabilities from a model match the actual observed outcomes. It plots mean predicted probabilities on the x-axis versus the fraction of positives (observed event rate) on the y-axis.

Step 1: Fit logistic regression model and output predicted probs
```sas
proc logistic data=myhmeq noprint;
  model bad(event='1') = debtinc;
  output out=predout predicted=phat;
run;
```
Step 2: Bin predicted probabilities into deciles
```sas
proc rank data=predout groups=10 out=binned;
  var phat;
  ranks phat_bin;
run;
```
Step 3: Calculate mean predicted (phat) and mean observed (BAD) outcome per bin
```sas
proc means data=binned noprint;
  class phat_bin;
  var phat bad;
  output out=cal_data mean=mean_phat mean_bad;
run;
```
Step 4: Plot calibration curve
```sas
proc sgplot data=cal_data(where=(_type_=1));
  scatter x=mean_phat y=mean_bad / markerattrs=(symbol=CircleFilled) name='pts';
  lineparm x=0 y=0 slope=1 / lineattrs=(color=gray pattern=shortdash) name='ref';
  xaxis label='Mean Predicted Probability';
  yaxis label='Observed Default Rate';
  title 'Calibration Plot: Predicting Loan Default (BAD) from DEBTINC';
run;
```


If the points lie close to the diagonal line, the model is well calibrated, meaning its predicted probabilities accurately reflect real-world risks. Deviations from this line reveal if the model tends to over- or under-predict outcomes, making calibration plots a straightforward and visual tool to evaluate the reliability of probabilistic predictions. This helps ensure predictions can be trusted for decision making.



## 2. ROC Curve
The ROC (Receiver Operating Characteristic) curve visually shows the trade-off between sensitivity (true positive rate) and 1-specificity (false positive rate) across different threshold levels of a classification model. It illustrates how well the model can distinguish between positive and negative cases at various thresholds, with the area under the curve (AUC) indicating overall discriminative ability.

```sas
proc logistic data=Employment plots(only)=roc;
model Employed=Education Experience;
run;
```
<img width="480" height="480" alt="image" src="https://github.com/user-attachments/assets/333ff978-df5c-421b-9205-1c40334c7553" />

An AUC of 0.5869 in this context means that there is a 59% chance that the model will correctly distinguish between an employed and a not-employed individual. This indicates fairly good discriminatory power, suggesting the model is fairly effective at predicting employment status, though there's still some room for improvement.


## 3. Kolmogorov-Smirnov Test
The Kolmogorov-Smirnov (K-S) chart is used to evaluate the discrimination ability of a credit scoring model by measuring how well the model distinguishes between good and bad borrowers. It plots the cumulative distribution of good and bad borrowers across the score spectrum.
<img width="640" height="480" alt="image" src="https://github.com/user-attachments/assets/99d8db2a-af36-465f-9a34-7865a4107d9c" />

![KS1](https://github.com/user-attachments/assets/35c8eeb7-44d8-4d77-9c71-a9abf87c1c27)

```sas
proc rank data=loandata groups=70 out=loandata1;
    var score;
    ranks score_bin;
run;

proc sql;
create table loandata2 as
select score_bin, 
        sum(case when default=0 then 1 else 0 end) as goods,
        sum(case when default=1 then 1 else 0 end) as bads
    from loandata1
    group by score_bin;
quit;

data loandata3;
    set loandata2;
    retain cum_goods cum_bads 0;
    cum_goods + goods;
    cum_bads + bads;
run;

proc sql;
    create table loandata4 as
    select score_bin, cum_goods, cum_bads,
           total_goods,
           total_bads,
           cum_goods/total_goods as cum_prop_goods,
           cum_bads/total_bads as cum_prop_bads
    from (
        select score_bin, cum_goods, cum_bads,
               sum(goods) as total_goods,
               sum(bads) as total_bads
        from loandata3
    ) as totals;
quit;

proc sgplot data=loandata4;
   series x=score_bin y=cum_prop_goods / lineattrs=(color=blue thickness=2) legendlabel="Cumulative Proportion Goods";
   series x=score_bin y=cum_prop_bads / lineattrs=(color=red thickness=2) legendlabel="Cumulative Proportion Bads";
   xaxis label="Score Bins";
   yaxis label="Cumulative Proportion";
   title "Kolmogorov Smirnov statistic";
run;
```

The K-S statistic, represented by the maximum difference between these two cumulative curves, indicates the model's effectiveness: a higher K-S value suggests better discrimination between good and bad credits.



## 4. Hosmer-Lemeshow Goodness-of-Fit Test
The Hosmer-Lemeshow test evaluates the goodness-of-fit for logistic regression models by comparing the observed and expected frequencies of outcomes in groups of data. It helps determine whether the model adequately fits the data.

H₀ (null hypothesis): The model fits the data well; there is no significant difference between observed outcomes and those predicted by the model across the groups.

![Hosmer1](https://github.com/user-attachments/assets/69656266-22d6-405b-b825-4fba0585affc)
```sas
proc logistic data=Employment plots=none;
model Employed=Education Experience/gof lackfit;
run;
```
![Hosmer2](https://github.com/user-attachments/assets/fb10f6e1-e977-4a29-b036-8c524b75fded)

Since the p-value from the test is above 0.05, we fail to reject H₀, suggesting our logistic regression model has a good fit to the data.




## 5. Brier Score Test
Brier score is an important measure of calibration. It evaluates the accuracy of probabilistic predictions by measuring the mean squared difference between predicted probabilities and actual outcomes (0 or 1). It assesses how well the predicted probabilities are calibrated, with lower scores indicating more accurate and well-calibrated predictions.

![Brier](https://github.com/user-attachments/assets/5b22ddd1-7e74-4d8e-bf51-42a0ac7f19fe)


- Brier Score is 0, the best score achievable.
- The model below gives a Brier score of 0.08 which is quite good.

```sas
%let seed = 12345;
```
Create column 'random' to assign random number between 0 and 1
```sas
data partitioned;
	set mydata.hmeq;
	random=ranuni(&seed);
run;
```
Partition into training_set 80% and validation_set 20%
```sas
data training_set validation_set;
	set partitioned;
	if random < 0.8 then
		output training_set;
	else
		output validation_set;
run;
```
Drop column 'random'
```sas
data training_set;
	set training_set;
	drop random;
run;
```
Drop column 'random'
```sas
data validation_set;
	set validation_set;
	drop random;
run;
```
Train model on training_set then SCORE model on validation_set
```sas
proc logistic data=training_set desc;
	class job reason /param=glm;
	model bad=debtinc delinq loan reason;
	score data=validation_set out=predicted_probabilities fitstat;
run;
```

## 6. PSI : Population Stability Index
PSI measures to what extent the population that was used to construct the rating system is similar to the population that is currently being observed (Baesens et al).
Banks use a Traffic Light System (RAG - Red, Amber, Green) status when tracking Population Stability Index (PSI) metrics. Industry standard thresholds used for Total PSI are:

      🟢 Green (PSI < 0.10): Minimal shift; the current population is stable compared to the base.
	  
      🟡 Amber (0.10 ≤ PSI < 0.25): Moderate shift; requires monitoring and investigation.
	  
      🔴 Red (PSI ≥ 0.25): Significant shift; requires action (potential model recalibration or redevelopment).
	  
PSI is calculated using the formula: (Current Percentage - Base Percentage) x LN (Current Percentage / Base Percentage).
In the PSI tables below, we can observe a moderate shift in the first table and a significant shift in the second table.

<img width="328" height="580" alt="PSI" src="https://github.com/user-attachments/assets/65b006b4-b9c6-40e0-bd4c-f6fb1fb46fed" />

Define Industry-Standard Regulatory RAG Thresholds for Reporting
```sas
proc format;
    value rag_colors
        low  - <0.10  = 'lightgreen'
        0.10 - <0.25  = 'lightorange'
        0.25 - high   = 'lightred';
run;
```
Create Baseline and Current Population Datasets
```sas
data moderate_PSI;
    length PD_Score $10;
    input PD_Score $ Base_Count Current_Count;
    format Base_Count Current_Count comma10.;
    datalines;
<100       100 98
100<=200   100 89
200<=300   100 100
300<=400   100 100
400<=500   100 92
500<=600   100 94
600<=700   100 95
700<=800   100 96
800<=900   100 100
900+       100 98
;
run;

data significant_PSI;
    length PD_Score $10;
    input PD_Score $ Base_Count Current_Count;
    format Base_Count Current_Count comma10.;
    datalines;
<100       100 210
100<=200   100 180
200<=300   100 160
300<=400   100 112
400<=500   100 100
500<=600   100 60
600<=700   100 60
700<=800   100 60
800<=900   100 10
900+       100 10
;
run;
```

Calculate Relative Distributions and Population Stability Index (PSI) Metrics
```sas
proc sql;
    create table moderate_PSI2 as
    select PD_Score, 
           Base_Count, 
           Base_Count / sum(Base_Count) as Base_Perc format=percent10.2,
           Current_Count,
           Current_Count / sum(Current_Count) as Current_Perc format=percent10.2,
           (calculated Current_Perc - calculated Base_Perc) * log(calculated Current_Perc / calculated Base_Perc) as PSI format=7.4
    from moderate_PSI;

    create table significant_PSI2 as
    select PD_Score, 
           Base_Count, 
           Base_Count / sum(Base_Count) as Base_Perc format=percent10.2,
           Current_Count,
           Current_Count / sum(Current_Count) as Current_Perc format=percent10.2,
           (calculated Current_Perc - calculated Base_Perc) * log(calculated Current_Perc / calculated Base_Perc) as PSI format=7.4
    from significant_PSI;
quit;
```

Generate Model Stability Reports with Visual Risk Highlighting
```sas
title "Moderate PSI Model Stability Report";
proc report data=moderate_PSI2;
    column PD_Score Base_Count=BC_sum Base_Perc=BP_sum Current_Count=CC_sum Current_Perc=CP_sum PSI=PSI_sum;
    
    define PD_Score / display "PD Score";
    define BC_sum   / sum format=comma10. "Base #";
    define BP_sum   / sum format=percent10.2 "Base %";
    define CC_sum   / sum format=comma10. "Current #";
    define CP_sum   / sum format=percent10.2 "Current %";
    define PSI_sum  / sum format=7.4 "PSI" style(column)={background=rag_colors.};
    
    rbreak after / summarize style={font_weight=bold}; 
    
    compute after; 
        PD_Score = 'Total';
    endcomp;
run;

title "Significant PSI Model Stability Report";
proc report data=significant_PSI2;
    column PD_Score Base_Count=BC_sum Base_Perc=BP_sum Current_Count=CC_sum Current_Perc=CP_sum PSI=PSI_sum;
    
    define PD_Score / display "PD Score";
    define BC_sum   / sum format=comma10. "Base #";
    define BP_sum   / sum format=percent10.2 "Base %";
    define CC_sum   / sum format=comma10. "Current #";
    define CP_sum   / sum format=percent10.2 "Current %";
    define PSI_sum  / sum format=7.4 "PSI" style(column)={background=rag_colors.};
    
    rbreak after / summarize style={font_weight=bold}; 
    
    compute after; 
        PD_Score = 'Total';
    endcomp;
run;
title;
```




