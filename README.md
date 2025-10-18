# STATISTICAL TESTS USING SAS SQL

# 1. Hosmer and Lemeshow Goodness-of-Fit Test
The Hosmer-Lemeshow test evaluates the goodness-of-fit for logistic regression models by comparing the observed and expected frequencies of outcomes in groups of data. It helps determine whether the model adequately fits the data.

H₀ (null hypothesis): The model fits the data well; there is no significant difference between observed outcomes and those predicted by the model across the groups.

![Hosmer1](https://github.com/user-attachments/assets/69656266-22d6-405b-b825-4fba0585affc)
```
proc logistic data=Employment plots=none;
model Employed=Education Experience/gof lackfit;
run;
```
![Hosmer2](https://github.com/user-attachments/assets/fb10f6e1-e977-4a29-b036-8c524b75fded)

Since the p-value from the test is above 0.05, we fail to reject H₀, suggesting our logistic regression model has a good fit to the data.




# 2. Brier score Test
Brier score is an important measure of calibration. It evaluates the accuracy of probabilistic predictions by measuring the mean squared difference between predicted probabilities and actual outcomes (0 or 1). It assesses how well the predicted probabilities are calibrated, with lower scores indicating more accurate and well-calibrated predictions.
- Brier Score is 0, the best score achievable.
- The model below gives a Brier score of 0.08 which is quite good.

```
%let seed = 12345;

/* Create column 'random' to assign random number between 0 and 1 */
data partitioned;
	set mydata.hmeq;
	random=ranuni(&seed);
run;

/* Partition into training_set 80% and validation_set 20% */
data training_set validation_set;
	set partitioned;
	if random < 0.8 then
		output training_set;
	else
		output validation_set;
run;

/* Drop column 'random' */
data training_set;
	set training_set;
	drop random;
run;

/* Drop column 'random' */
data validation_set;
	set validation_set;
	drop random;
run;

/* Train model on training_set then SCORE model on validation_set */
proc logistic data=training_set desc;
	class job reason /param=glm;
	model bad=debtinc delinq loan reason;
	score data=validation_set out=predicted_probabilities fitstat;
run;
```
![Brier](https://github.com/user-attachments/assets/5b22ddd1-7e74-4d8e-bf51-42a0ac7f19fe)


# 3. Kolmogorov Smirnov Test
The Kolmogorov-Smirnov (K-S) chart is used to evaluate the discrimination ability of a credit scoring model by measuring how well the model distinguishes between good and bad borrowers. It plots the cumulative distribution of good and bad borrowers across the score spectrum.

![KS1](https://github.com/user-attachments/assets/35c8eeb7-44d8-4d77-9c71-a9abf87c1c27)

```
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
![KS2](https://github.com/user-attachments/assets/42cbfebc-b946-4f3e-b50e-e9144b41e68c)

The K-S statistic, represented by the maximum difference between these two cumulative curves, indicates the model's effectiveness: a higher K-S value suggests better discrimination between good and bad credits.

# 4. Calibration plot
A calibration plot helps assess how well predicted probabilities from a model match the actual observed outcomes. It plots mean predicted probabilities on the x-axis versus the fraction of positives (observed event rate) on the y-axis.

```
/* Step 2: Fit logistic regression model and output predicted probs */
proc logistic data=myhmeq noprint;
  model bad(event='1') = debtinc;
  output out=predout predicted=phat;
run;

/* Step 3: Bin predicted probabilities into deciles */
proc rank data=predout groups=10 out=binned;
  var phat;
  ranks phat_bin;
run;

/* Step 4: Calculate mean predicted (phat) and mean observed (BAD) outcome per bin */
proc means data=binned noprint;
  class phat_bin;
  var phat bad;
  output out=cal_data mean=mean_phat mean_bad;
run;

/* Step 5: Plot calibration curve */
proc sgplot data=cal_data(where=(_type_=1));
  scatter x=mean_phat y=mean_bad / markerattrs=(symbol=CircleFilled) name='pts';
  lineparm x=0 y=0 slope=1 / lineattrs=(color=gray pattern=shortdash) name='ref';
  xaxis label='Mean Predicted Probability';
  yaxis label='Observed Default Rate';
  title 'Calibration Plot: Predicting Loan Default (BAD) from DEBTINC';
run;
```

![Calib1](https://github.com/user-attachments/assets/4499247c-2ad9-4601-8ae1-7bc27bde9bb4)

If the points lie close to the diagonal line, the model is well calibrated, meaning its predicted probabilities accurately reflect real-world risks. Deviations from this line reveal if the model tends to over- or under-predict outcomes, making calibration plots a straightforward and visual tool to evaluate the reliability of probabilistic predictions. This helps ensure predictions can be trusted for decision making.



# 5. ROC chart
The ROC (Receiver Operating Characteristic) curve visually shows the trade-off between sensitivity (true positive rate) and 1-specificity (false positive rate) across different threshold levels of a classification model. It illustrates how well the model can distinguish between positive and negative cases at various thresholds, with the area under the curve (AUC) indicating overall discriminative ability.

```
proc logistic data=Employment plots(only)=roc;
model Employed=Education Experience;
run;
```

![ROC1](https://github.com/user-attachments/assets/8bd055e8-f7f1-44a2-a073-eb899757d4ed)

An AUC of 0.74 in this context means that there is a 74% chance that the model will correctly distinguish between an employed and a not-employed individual. This indicates good discriminatory power, suggesting the model is fairly effective at predicting employment status, though there's still some room for improvement.


