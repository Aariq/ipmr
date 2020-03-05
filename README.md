[![Travis build
status](https://travis-ci.org/levisc8/ipmr.svg?branch=master)](https://travis-ci.org/levisc8/ipmr)
[![AppVeyor build
status](https://ci.appveyor.com/api/projects/status/github/levisc8/ipmr?branch=master&svg=true)](https://ci.appveyor.com/project/levisc8/ipmr)
[![Lifecycle:
experimental](https://img.shields.io/badge/lifecycle-experimental-orange.svg)](https://www.tidyverse.org/lifecycle/#experimental)
[![CRAN
status](https://www.r-pkg.org/badges/version/ipmr)](https://cran.r-project.org/package=ipmr)
[![Codecov test
coverage](https://codecov.io/gh/levisc8/ipmr/branch/master/graph/badge.svg)](https://codecov.io/gh/levisc8/ipmr?branch=master)

# ipmr

`ipmr` is a package for implementing Integral Projection Models (IPMs)
in *R*. It relies heavily on the mathematical syntax of the models, and
does not try to abstract over the process of fitting vital rates. It is
still very much a work in progress, and so models implemented using
current code may not run in the not-so-distant future as more
complicated methods are brought online and tweaks are made. Below is a
brief overview of how `ipmr` classifies different model types followed
by examples of how to implement those types in this framework.

Note that this package **will not** help with the process of fitting
vital rate models at all\! That is a sufficiently different (and vast)
question that we decided it was not within the scope of this project.
This will only help you turn those regression models into an IPM without
shooting yourself in the foot. Thus, everything that follows assumes you
have parameterized those vital rate models and are now ready to begin
implementing your model.

Below is a brief overview of the package and some examples of how to
implement models with it. A more thorough introduction is available
[here](https://levisc8.github.io/ipmr/articles/ipmr-introduction.html).

## Model classes

The first step of defining a model in `ipmr` (assuming all parameters
have already been estimated) is to initialize the model using
`init_ipm()`. This function takes a single argument: `model_class`. The
`model_class` defines the basic infrastructure that will be available
for subsequent analyses and helps make sure the kernels are correctly
implemented from the underlying vital rates. `model_class` should be a
character string with at least 3 (but possibly 4) entries separated by
underscores (`_`). Below, the are the possible entries for each
position.

  - Position 1: `"simple"`/`"general"`

  - 1.  **simple**: This describes an IPM with a single continuous state
        variable and no discrete stages.

  - 2.  **general**: This describes and IPM with either more than one
        continuous state variable, one or more discrete stages, or both
        of the above. Basically, anything other than an IPM with a
        single continuous state variable.

  - Position 2: `"di"`/`"dd"`

  - A. **di**: This is used to denote a density-independent IPM.

  - B. **dd**: This is used to denote a density-dependent IPM.

  - Position 3: `"det"`/`"stoch"`

  - A. **det**: This is used to denote a deterministic IPM. If this is
    used in the third position of `model_class`, there should not be a
    fourth entry.

  - B. **stoch**: This is used to denote a stochastic IPM. If this is
    used in the third position of `model_class`, there should always be
    a fourth entry. The two possibilities for the fourth are described
    next.

  - Position 4: `"kern"`/`"param"`

  - A. **kern**: This describes an IPM with discretely varying
    parameters such that there values are known before the model is
    specified. This is usually the case with models that estimate random
    year/site effects and for which defining a multivariate joint
    distribution to sample parameters from is not desirable/needed.
    These models can be a bit more computationally efficient than the
    `param` alternative because all kernels can be constructed before
    the iteration procedure begins, as opposed to requiring
    reconstruction for every single iteration.

  - B. **param**: This describes an IPM with parameters that are
    re-sampled from some distribution at each iteration of the model
    (usually a multivariate joint distribution). This can be a
    multivariate normal defined by covarying slopes and intercepts, or
    posterior distribution from a Bayesian model. All that is required
    is that the parameters for the distribution are specified and that
    the function that generates the parameters at each iteration returns
    named lists that correspond to the parameter names in the model.
    Jump down to the `"simple_di_stoch_param"` example for some
    inspiration in writing those.

With the type of model selected, the `model_class` becomes a string and
the call to `init_ipm` is composed like so: `init_ipm(model_class =
"position1_position_2_position3_position4")`.

The following possibilities are currently or will become available in
`ipmr` (bold text denotes development progress):

  - Simple, density independent models: **Completed and ready**

<!-- end list -->

1.  `"simple_di_det"`

2.  `"simple_di_stoch_kern"`

3.  `"simple_di_stoch_param"`

<!-- end list -->

  - Simple, density dependent models: **Please be patient**

<!-- end list -->

4.  `"simple_dd_det"`

5.  `"simple_dd_stoch_kern"`

6.  `"simple_dd_stoch_param"`

<!-- end list -->

  - General, density independent models: **Completed and ready**

<!-- end list -->

7.  `"general_di_det"`

8.  `"general_di_stoch_kern"`

9.  `"general_di_stoch_param"`

<!-- end list -->

  - General, density dependent models: **Please be patient**

<!-- end list -->

10. `"general_dd_det"`

11. `"general_dd_stoch_kern"`

12. `"general_dd_stoch_param"`

Simple density-independent deterministic, simple kernel-resampled
stochastic, and simple parameter resampled stochastic models
(`simple_di_det`, `simple_di_stoch_kern`, `simple_di_stoch_param`) are
functional and detailed below as well as
[here](https://levisc8.github.io/ipmr/articles/ipmr-introduction.html).
The `general_*` versions of these are also ready, and an introduction to
them is available
[here](https://levisc8.github.io/ipmr/articles/general-ipms.html).
However, expect changes as more complicated methods are implemented\!
See below for an example of how to implement an IPM in this framework.

Next on the to-do list is to write generic functions for `lambda`,
`right_ev` (right eigenvector), `left_ev` (left eigenvector),
`sensitivity`, `elasticity`, and `plot`/`print` methods for all exported
classes (`proto_ipm`, implemented `ipm` objects). After that, Tables 3.2
and 3.3 from Ellner, Childs & Rees (2016). Finally, density dependant
methods for `make_ipm()` and the above mentioned generics (if
applicable).

## Examples for implemented IPM types

Here is a simple model implemented with `ipmr`. It will use the
following set of linear models:

1.  Survival (`s`): a generalized linear model w/ a logit link.

<!-- end list -->

  - Example model formula: `glm(surv ~ size_1, data = my_surv_data,
    family = binomial())`

<!-- end list -->

2.  Growth (`g`): a linear model with a Normal error distribution.

<!-- end list -->

  - Example model formula: `lm(size_2 ~ size_1, data = my_grow_data)`

<!-- end list -->

3.  Pr(flowering) (`f_r`): a generalized linear model w/ a logit link.

<!-- end list -->

  - Example model formula: `glm(flower ~ size_1, data = my_repro_data,
    family = binomial())`

<!-- end list -->

4.  Seed production (`f_s`): a generalized linear model w/ log link.

<!-- end list -->

  - Example model formula: `glm(seeds ~ size_1, data = my_repro_data,
    family = poisson())`

<!-- end list -->

5.  Recruit size distribution (`f_d`): a normal distribution w
    parameters `mu_fd` (mean) and `sd_fd` (standard deviation).

The example below assumes we’ve already fit our vital rate models from
the raw data. In this example, the numbers are made up, but code that
extracts the values you need from a real regression model is provided in
comments.

``` r
# Load ipmr and get the parameter values. The data_list argument for define_kernel
# should hold every regression parameter and every constant used in the model.

library(ipmr)

data_list = list(s_int     = 2.2,   # coefficients(my_surv_mod)[1]
                 s_slope   = 0.25,  # coefficients(my_surv_mod)[2]
                 g_int     = 0.2,   # coefficients(my_grow_mod)[1]
                 g_slope   = 1.02,  # coefficients(my_grow_mod)[2]
                 sd_g      = 0.7,   # sd(resid(my_grow_mod))
                 f_r_int   = 0.003, # coefficients(my_pr_flower_mod)[1]
                 f_r_slope = 0.015, # coefficients(my_pr_flower_mod)[2]
                 f_s_int   = 1.3,   # coefficients(my_seed_mod)[1]
                 f_s_slope = 0.075, # coefficients(my_seed_mod)[2]
                 mu_fd     = 2,     # mean(recruit_data$size_next)
                 sd_fd     = 0.3)   # sd(recruit_data$size_next)

my_simple_ipm <- init_ipm('simple_di_det') %>%
  define_kernel(
    
    # Name of the kernel
    
    name      = "P_simple",
    
    # The type of transition it describes (e.g. continuous - continuous, discrete - continuous).
    # These must be specified for all kernels!
    
    family    = "CC",
    
    # The formula for the kernel. 
    
    formula   = s * g,
    
    # A named set of expressions for the vital rates it includes. 
    # note the use of user-specified functions here. Additionally, each 
    # state variable has a stateVariable_1 and stateVariable_2 internally defined
    # for the domain associated with it. Use these to distinguish between 
    # size/weight/etc at time t vs size/weight/etc at time t+1
    
    # Perform the inverse logit transformation to get survival probabilities
    # from your model. For examples on using predict(my_surv_mod,...),
    # see below.
    
    s         = 1 / (1 + exp(-(s_int + s_slope * dbh_1))), 
    
    # The growth model requires a function to compute the mean as a function of dbh.
    # The SD is a constant, so we don't need to define that in ... expression, 
    # just the data_list.
    
    g         = dnorm(dbh_2, mu_g, sd_g),
    mu_g      = g_int + g_slope * dbh_1,

    
    # Specify the constants in the model in the data_list. 
    
    data_list = data_list,
    states    = list(c('dbh')),
    
    # If you want to correct for eviction, set evict = TRUE and specify an
    # evict_fun. ipmr provides truncated_distributions() to help.
    
    evict     = TRUE,
    evict_fun = truncated_distributions('norm',
                                        'g')
  ) %>%
  define_kernel('F_simple',
                formula   = f_r * f_s * f_d,
                family    = 'CC',
                
                # Inverse logit transformation for flowering probability
                # (because we used a logistic regression)
                
                f_r       = 1 / (1 + exp( - (f_r_int + f_r_slope * dbh_1))),
                
                # Exponential function for seed progression 
                # (because we used a Poisson)
                
                f_s       = exp(f_s_int + f_s_slope * dbh_1),
                
                # The recruit size distribution has no maternal effect for size,
                # so mu_fd and sd_fd are constants. These get passed in the 
                # data_list
                
                f_d       = dnorm(dbh_2, mu_fd, sd_fd),
                data_list = data_list,
                states    = list(c('dbh')),
                
                # Again, we'll correct for eviction in new recruits by
                # truncating the normal distribution.
                
                evict     = TRUE,
                evict_fun = truncated_distributions('norm',
                                                    'f_d')
  ) %>%
  
  # K kernels get their own special define_k function. Rather than use the formula
  # parameter, it simply takes named expressions for the format of the iteration
  # kernels. It can also take expressions showing how these kernels generate 
  # population states at t+1 as a function of population state at t - see examples
  # below for how to do that.
  
  define_k('K',
           K         = P_simple + F_simple,
           
           # This is a new family - at the moment all K's will get the
           # 'IPM' family
           
           family    = 'IPM',
           
           # This kernel has no additional parameters, so the data_list
           # is empty
           
           data_list = list(),
           states    = list(c('dbh')),
           # We've already corrected eviction in the sub-kernels, so there's no
           # need to do that here
           evict     = FALSE
  ) %>%
  # Next, we have to define the implementation details for the model. 
  # We need to tell ipmr how each kernel is integrated, what domain
  # it starts on (i.e. the size/weight/etc from above), and what domain
  # it ends on. In simple_* models, dom_start and dom_end will always be the same,
  # because we only have a single continuous state variable. General_*
  # models will be more complicated.
  
  define_impl(
    make_impl_args_list(
      kernel_names = c("K_simple", "P_simple", "F_simple"),
      int_rule     = rep("midpoint", 3),
      dom_start    = rep("dbh", 3),
      dom_end      = rep("dbh", 3)
    )
  ) %>%
  define_domains(
    dbh = c(0, # the first entry is the lower bound of the domain.
            50, # the second entry is the upper bound of the domain.
            100 # third entry is the number of meshpoints for the domain.
    ) 
  )  %>%
  make_ipm()


lambda_ipmr <- lambda(my_simple_ipm, comp_method = 'eigen')
w_ipmr      <- right_ev(my_simple_ipm)
```

If you’re interested in seeing how `ipmr` output compares to models
implemented by hand, there is an article on that
[here](https://levisc8.github.io/ipmr/articles/sanity-checks.html).

## Simple, density independent, stochastic kernel resampled models

These models are usually used to model discretely varying environments.
For example, data may come from multiple sites and/or multiple years,
and so the vital rate regressions could have fixed or random effects for
those variables. They have a special syntax that mirrors the
mathematical notation of these models (and has the side effect of saving
you a considerable amount of copying/pasting/typing in general).

The syntax uses a `name_hierarchicalVariable` notation. These names are
automatically expanded to include the multiple levels of the
hierarchical variable. For example, the P kernel for a model with 5
years of data could be `name`’d `P_yr`, and the underlying vital rates
would be denoted `vitalRate_yr`. The parameter values in the `data_list`
are the exception to this - they must be labeled with the actual values
that the hierarchical variable takes.

The example below simulates a lifecycle where reproduction is always
fatal. Thus, the survival term also includes the probability of
reproducing. This is meant to (hopefully) demonstrate the flexibility of
the framework. Along the way, it’ll make use of the `purrr` R package to
manipulate parameters and lists. The vital rates take the following
form:

1.  survival (`s`): a logistic regression with a random year intercept
    (`s_r_yr`).

<!-- end list -->

  - Example model formula: `glmer(surv ~ size_1 + (1 | yr), data =
    my_surv_data, family = binomial()))`

<!-- end list -->

2.  growth (`g`): A linear regression random year intercept (`g_r_yr`).

<!-- end list -->

  - Example model formula: `lmer(size_2 ~ size_1 + (1 | yr), data =
    my_grow_data, family = gaussian()))`

<!-- end list -->

3.  pr(flowering) (`p_r`): A logistic regression. This has no random
    year effect.

<!-- end list -->

  - Example model formula: `glm(flower ~ size_1 , data = my_surv_data,
    family = binomial()))`

<!-- end list -->

4.  seed production (`f_s`): A poisson regression with a random year
    intercept (`f_s_r_yr`)

<!-- end list -->

  - Example model formula: `glmer(seed_num ~ size_1 + (1 | yr), data =
    my_surv_data, family = poisson()))`

<!-- end list -->

5.  recruit size distribution (`f_d`): A normal distribution with two
    constant parameters, the mean (`mu_fd`) and standard deviation
    (`sd_fd`).

In this example, the random effects are not correlated with each other,
corresponding to separate models for each vital rate.

``` r
# This is intended to simulate a monocarpic perennial life history where flowering is always fatal.
# Note that this means the survival function also includes the probability of reproduction function.

library(ipmr)
library(purrr)

set.seed(50127)

# Define some fixed parameters

data_list = list(
  s_int     = 1.03,
  s_slope   = 2.2,
  g_int     = 8,
  g_slope   = 0.92,
  sd_g      = 0.9,
  f_r_int   = 0.09,
  f_r_slope = 0.05,
  f_s_int   = 0.1,
  f_s_slope = 0.005,
  mu_fd     = 9,
  sd_fd     = 2
)

# Now, simulate some random intercepts for growth, survival, and offspring production.
# For extracting random effects from real models, the following snippet should work
# for most: 

# t(ranef(my_grow_mod))

# The ranef() function is generic, and so whether it exists or not depends on 
# the package used to fit the vital rate regression. A google search for
# "extract random effect estimates from class(insert_your_model_here) in R"
# should get you close to the code you need.

g_r_int   <- rnorm(5, 0, 0.3) 
s_r_int   <- rnorm(5, 0, 0.7)
f_s_r_int <- rnorm(5, 0, 0.2)

# Now, generate parameter names for each set of random effect estimates. 

nms <- paste("r_", 1:5, sep = "")

names(g_r_int)   <- paste('g_', nms, sep = "")
names(s_r_int)   <- paste('s_', nms, sep = "")
names(f_s_r_int) <- paste('f_s_', nms, sep = "")

# Each set of parameters is converted to a named list. The names should match
# the variables referenced in each define_kernel()/define_k() call.

g_params   <- as.list(g_r_int)
s_params   <- as.list(s_r_int)
f_s_params <- as.list(f_s_r_int)

# purrr::splice combines each separate list into one without creating any 
# additional depth to it. This keeps things a bit tidier.

params     <- splice(data_list, g_params, s_params, f_s_params)

## Now, let's get to the ipmr model

# This example will use additional 'usr_funs' to be passed into make_ipm(). 

inv_logit <- function(sv, int, slope) {
  return(
    1/(1 + exp(-(int + slope * sv)))
  )
}

inv_logit_r <- function(sv, int, slope, r_eff) {
  return(
    1/(1 + exp(-(int + slope * sv + r_eff)))
  )
}

pois_r <- function(sv, int, slope, r_eff) {
  return(
    exp(
      int + slope * sv + r_eff
    )
  )
}

# The "model_class" argument is now changed to reflect a different model type

monocarp_sys <- init_ipm('simple_di_stoch_kern') %>%
  define_kernel(
    
    # The yr suffix is appended to the kernel name and the parameter names
    # within each vital rate expression. ipmr substitutes in the hier_levels
    # for each suffix occurrence, thus changing P_yr in P_1, P_2, P_3, P_4, P_5,
    # s_yr in s_1, s_2, s_3, s_4, and s_5. s_r_yr is converted to s_r_1, s_r_2, 
    # etc. In the case of s_r_yr, provided that the names in the data_list match
    # the expanded names, all will go well!
    
    name             = 'P_yr',
    
    # The formula is updated to reflect that survival and growth
    # vary from year to year. We append the _yr suffix to each to let ipmr know
    # to expand the expression
    
    formula          = s_yr * g_yr ,
    family           = "CC",
    
    # This is a monocarpic perennial, so flowering is fatal. Thus, in order
    # to survive to t+1, you have to both survive AND not flower. Also, note
    # the use of inv_logit_r and inv_logit functions that we defined above.
    
    s_yr             = inv_logit_r(ht_1, s_int, s_slope, s_r_yr) * 
                       (1 - inv_logit(ht_1, f_r_int, f_r_slope)),
    
    # Since the model for the mean of growth has a random year intercept,
    # we modify all of these expressions as well to include the _yr
    
    g_yr             = dnorm(ht_2, mu_g_yr, sd_g),
    mu_g_yr          = g_int + g_slope * ht_1 + g_r_yr,
    
    data_list        = params,
    states           = list(c('ht')),
    
    # This where we tell ipmr that the model has hierarchical effects. 
    # The levels that it takes should specified in a named list. We're pretending
    # this a random year effect, so we use the name 'yr'. This name can be whatever you
    # want though.
    
    has_hier_effs    = TRUE,
    levels_hier_effs = list(yr = 1:5),
    
    # Again, correct for eviction. Note that the 2nd parameter is also modified
    # with the _yr suffix
    
    evict            = TRUE,
    evict_fun        = truncated_distributions("norm", "g_yr")
  ) %>%
  define_kernel(
    
    # As above, we modify all the expressions that have a random effect with the _yr
    # suffix to let ipmr know to expand those.
    
    name             = "F_yr",
    formula          = f_r * f_s_yr * f_d,
    family           = "CC",
    f_r              = inv_logit(ht_1, f_r_int, f_r_slope),
    f_s_yr           = pois_r(ht_1, f_s_int, f_s_slope, f_s_r_yr),
    f_d              = dnorm(ht_2, mu_fd, sd_fd),
    data_list        = params,
    states           = list(c('ht')), 
    has_hier_effs    = TRUE,
    levels_hier_effs = list(yr = 1:5),
    evict            = TRUE,
    evict_fun        = truncated_distributions("norm", "f_d")
  ) %>%
  define_k(
    name             = 'K_yr',
    K_yr             = P_yr + F_yr,
    family           = "IPM",
    data_list        = list(),
    states           = list(c("ht")),
    has_hier_effs    = TRUE,
    levels_hier_effs = list(yr = 1:5)
  ) %>%
  define_impl(
    make_impl_args_list(
      kernel_names = c("K_yr", "P_yr", "F_yr"),
      int_rule     = rep("midpoint", 3),
      dom_start    = rep("ht", 3),
      dom_end      = rep("ht", 3)
    )
  ) %>%
  define_domains(ht = c(0.2, 40, 100)) %>%
  make_ipm(usr_funs = list(inv_logit   = inv_logit,
                           inv_logit_r = inv_logit_r,
                           pois_r      = pois_r))

lambda_ipmr <- lambda(monocarp_sys, 
                      comp_method = 'eigen',
                      type_lambda = 'all')
```

## Simple, density independent models in continuously varying environments

These models are stochastic, but rather than building the iteration
kernels first and then iterating them - distributions for varying
parameters are specified ahead of time and then resampled at each
iteration. `ipmr` is careful to only evaluate each expression once per
iteration, allowing you to specify parameters sampled from a joint
distribution with a single expression. Their notation *usually* does not
require suffix notation, though this example uses it to make clear which
parameters are varying and which are fixed. All that’s required is that:

1.  the function returns a named list of parameter values and

2.  the names of the list values and the names used in the vital
    rate/kernel expressions match.

The user-specified function should go into `define_env_state()` call.
The values can then be referenced using `list_name$parameter_name`
notation in the vital rate expressions.

This example uses the `rmvnorm` function from the `mvtnorm` package to
demonstrate how one might sample from a joint distribution of
parameters. One could also pass a function that samples from a posterior
distribution of a Bayesian model, and pass the associated posterior to
the `data_list` argument in `define_env_state` to propagate parameter
uncertainty throughout the model.

``` r
library(ipmr)
library(mvtnorm)

# Define the fixed parameters in a list

set.seed(2123008)

data_list <- list(s_slope   = 0.3,
                  g_slope   = 0.99,
                  g_sd      = 0.2,
                  f_r_slope = 0.03,
                  f_s_slope = 0.001,
                  f_d_mu    = 1.1,
                  f_d_sd    = 0.1)

# Simulate some random parameters to define the multivariate distribution
# to sample from. In user-specified models, these will likely come from mixed
# effects models fit to real data.
r_means <- c(s_int_yr   = 0.8,
             g_int_yr   = 0.1,
             f_r_int_yr = 0.3,
             f_s_int_yr = 0.4)

r_sigma <- runif(16)
r_sigma <- matrix(r_sigma, nrow = 4)

r_sigma <- r_sigma %*% t(r_sigma) # make symmetrical - most users should skip this step.

iterations <- 10

# Specify an initial population vector. This can also be done in the call to
# define_pop_state

init_pop_vec <- runif(100)

# Again, we can define our own functions and pass them into calls to make_ipm
inv_logit <- function(int, slope, sv1) {
  1/(1 + exp(-(int + slope * sv1)))
}

# In this case, we define a wrapper function around rmvnorm to make sure
# that the names of the returned values match those in the vital rate
# expressions.

mvt_wrapper <- function(r_means, r_sigma, nms) {
  out <- mvtnorm::rmvnorm(1, r_means, r_sigma) %>%
    as.list()
  
  names(out) <- nms
  return(out)
}

param_resamp_model <- init_ipm('simple_di_stoch_param') %>%
  define_kernel(
    name    = 'P',
    formula = s_g_mult(s, g),
    family  = 'CC',
    
    # Note here that the growth and survival intercepts are prefaced with env_params$...
    # This is because the mvt_wrapper function is named env_params in the call
    # to define_env_state(). env_params is not a reserved word, so you can call
    # it whatever you like.
    
    g_mu          = env_params$g_int_yr + g_slope * surf_area_1,
    s             = inv_logit(env_params$s_int_yr, s_slope, surf_area_1),
    g             = dnorm(surf_area_2, g_mu, g_sd),
    data_list     = data_list,
    states        = list(c('surf_area')),
    has_hier_effs = FALSE,
    evict         = TRUE,
    evict_fun     = truncated_distributions("norm", "g")
  ) %>%
  define_kernel(
    name          = 'F',
    formula       = f_r * f_s * f_d,
    family        = 'CC',
    
    # Note the insertion of env_params$param_name here as well
    
    f_r           = inv_logit(env_params$f_r_int_yr, f_r_slope, surf_area_1),
    f_s           = exp(env_params$f_s_int_yr + f_s_slope * surf_area_1),
    f_d           = dnorm(surf_area_2, f_d_mu, f_d_sd),
    
    data_list     = data_list,
    states        = list(c('surf_area')),
    has_hier_effs = FALSE,
    evict         = TRUE,
    evict_fun     = truncated_distributions("norm", "f_d")
  ) %>%
  define_k(
    name = 'K',
    
    # Note that here, we specify both the form of the iteration kernel and
    # the iteration procedure. right_mult() is a helper function to make sure
    # population states are multiplied by the kernels correctly.
    
    K               = P + F,
    n_surf_area_t_1 = right_mult(K, n_surf_area_t),
    
    family          = 'IPM',
    data_list       = data_list,
    states          = list(c('surf_area')),
    has_hier_effs   = FALSE,
    evict           = FALSE
  ) %>%
  define_impl(
    make_impl_args_list(
      kernel_names = c('P', "F", "K"),
      int_rule     = rep('midpoint', 3),
      dom_start    = rep('surf_area',3),
      dom_end      = rep('surf_area', 3)
    )
  ) %>%
  define_domains(surf_area = c(0, 10, 100)) %>%
  define_env_state(
    env_params = mvt_wrapper(r_means, r_sigma, nms = c('s_int_yr',
                                                       'g_int_yr',
                                                       'f_r_int_yr',
                                                       'f_s_int_yr')),
    data_list = list(
      r_means = r_means,
      r_sigma = r_sigma
    )
  ) %>%
  define_pop_state(
    pop_vectors = list(n_surf_area_t = init_pop_vec),
  ) %>%
  make_ipm(usr_funs   = list(inv_logit = inv_logit,
                             mvt_wrapper = mvt_wrapper),
           iterate    = TRUE,
           iterations = 10)

lambda(param_resamp_model,
       comp_method = 'pop_size',
       type_lambda = 'all')

lambda_ipmr
```
