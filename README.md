# STATISTICAL TESTS USING SAS

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


# 3. KS


# 4. Calibration plot


# 5. ROC chart
The ROC (Receiver Operating Characteristic) curve visually shows the trade-off between sensitivity (true positive rate) and 1-specificity (false positive rate) across different threshold levels of a classification model. It illustrates how well the model can distinguish between positive and negative cases at various thresholds, with the area under the curve (AUC) indicating overall discriminative ability.

```
proc logistic data=Employment plots(only)=roc;
model Employed=Education Experience;
run;
```

![ROC1](https://github.com/user-attachments/assets/8bd055e8-f7f1-44a2-a073-eb899757d4ed)




