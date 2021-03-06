# 1-D Monte Carlo simulation of microbial exposure
Jane Pouzou and Brian High  
![CC BY-SA 4.0](cc_by-sa_4.png)  

## Introduction

This document offers a 1-D 
[Monte Carlo](https://en.wikipedia.org/wiki/Monte_Carlo_method) probabilistic 
solution in R for the daily microbial exposure from drinking water consumption, 
swimming in surface water and shellfish consumption for 
[Example 6.18](images/ex0618.png) from pages 215-216 of:

[Quantitative Microbial Risk Assessment, 2nd Edition](http://www.wiley.com/WileyCDA/WileyTitle/productCd-1118145291,subjectCd-CH20.html) 
by Charles N. Haas, Joan B. Rose, and Charles P. Gerba. (Wiley, 2014).

## Set global options


```r
# Set knitr options for use when rendering this document.
library("knitr")
opts_chunk$set(cache=TRUE, message=FALSE)
```

## Define variables


```r
# Define variables provided in the example for three exposure types.

# Shellfish consumption
shellfish.viral.load <- 1
shellfish.cons.g <- 9e-4 * 150  # 9e-4 days/year * 150 g/day

# Drinking water consumption
dw.viral.load <- 0.001

# Surface water consumption while swimming
sw.viral.load <- 0.1
sw.daily.IR <- 50               # Ingestion rate in mL of surface water
sw.frequency <- 7               # Exposure frequency of 7 swims per year
```

## Sample from probability distributions


```r
# Generate 5000 random values from a log-normal distribution to estimate 
# exposure from consumption of drinking water (ml/day). Divide by 1000 
# mL/L to get consumption in liters/day.  Values for meanlog and sdlog 
# are from the QMRA textbook (Haas, 2014), page 216, Table 6.30.
set.seed(1)
water.cons.L <- rlnorm(5000, meanlog = 7.49, sdlog = 0.407) / 1000

# Plot the kernal density curve of the generated values just as a check.
plot(density(water.cons.L))
```

![](ex0618prob_files/figure-html/unnamed-chunk-3-1.png)

```r
# Sample 5000 times from a discrete distribution of swim duration with 
# assigned probabilities of each outcome. These values are hypothetical 
# and are not found in the text, but are defined here to provide an 
# example of sampling from a discrete distribution.
set.seed(1)
swim.duration <- sample(x = c(0.5, 1, 2, 2.6), 5000, replace = TRUE, 
                        prob = c(0.1, 0.1, 0.2, 0.6))

# Create a simple histogram of our distribution as a check.
hist(swim.duration)
```

![](ex0618prob_files/figure-html/unnamed-chunk-3-2.png)

## Estimate daily dose

Calculate estimated daily dose using a probabilistic simulation model.


```r
# Define a function to calculate microbial exposure risk.
Risk.fcn <- function(shellfish.vl, shellfish.cons.g, water.cons.L, dw.vl, sw.vl, 
                     sw.daily.IR, sw.duration, sw.frequency) {
    ((shellfish.vl * shellfish.cons.g) + (water.cons.L * dw.vl) + 
         ((sw.vl * (sw.daily.IR * sw.duration * sw.frequency)) / 365 / 1000))
}

# Compute 5000 simulated daily dose results and store as a vector.
daily.dose <- sapply(1:5000, 
                     function(i) Risk.fcn(water.cons.L = water.cons.L[i], 
                                          sw.duration = swim.duration[i], 
                                          shellfish.vl = shellfish.viral.load, 
                                          dw.vl = dw.viral.load, 
                                          shellfish.cons.g = shellfish.cons.g, 
                                          sw.vl = sw.viral.load, 
                                          sw.daily.IR = sw.daily.IR, 
                                          sw.frequency = sw.frequency))
```

## Summarize results


```r
# Set display options for use with the print() function.
options(digits=3)

# Print the geometric mean of the vector of simulated daily dose results.
print(format(exp(mean(log(daily.dose))), scientific = TRUE))
```

```
## [1] "1.37e-01"
```

### Calculate kernel density estimates


```r
# Calculate kernel density estimates.
dens <- density(daily.dose)
dens
```

```
## 
## Call:
## 	density.default(x = daily.dose)
## 
## Data: daily.dose (5000 obs.);	Bandwidth 'bw' = 0.0001264
## 
##        x               y      
##  Min.   :0.135   Min.   :  0  
##  1st Qu.:0.137   1st Qu.:  1  
##  Median :0.140   Median : 11  
##  Mean   :0.140   Mean   :112  
##  3rd Qu.:0.142   3rd Qu.:158  
##  Max.   :0.144   Max.   :577
```

### Calculate measures of central tendency


```r
# Calculate measures of central tendency.
meas <- data.frame(
    measure = c("mean", "g. mean", "median", "mode"),
    value = round(c(
        mean(daily.dose), exp(mean(log(daily.dose))),
        median(daily.dose), dens$x[which.max(dens$y)]
    ), 6),
    color = c("red", "orange", "green", "blue"),
    stringsAsFactors = FALSE
)

# Set display options for use with the print() function.
options(digits=6)

# Print measures of central tendency.
print(meas[1:2])
```

```
##   measure    value
## 1    mean 0.137151
## 2 g. mean 0.137149
## 3  median 0.136990
## 4    mode 0.136810
```

### Plot the kernel density estimates with measures of central tendency


```r
# Contruct text labels by combining each measure with its value.
meas$label <- sapply(1:nrow(meas), function(x) 
    paste(meas$measure[x], as.character(meas$value[x]), sep = ' = '))

# Add lines for measures of central tendency and a legend to a plot.
add_lines_and_legend <- function(meas, x.pos = 0, y.pos = 0, cex = 1) {
    n <- nrow(meas)
    
    # Plot measures of central tendency as vertical lines.
    res <- sapply(1:n, function(x) 
        abline(v = meas$value[x], col = meas$color[x]))
    
    # Add a legend to the plot.
    legend(x.pos, y.pos, meas$label, col = meas$color, 
           cex = cex, lty = rep(1, n), lwd = rep(2, n))
}

# Plot the kernel density estimates.
plot(dens)

# Add lines for measures of central tendency and a legend.
add_lines_and_legend(meas, 0.139, 550)
```

![](ex0618prob_files/figure-html/unnamed-chunk-8-1.png)

### Plot the empirical cumulative distribution


```r
# Plot the empirical cumulative distribution for the exposure estimates.
plot(ecdf(daily.dose))

# Add lines for measures of central tendency and a legend.
add_lines_and_legend(meas, 0.139, 0.8)
```

![](ex0618prob_files/figure-html/unnamed-chunk-9-1.png)
