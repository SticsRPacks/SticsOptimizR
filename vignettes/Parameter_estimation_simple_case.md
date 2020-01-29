


---
title: "Parameter estimation with the Stics crop Model: a simple case"
output:
  rmarkdown::html_vignette
author:
- name: "Samuel Buis"
affiliation: "INRA - EMMAH"
date: "2020-01-29"
vignette: >
  %\VignetteIndexEntry{Vignette Title}
  %\VignetteEngine{knitr::rmarkdown}
  %\VignetteEncoding{UTF-8}
params:
  eval_rmd: FALSE
---





## Study Case

This document presents an example of a simple parameter estimation using the Stics model with a single situation, a single observed variable and 2 estimated parameters, just to illustrate how to use the package.
A more complex example with simultaneous estimation of specific and varietal plant parameters from a multi-varietal dataset is presented in another [vignette](https://SticsRPacks.github.io/CroptimizR/articles/Parameter_estimation_Specific_and_Varietal.html).

**Please note that parameter estimation on intercrops and chained situations (e.g. rotations) are not possible for the moment with the Stics model. It will be provided in next versions.**

Data comes from a maize crop experiment (see description in Wallach et al., 2011).

The parameter estimation is performed using the Nelder-Meade simplex method implemented in the nloptr package.

## Initialisation step



```r
# Install and load the needed libraries
devtools::install_github("SticsRPacks/SticsRPacks")
library("SticsRPacks")

# Download the example USMs:
data_dir= file.path(SticsRFiles::download_data(),"study_case_1","V9.0")
# NB: all examples are now in data_dir

# DEFINE THE PATH TO YOUR LOCALLY INSTALLED VERSION OF JAVASTICS
javastics_path=file.path(path_to_JavaStics,"JavaSTICS-1.41-stics-9.0")
stics_path=file.path(javastics_path,"bin/stics_modulo.exe")
```


## Generate Stics input files from JavaStics input files

The Stics wrapper function used in CroptimizR works on text formatted input files (new_travail.usm,
climat.txt, ...) stored per USMs in different directories (which names must be the USM names).
`stics_inputs_path` is here the path of the directory that will contain these USMs folders.

If you start from xml formatted input files (JavaStics format: usms.xml, sols.xml, ...)
the following lines allow generating txt files from xml files.
In this case, xml files are stored in file.path(data_dir,"XmlFiles").



```r
stics_inputs_path=file.path(data_dir,"TxtFiles")
dir.create(stics_inputs_path)

gen_usms_xml2txt(javastics_path = javastics_path, workspace_path = file.path(data_dir,"XmlFiles"),
  target_path = stics_inputs_path, display = TRUE)
```



## Run the model before optimization for a prior evaluation

Here model parameters values are read in the model input files.



```r
# Set the model options (see '? stics_wrapper_options' for details)
model_options=stics_wrapper_options(stics_path,stics_inputs_path,
                                    parallel=FALSE)

# Run the model on all situations found in stics_inputs_path
sim_before_optim=stics_wrapper(model_options=model_options)
```


## Read and select the observations for the parameter estimation

For Stics, observation files must for the moment have exactly the same names as the corresponding USMs and be stored in a unique folder to be read by the get_obs function. This will be improved in next versions.

In this example, we only keep observations for situation (i.e. USM for Stics) `sit_name` and variable `var_name`.

**`obs_list` defines the list of situations, variables and dates that will be used to estimate the parameters.**

In variables and parameters names, "(\*)" must be replaced by "_\*" to be handled by R (e.g. lai(n) is denoted here lai_n).



```r
sit_name="bo96iN+"  ## among bo96iN+, bou00t1, bou00t3, bou99t1, bou99t3,
                    ## lu96iN+, lu96iN6 or lu97iN+

obs_list=get_obs(file.path(data_dir,"XmlFiles"),
                          obs_filenames = paste0(sit_name,".obs"))

var_name="lai_n"    ## lai_n or masec_n
obs_list[[sit_name]]=obs_list[[sit_name]][,c("Date",var_name)]
```


## Set information on the parameters to estimate

**`param_info` determines the list of parameters that will be estimated in the parameter estimation process from the situations, variables and dates defined in `obs_list`**, and associated bounds (-Inf and Inf can be used in bounds list).

All the numerical parameters which values can be provided to the model through its R wrapper can be estimated using the provided parameter estimation methods (although it may not work well for integer parameters).



```r
# 2 parameters here: dlaimax and durvieF, of bounds [0.0005,0.0025] and [50,400].
param_info=list(lb=c(dlaimax=0.0005, durvieF=50),
                       ub=c(dlaimax=0.0025, durvieF=400))
```


## Set options for the parameter estimation method

`optim_options` must contain the options of the parameter estimation method.
Here we defined a few options for the simplex method of the nloptr package (default method in estim_param).
The full set of options for the simplex method can be found in the [vignette of nloptr package](https://cran.r-project.org/web/packages/nloptr/vignettes/nloptr.pdf).

The number of repetitions is advised to be set to at least 5, while 10 is a reasonable maximum value.
`maxeval` should be used to stop the minimization only if results have to be produced within a given duration, otherwise set it to a high value so that the minimization stops when the criterion based on `xtol_rel` is satisfied.



```r
optim_options=list()
optim_options$nb_rep <- 7 # Number of repetitions of the minimization
                          # (each time starting with different initial
                          # values for the estimated parameters)
optim_options$maxeval <- 500 # Maximum number of evaluations of the
                             # minimized criteria
optim_options$xtol_rel <- 1e-03 # Tolerance criterion between two iterations
                                # (threshold for the relative difference of
                                # parameter values between the 2 previous
                                # iterations)
optim_options$path_results <- data_dir # path where to store the results (graph and Rdata)
optim_options$ranseed <- 1234 # set random seed so that each execution give the same results
                              # If you want randomization, don't set it.
```


## Run the optimization

The Nelder-Meade simplex is the default method => no need to set the
optim_method argument. For the moment it is the only method interfaced (others will come soon).
Same for crit_function: a value is set by default (`crit_cwss`, see `? crit_cwss` for more details and list of available criteria).



```r
optim_results=estim_param(obs_list=obs_list,
                            model_function=stics_wrapper,
                            model_options=model_options,
                            optim_options=optim_options,
                            param_info=param_info)
```


The results printed in output on the R console are the following:


```r
## ## [1] "Estimated value for dlaimax :  0.00169614928696274"
## ## [1] "Estimated value for durvieF :  53.9691276907021"
## ## [1] "Minimum value of the criterion : 112.530331140718"
```


Complementary graphs and data are stored in the `optim_options$path_results` folder. Among them, the EstimatedVSinit.pdf file containing the following figures:


<img src="ResultsSimpleCase/estimInit_dlaimax.PNG" title="plot of chunk unnamed-chunk-7" alt="plot of chunk unnamed-chunk-7" width="45%" /><img src="ResultsSimpleCase/estimInit_durvieF.PNG" title="plot of chunk unnamed-chunk-7" alt="plot of chunk unnamed-chunk-7" width="45%" />


Figure 1: plots of estimated vs initial values of parameters dlaimax and durvieF. Numbers represent the repetition number of the minimization. The number in red, 2 in this case, is the minimization that lead to the minimal value of the criterion among all repetitions. In this case, the minimizations converge towards 2 different values for the parameters, which indicates the presence of a local minimum. Values of durvieF are close to the bounds. In realistic calibration cases with many observed situations / variables / dates this may indicate the presence of biases in the observation values or in the model output values simulated (this simple case with only one situation does not allow to derive such conclusion).

The `optim_options$path_results` folder also contains the optim_results.Rdata file that store the `nlo` variable, a list containing the results of the minimization for each repetition. If we print it for repetition 2 ...


```r
## load(file.path(optim_options$path_results,"optim_results.Rdata"))
## nlo[[2]]
```

... this returns:


```
## 
## Call:
## nloptr(x0 = init_values[irep, ], eval_f = main_crit, lb = bounds$lb,     ub = bounds$ub, opts = list(algorithm = "NLOPT_LN_NELDERMEAD", 
##         xtol_rel = xtol_rel, maxeval = maxeval, ranseed = ranseed),     crit_options = crit_options_loc)
## 
## 
## Minimization using NLopt version 2.4.2 
## 
## NLopt solver status: 4 ( NLOPT_XTOL_REACHED: Optimization stopped because xtol_rel or xtol_abs (above) was reached. )
## 
## Number of Iterations....: 44 
## Termination conditions:  xtol_rel: 0.001	maxeval: 500 
## Number of inequality constraints:  0 
## Number of equality constraints:    0 
## Optimal value of objective function:  112.530331140718 
## Optimal value of controls: 0.001696149 53.96913
```



## Run the model after optimization

We run here the Stics model using the estimated values of the parameters.
In this case, the `param_values` argument of the stics_wrapper function is thus set so that estimated values of the parameters overwrite the values defined in the model input files.



```r
sim_after_optim=stics_wrapper(param_values=optim_results$final_values,
                              model_options=model_options)
```


## Plot the results




```r
png(file.path(optim_options$path_results,"sim_obs_plots.png"),
    width = 15, height = 10, units = "cm", res=1000)
par(mfrow = c(1,2))

# Simulated and observed LAI before optimization
Ymax=max(max(obs_list[[sit_name]][,var_name], na.rm=TRUE),
         max(sim_before_optim$sim_list[[sit_name]][,var_name], na.rm=TRUE))
plot(sim_before_optim$sim_list[[sit_name]][,c("Date",var_name)],type="l",
     main="Before optimization",ylim=c(0,Ymax+Ymax*0.1))
points(obs_list[[sit_name]],col="green")

# Simulated and observed LAI after optimization
plot(sim_after_optim$sim_list[[sit_name]][,c("Date",var_name)],type="l",
     main="After optimization",ylim=c(0,Ymax+Ymax*0.1))
points(obs_list[[sit_name]],col="green")

dev.off()
```


This gives:


<img src="ResultsSimpleCase/sim_obs_plots.png" title="Figure 2: plots of simulated and observed target variable before and after optimization. The gap between simulated and observed values has been drastically reduced: the minimizer has done its job!" alt="Figure 2: plots of simulated and observed target variable before and after optimization. The gap between simulated and observed values has been drastically reduced: the minimizer has done its job!" width="80%" />

