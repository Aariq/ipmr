[![R build
status](https://github.com/levisc8/ipmr/workflows/R-CMD-check/badge.svg)](https://github.com/levisc8/ipmr/actions)
[![Vignette
Code](https://github.com/levisc8/ipmr/workflows/check-vignettes/badge.svg)](https://github.com/levisc8/ipmr/actions)
[![Lifecycle:
maturing](https://img.shields.io/badge/lifecycle-maturing-blue.svg)](https://www.tidyverse.org/lifecycle/#maturing)
[![CRAN
status](https://www.r-pkg.org/badges/version/ipmr)](https://cran.r-project.org/package=ipmr)
[![Codecov test
coverage](https://codecov.io/gh/levisc8/ipmr/branch/main/graph/badge.svg)](https://codecov.io/gh/levisc8/ipmr?branch=main)
[![CRAN
Downloads](https://cranlogs.r-pkg.org/badges/ipmr)](https://cran.r-project.org/package=ipmr)
[![DOI](https://zenodo.org/badge/DOI/10.5281/zenodo.5076190.svg)](https://doi.org/10.5281/zenodo.5076190)
[![DOI](https://zenodo.org/badge/DOI/10.1111/2041-210X.13683.svg)](https://doi.org/10.1111/2041-210X.13683)

# ipmr

`ipmr` is a package for implementing Integral Projection Models (IPMs)
in *R*. It relies heavily on the mathematical syntax of the models, and
does not try to abstract over the process of fitting vital rates. It is
now relatively stable, though tweaks may be made. Below is a brief
overview of how `ipmr` classifies different model types followed by
examples of how to implement those types in this framework.

## Installation

`ipmr` is available on CRAN and can be installed with the following
snippet:

``` r
install.packages("ipmr")
```

You can also install the development version from GitHub with the
snippet below:

``` r
if(!require('remotes', quietly = TRUE)) {
  install.packages("remotes")
}

remotes::install_github("levisc8/ipmr", build_vignettes = TRUE)
```

## Package scope

`ipmr` is intended to assist with IPM implementation and analysis. It is
important to note that this package **will not** help with the process
of fitting regression models for vital rates at all! That is a
sufficiently different (and vast) topic that we decided it was not
within the scope of this project. This will only help you turn those
regression models into an IPM without shooting yourself in the foot.
Furthermore, most of the documentation assumes a basic knowledge of IPM
theory and construction. For those that are totally new to IPMs, it is
strongly recommended to read a theoretical overview of the models first.
Some favorites are [Easterling et
al. 2000](https://esajournals.onlinelibrary.wiley.com/doi/abs/10.1890/0012-9658%282000%29081%5B0694%3ASSSAAN%5D2.0.CO%3B2),
[Merow et al. 2013](https://doi.org/10.1111/2041-210X.12146), and [Rees
et al. 2014](https://doi.org/10.1111/1365-2656.12178). The [Introduction
to ipmr
vignette](https://levisc8.github.io/ipmr/articles/ipmr-introduction.html)
also contains a brief overview of IPM theory as well as a far more
detailed introduction to this package. Thus, everything that follows
assumes you have basic understanding of IPM theory, parameterized vital
rate models, and are now ready to begin implementing your IPM.

Below is a brief overview of the package and some examples of how to
implement models with it. A more thorough introduction is available
[here](https://levisc8.github.io/ipmr/articles/ipmr-introduction.html).

## Model classes

Once all parameters are estimated, the first step of defining a model in
`ipmr` is to initialize the model using `init_ipm()`. This function has
five arguments: `sim_gen`, `di_dd`, `det_stoch`, `kern_param`, and
`uses_age`. We will ignore `uses_age` for now, because age-size models
are less common and have their [own
vignette](https://levisc8.github.io/ipmr/articles/age_x_size.html).

The combination of these arguments defines the type of projection model,
and makes sure that the machinery for subsequent analyses works
correctly. The possible entries for each argument are as follows:

-   `sim_gen`: `"simple"`/`"general"`

    -   A. **simple**: This describes an IPM with a single continuous
        state variable and no discrete stages.

    -   B. **general**: This describes and IPM with either more than one
        continuous state variable, one or more discrete stages, or both
        of the above. Basically, anything other than an IPM with a
        single continuous state variable.

-   `di_dd`: `"di"`/`"dd"`

    -   A. **di**: This is used to denote a **d**ensity-**i**ndependent
        IPM.

    -   B. **dd**: This is used to denote a **d**ensity-**d**ependent
        IPM.

-   `det_stoch`: `"det"`/`"stoch"`

    -   A. **det**: This is used to denote a deterministic IPM. If this
        is the third argument of `init_ipm`, `kern_param` must be left
        as `NULL`.

    -   B. **stoch**: This is used to denote a stochastic IPM. If this
        is the third argument of `init_ipm`, `kern_param` must be
        specified. The two possibilities for the fourth are described
        next.

-   `kern_param`: `"kern"`/`"param"` (Complete definitions found in
    [Metcalf et
    al. 2015](https://besjournals.onlinelibrary.wiley.com/doi/10.1111/2041-210X.12405))

    -   A. **kern**: This describes an IPM with discretely varying
        parameters such that their values are known before the model is
        specified. This is usually the case with models that estimate
        fixed and/or random year/site effects and for which defining a
        multivariate joint distribution to sample parameters from is not
        desirable/needed. These models can be a bit more computationally
        efficient than the `param` alternative because all kernels can
        be constructed before the iteration procedure begins, as opposed
        to requiring reconstruction for every single iteration.

    -   B. **param**: This describes an IPM with parameters that are
        re-sampled from some distribution at each iteration of the
        model. This could be a multivariate normal defined by covarying
        slopes and intercepts, or distributions of environmental
        variables that change from time to time. All that is required is
        that the parameters for the distribution are specified and that
        the function that generates the parameters at each iteration
        returns named lists that correspond to the parameter names in
        the model. Examples of this are available in the Introduction
        and General IPM vignettes.

The following possibilities are currently or will become available in
`ipmr` (bold text denotes development progress):

-   Simple, density independent models: **Completed and ready**

1.  `"simple_di_det"`

2.  `"simple_di_stoch_kern"`

3.  `"simple_di_stoch_param"`

-   Simple, density dependent models: **Completed, likely stable**

1.  `"simple_dd_det"`

2.  `"simple_dd_stoch_kern"`

3.  `"simple_dd_stoch_param"`

-   General, density independent models: **Completed and ready**

1.  `"general_di_det"`

2.  `"general_di_stoch_kern"`

3.  `"general_di_stoch_param"`

-   General, density dependent models: **Completed, likely stable**

1.  `"general_dd_det"`

2.  `"general_dd_stoch_kern"`

3.  `"general_dd_stoch_param"`

Simple density-independent deterministic, simple kernel-resampled
stochastic, and simple parameter resampled stochastic models
(`simple_di_det`, `simple_di_stoch_kern`, `simple_di_stoch_param`) are
described in detail
[here](https://levisc8.github.io/ipmr/articles/ipmr-introduction.html).
The `general_*` versions of these are described
[here](https://levisc8.github.io/ipmr/articles/general-ipms.html).
Density dependent versions are completed for simple and general models,
and are probably stable, but have not been tested enough to be certain.
A very brief, though incomplete introduction is available
[here](https://levisc8.github.io/ipmr/articles/density-dependence.html).
Below is an example implementing a `simple_di_det` IPM.

## Quick example of a simple, deterministic IPM

Here is a simple model implemented with `ipmr`. This is a hypothetical
plant species where plants can survive and grow (*P*(*z*′,*z*)), and
reproduce sexually (*F*(*z*′,*z*)). We’ll use 4 regressions: survival
(*s*(*z*)), growth (*G*(*z*′,*z*), *f*<sub>*g*</sub>), probability of
reproducing (*r*<sub>*r*</sub>(*z*)), and number of seeds produced
conditional on flowering (*r*<sub>*s*</sub>(*z*)). New recruits will be
generated with a Gaussian distribution (*f*\_*r*<sub>*d*</sub>), which
requires calculating the mean and standard deviation of new recruits
from the data. For simplicity, we’ll assume there’s no maternal effect
on recruit size. First, we’ll write out the functional forms for each
component of the model:

1.  *n*(*z*′,*t*+1) = ∫<sub>*L*</sub><sup>*U*</sup>*K*(*z*′,*z*)*n*(*z*,*t*)*d**z*

2.  *K*(*z*′,*z*) = *P*(*z*′,*z*) + *F*(*z*′,*z*)

3.  *P*(*z*′,*z*) = *s*(*z*) \* *G*(*z*′,*z*)

4.  *L**o**g**i**t*(*s*(*z*)) = *α*<sub>*s*</sub> + *β*<sub>*s*</sub> \* *z*

5.  *G*(*z*′,*z*) = *f*<sub>*g*</sub>(*μ*<sub>*g*</sub>,*σ*<sub>*g*</sub>)

6.  *μ*<sub>*g*</sub> = *α*<sub>*g*</sub> + *β*<sub>*g*</sub> \* *z*

7.  *F*(*z*′,*z*) = *r*<sub>*r*</sub>(*z*) \* *r*<sub>*s*</sub>(*z*) \* *r*<sub>*d*</sub>(*z*′)

8.  *L**o**g**i**t*(*r*<sub>*r*</sub>(*z*)) = *α*<sub>*r*<sub>*r*</sub></sub> + *β*<sub>*r*<sub>*r*</sub></sub> \* *z*

9.  *L**o**g*(*r*<sub>*s*</sub>(*z*)) = *α*<sub>*r*<sub>*s*</sub></sub> + *β*<sub>*r*<sub>*s*</sub></sub> \* *z*

10. *r*<sub>*d*</sub>(*z*′) = *f*<sub>*r*<sub>*d*</sub></sub>(*μ*<sub>*r*<sub>*d*</sub></sub>,*σ*<sub>*r*<sub>*d*</sub></sub>)

Equation 1 describes how all the vital rates act on the initial trait
distribution to produce a new one at *t* + 1. Equations 3-6 describe how
existing individuals can survive, and if they survive, grow or shrink.
Equations 7-10 describe how existing individuals create new individuals.
In order to implement this, we usually fit regression models to our
data. The following set of generalized linear models correspond to the
functional forms described above:

1.  Survival (*s*(*z*) / `s`): a generalized linear model w/ a logit
    link.

    -   Example model formula:
        `glm(surv ~ size_1, data = my_surv_data, family = binomial())`

2.  Growth (*G*(*z*′,*z*) / `g`): a linear model with a Normal error
    distribution.

    -   Example model formula:
        `lm(size_2 ~ size_1, data = my_grow_data)`

3.  Pr(flowering) (*r*<sub>*r*</sub>(*z*) / `r_r`): a generalized linear
    model w/ a logit link.

    -   Example model formula:
        `glm(flower ~ size_1, data = my_repro_data, family = binomial())`

4.  Seed production (*r*<sub>*s*</sub>(*z*) / `r_s`): a generalized
    linear model w/ log link.

    -   Example model formula:
        `glm(seeds ~ size_1, data = my_flower_data, family = poisson())`

5.  Recruit size distribution (*r*<sub>*d*</sub>(*z*′) / `r_d`): a
    normal distribution w parameters `mu_fd` (mean) and `sd_fd`
    (standard deviation).

    -   Example computations:

        -   `mu_fd = mean(seedling_data$size_2)`

        -   `sd_fd = sd(seedling_data$size_2)`

The example below assumes we’ve already fit our vital rate models from
the raw data. In this example, the numbers are made up, but code that
extracts the values you need from a real regression model is provided in
comments.

``` r
# Load ipmr and get the parameter values. The data_list argument for define_kernel
# should hold every regression parameter and every constant used in the model.

library(ipmr)

my_data_list = list(s_int     = -2.2,   # coefficients(my_surv_mod)[1]
                    s_slope   = 0.25,  # coefficients(my_surv_mod)[2]
                    g_int     = 0.2,   # coefficients(my_grow_mod)[1]
                    g_slope   = 0.99,  # coefficients(my_grow_mod)[2]
                    sd_g      = 0.7,   # sd(resid(my_grow_mod))
                    r_r_int   = 0.003, # coefficients(my_pr_flower_mod)[1]
                    r_r_slope = 0.015, # coefficients(my_pr_flower_mod)[2]
                    r_s_int   = 0.45,   # coefficients(my_seed_mod)[1]
                    r_s_slope = 0.075, # coefficients(my_seed_mod)[2]
                    mu_fd     = 2,     # mean(recruit_data$size_next)
                    sd_fd     = 0.3)   # sd(recruit_data$size_next)

my_simple_ipm <- init_ipm(sim_gen   = "simple",
                          di_dd     = "di",
                          det_stoch = "det")


my_simple_ipm <- define_kernel(
  
  proto_ipm = my_simple_ipm,
    
  # Name of the kernel
  
  name      = "P_simple",
  
  # The type of transition it describes (e.g. continuous - continuous, discrete - continuous).
  # These must be specified for all kernels!
  
  family    = "CC",
  
  # The formula for the kernel. We dont need to tack on the "z'/z"s here.  
  
  formula   = s * g,
  
  # A named set of expressions for the vital rates it includes. 
  # note the use of user-specified functions here. Additionally, each 
  # state variable has a stateVariable_1 and stateVariable_2, corresponding to
  # z and z' in the equations above. We don't need to define these variables ourselves,
  # just reference them correctly based on the way we've set up our model on paper.
  
  # Perform the inverse logit transformation to get survival probabilities
  # from your model. plogis from the "stats" package does this for us. 

  s         = plogis(s_int + s_slope * dbh_1), 
  
  # The growth model requires a function to compute the mean as a function of dbh.
  # The SD is a constant, so we don't need to define that in ... expression, 
  # just the data_list.
  
  g         = dnorm(dbh_2, mu_g, sd_g),
  mu_g      = g_int + g_slope * dbh_1,
  
  
  # Specify the constant parameters in the model in the data_list. 
  
  data_list = my_data_list,
  states    = list(c('dbh')),
  
  # If you want to correct for eviction, set evict_cor = TRUE and specify an
  # evict_fun. ipmr provides truncated_distributions() to help. This function
  # takes 2 arguments - the type of distribution, and the name of the parameter/
  # vital rate that it acts on.
  
  evict_cor = TRUE,
  evict_fun = truncated_distributions(fun    = 'norm',
                                      target = 'g')
  ) 

my_simple_ipm <- define_kernel(
  proto_ipm = my_simple_ipm,
  name      = 'F_simple',
  formula   = r_r * r_s * r_d,
  family    = 'CC',
  
  # Inverse logit transformation for flowering probability
  # (because we used a logistic regression)
  
  r_r       = plogis(r_r_int + r_r_slope * dbh_1),
  
  # Exponential function for seed progression 
  # (because we used a Poisson)
  
  r_s       = exp(r_s_int + r_s_slope * dbh_1),
  
  # The recruit size distribution has no maternal effect for size,
  # so mu_fd and sd_fd are constants. These get passed in the 
  # data_list
  
  r_d       = dnorm(dbh_2, mu_fd, sd_fd),
  data_list = my_data_list,
  states    = list(c('dbh')),
  
  # Again, we'll correct for eviction in new recruits by
  # truncating the normal distribution.
  
  evict_cor = TRUE,
  evict_fun = truncated_distributions(fun    = 'norm',
                                      target = 'r_d')
) 

# Next, we have to define the implementation details for the model. 
# We need to tell ipmr how each kernel is integrated, what state
# it starts on (i.e. z from above), and what state
# it ends on (i.e. z' above). In simple_* models, state_start and state_end will 
# always be the same, because we only have a single continuous state variable. 
# General_* models will be more complicated.

my_simple_ipm <- define_impl(
  proto_ipm = my_simple_ipm,
  make_impl_args_list(
    kernel_names = c("P_simple", "F_simple"),
    int_rule     = rep("midpoint", 2),
    state_start  = rep("dbh", 2),
    state_end    = rep("dbh", 2)
  )
) 

my_simple_ipm <- define_domains(
  proto_ipm = my_simple_ipm,
  dbh = c(0, # the first entry is the lower bound of the domain.
          50, # the second entry is the upper bound of the domain.
          100 # third entry is the number of meshpoints for the domain.
  ) 
) 

# Next, we define the initial state of the population. We must do this because
# ipmr computes everything through simulation, and simulations require a 
# population state.

my_simple_ipm <- define_pop_state(
  proto_ipm = my_simple_ipm,
  n_dbh     = runif(100)
)

my_simple_ipm <- make_ipm(proto_ipm  = my_simple_ipm,
                          iterations = 200)


lambda_ipmr <- lambda(my_simple_ipm)
w_ipmr      <- right_ev(my_simple_ipm, iterations = 200)
v_ipmr      <- left_ev(my_simple_ipm, iterations = 200)

# make_ipm_report works on either proto_ipms or made ipm objects

make_ipm_report(my_simple_ipm, 
                render_output = TRUE, 
                title         = "my_simple_ipm_report")
```

`make_ipm_report()` generates an Rmarkdown file containing Latex
equations and parameter values used to implement the IPM. This may be
useful for publications/appendices, or for sending an IPM to the
[PADRINO](https://padrinodb.github.io/Padrino/) project for archiving.

## More complicated models

Examples of more complicated models are included in the vignettes,
accessible using either `browseVignettes('ipmr')` or by visiting the
Articles tab on [project’s webpage](https://levisc8.github.io/ipmr/).
Please file all bug reports in the Issues tab of this repository or
contact me via [email](mailto:levisc8@gmail.com) with a reproducible
example.

## Code of Conduct

We welcome contributions from other developers. Please note that the
ipmr project is released with a [Contributor Code of
Conduct](https://levisc8.github.io/ipmr/CODE_OF_CONDUCT.html). By
contributing to this project, you agree to abide by its terms.
