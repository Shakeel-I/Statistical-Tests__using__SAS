# Statistical-Tests-using-SAS
Hosmer and Lemeshow Goodness-of-Fit Test

Brier score Test

KS

Calibration plot

ROC chart

# Hosmer and Lemeshow Goodness-of-Fit Test
The Hosmer-Lemeshow test evaluates the goodness-of-fit for logistic regression models by comparing the observed and expected frequencies of outcomes in groups of data. It helps determine whether the model adequately fits the data.

![Hosmer1](https://github.com/user-attachments/assets/69656266-22d6-405b-b825-4fba0585affc)
```
proc logistic data=Employment plots=none;
model Employed=Education Experience/gof lackfit;
run;
```
![Hosmer2](https://github.com/user-attachments/assets/fb10f6e1-e977-4a29-b036-8c524b75fded)


# Brier score Test

KS

Calibration plot

ROC chart
