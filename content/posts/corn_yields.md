---
title: "Corn Yields"
name: corn-yields
theme: Toha
date: 2022-03-26
draft: false
hero: images/posts/corn_yields/blog_harvest.jpeg
menu: 
    sidebar:
        name: Corn Yields
math: true
---

## Summary

I wanted to take a look at corn yields and how they were effected by
variables in data that I could find. Luckily the
[USDA](https://quickstats.nass.usda.gov/api) has a relatively extensive
dataset on crop production and yields by state and also includes
information such as if the crops were irrigated or not. They have a
query tool which I used to get the data located in `data/USDA/corn.csv`
which I reference below and is available in the repo. They also have a
query tool that I may attempt to make more use of in the future. With
the data given I wanted to understand how the corn yields have changed
over time, and given the information I was able to find, what variables
were important factors for corn yields (what area and if the land was
irrigated or not).

I will also mention that analyzing this data was a good way for me to
practice methods that I learned in my regression analysis, so as you see
a strong focus on linear models in this analysis.

## Data Loading

First I load the data and filter such that we are looking for entries
related to irrigated (or non-irrigated) grain corn.

Next we do some reprocessing such as separating out the description to
get the data we need on if it was irrigated or not, and what the measure
was. Then we pivot the data so it is easier to do some calculations with
the area and production volumes (bushels).

Next we will plot the production and yield.

![image](/posts/corn_yields/unnamed-chunk-5-2.png) 

From
the two plots above it is amazing to see how the grain production and
yields have improved over time! Because there are quite a few states, I
will group some states which I think more alike (places like Wyoming,
Montana and Idaho vs. Nebraska and Kansas). It is likely that there
could be other methods used to find groupings which maximize the
differences between each group of states; this was not used in this
analysis and I will move forward with my intuition which as you will see
will be quite subjective). You will also see that I move forward with
using yields (bu/acre) so that we can get a measure of how efficiently
we are able to produce corn for a given unit of area.

## Data Aggregation

![](/posts/corn_yields/unnamed-chunk-6-1.png) \###
Visual analysis Now that we have things grouped, we should note how
similar the areas are for yields where the land is irrigated after 1980
while there are some differences prior to 1980. It is also interesting
to note how different each areas non-irrigated acreage performs (my
guess would be due to rainfall amounts and potentially temperatures).
Also note how the yields for the non-irrigated acreage appears
significantly lower than that of the irrigated acreage, this should not
be surprising.

Let’s use a linear regression model to see if we can parse out the
effects of the area, irrigation, and time.

**Confounders!** Something to be aware of as we move forward is that
time, or specifically, the year that the corn was planted and harvested
appears to have a large effect on the yields. There is no physical
reason why corn with all the same treatments (irrigation, fertilization,
pest control, etc.) would yield more grain in 2020 in comparison to
1980. I say this because there are many confounding variables that are
not present in this data set such as fertilization, pest control, crop
rotation practices which have increase yields over time as we have
continually discovered better farming methods to increase the yields of
our crops. This presents a spurious relationship between the year that
the crop was planted and harvested and the observed yeilds.

## Regression Modeling (OLS)

First a linear regression model with the ordinary least squares (OLS)
method will be fit. In this case our model will look like

$$
Yield = \beta_0 + \beta_1 Area + \beta_1 Irrigated + \beta_1 Year + \epsilon
$$

Where the expectation of \\(\epsilon\\) is 0 (\\(E[\epsilon]=0\\)) and the variance of
\\(\epsilon\\) is \\(\sigma^2\\) (\\(Var(\epsilon)=\sigma^2\\)).


Something to notice is that we have not stated anything about the
distribution of any of the parameters in the model. Two key assumptions
that we currently have and will have to verify is that our samples are
independent \\(corr(\epsilon_i, \epsilon_j)=0\\) where \\(i\not=j\\) and the variance is constant.

With that said you will see summaries of the model which will include
plenty of t-tests and F-tests on the model and coefficients as well as
confidence intervals. For these to be valid we need to assume (and
verify) that our residuals are normally distributed such that \\(\epsilon \sim N(0,\sigma^2)\\).

### Additive Model
![](/posts/corn_yields/unnamed-chunk-7-1.png)![](/posts/corn_yields/unnamed-chunk-7-2.png)![](/posts/corn_yields/unnamed-chunk-7-3.png)![](/posts/corn_yields/unnamed-chunk-7-4.png)


![](/posts/corn_yields/unnamed-chunk-7-5.png) From
the plots above we it appears the model does a reasonable job of
predicting the yield, however the residual plots show that there
certainly is room for improvement. We have some trends in the mean of
the residuals vs. the predicted values and the QQ plot shows that they
are not normally distributed around the mean. There may also be some
issues with the assumption around constant variance, and if we look
closely at the plots above showing the production and yields over time
we may suspect that the residuals are correlated and not independent.
This means we’re breaking a lot, if not all, of our assumptions. Lets
see if we can get in a little better position by allowing the
categorical variables to have a multiplicative influence instead (so
they can change the slope of the line) instead of having only an
additive influence which allows them to only change the intercept.

### Multiplicative model

Luckily R makes it easy to add the multiplicative influence by asking R
to fit a model with all of the interactions as shown below. As mentioned
above, this allows for the categorical values to influence not only the
intercept but also the slope of yield over time.

![](/posts/corn_yields/unnamed-chunk-8-1.png)![](/posts/corn_yields/unnamed-chunk-8-2.png)![](/posts/corn_yields/unnamed-chunk-8-3.png)![](/posts/corn_yields/unnamed-chunk-8-4.png)

![](/posts/corn_yields/unnamed-chunk-8-5.png)
Things area starting to look a little bit better, we don’t have quite as
bad of a trend in the mean of our residuals and our QQ-plot while not
great is looking like we’re getting a little closer to normally
distributed. With that said we should look into the potential for
autocorrelation for our groups.

We will look at the autocorrelation by grouping our data then using the
`acf()` function to look for correlation over time.

![](/posts/corn_yields/unnamed-chunk-9-1.png)

![](/posts/corn_yields/unnamed-chunk-9-2.png)

![](/posts/corn_yields/unnamed-chunk-9-3.png)

![](/posts/corn_yields/unnamed-chunk-9-4.png)

As suspected, it looks like we do have some pretty serious
auto-correlation issues to deal with; these most definitely cannot be
ignored. In order to address this I will use of Generalized Least
Squares (GLS).

## GLS

For the linear regression model above we assumed that \\(\epsilon \sim N(0, \sigma^2) \\), although in reality we have been working with a multivariate normal distribution where \\(\epsilon \sim N(0, \sigma^2) \\) where \\(I\\) is the identity matrix.
Writing it in this way and examining the covariance matrix
\\(\sigma^2I \\) makes it pretty easy to see the two assumptions
(constant variance and no correlation). Below is a an example of what
the matrix would look like:
$$
\\begin{bmatrix}
\\sigma^2 & 0  & \\ldots & 0 \\\\
0 & \\sigma^2  & \\ldots & 0 \\\\
\\vdots & \\vdots & \\vdots & \\vdots \\\\
0 & 0 & 0 & \\sigma^2
\\end{bmatrix}
$$
However in generalized least squares (GLS), there is no structure to the
covariance matrix specified, so we can write

\\(\epsilon \sim N(0, \Sigma) \\) where \\(\Sigma \\) is the covariance matrix. In practice this
give us far more parameters than we have observations (if we have n
observations then \\(\Sigma \\) is an n x n matrix), so it is very helpful to
assume some structure for the covariance matrix. For a problem such as
this we will assume that the residuals follow an *autoregressive process
of order 1*, in other words the residual for a group is correlated with
the previously sampled residual (assuming our sampling occurs on an
regular basis, yearly in this case). Now our covariance matrix will look
more like this:

$$
\\begin{bmatrix}
\\sigma^2 & \\rho \\sigma^2 & \\rho^2\\sigma^2 & \\ldots & \\rho^n \\sigma^2 \\\\
\\rho\\sigma^2 & \\sigma^2 & \\rho\\sigma^2 & \\ldots & \\rho^{n-1} \\sigma^2 \\\\
\\vdots & \\vdots & \\vdots & \\vdots & \\vdots \\\\
\\rho^n \\sigma^2 & \\rho^{n-1} \\sigma^2 & \\rho ^{n-2}\\sigma^2 & \\ldots & \\sigma^2 \\\\
\\end{bmatrix}
$$
Where |*ρ*| &lt; 1.

![](/posts/corn_yields/unnamed-chunk-10-1.png)

![](/posts/corn_yields/unnamed-chunk-10-2.png)

Now that we’ve used GLS to avoid breaking our assumptions we can see a
few things: \* From the ANOVA summary on the GLS model, we have strong
evidence to conclude that the variables are capturing a significant
amount of variance as shown by the very small p-values \* The 95%
confidence interval for `Phi1` (nlme’s correlation coefficient, what I
wrote as *ρ*) does not include 0, so we have strong evidence to conclude
that there indeed was autocorrelation present in our residuals.

Interpreting the model is a little more difficult, with the interaction
parameters, but we can see the -1.285 parameter for Year:NON-IRRIGATED
more than offsets the addition to the intercept for the non irrigated
acres in the time periods we’re interested in. For example even in 1950
(for a MTN\_WEST state) the effect of not having irrigation was -42
bu/acre (2463 - 1.285 \* 1950), whereas in 2020 it would have been -132
bu/acre. This is slightly offset in areas such as the MID\_WEST where
~30 bu/acre was added to the intercept for acreage without irrigation
(the EAST has even more, but take a look at that standard error!) \*
Time is a big component! For all of our regions the yield increases by
&gt; 2 bu/acre/year. As dicussed earlier this is likely due to other
counfounding variables not accounted for in this analysis.

As we saw before we can also see each area can have a significant impact
to the yields. The differences are not nearly as large as that due to
irrigation, but they are still significant.

# Conclusions

Above we have discussed linear regression, it’s main assumptions, and
analyzing the residuals to verify those assumptions hold true. We also
discussed generalized least squares and how we can use that method to
help address concerns identified from OLS which cannot be addressed
through other methods such as transformations. We also found that the
area that corn is being produced from has a significant factor on yields
as does irrigation. Time was also found to explain a lot of variance in
the model but we also covered the presence of confounding variables such
as technology and farming practices that have changed over time and
helped improve yields.
