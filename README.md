
[![Travis build
status](https://travis-ci.org/levisc8/ipmr.svg?branch=master)](https://travis-ci.org/levisc8/ipmr)
[![Codecov test
coverage](https://codecov.io/gh/levisc8/ipmr/branch/master/graph/badge.svg)](https://codecov.io/gh/levisc8/ipmr?branch=master)

# ipmr

Simple density-independent deterministic, simple kernel-resampled
stochastic, and simple parameter resampled stochastic models
(`simple_di_det`, `simple_di_stoch_kern`, `simple_di_stoch_param`) are
now functional. However, expect changes as more complicated methods are
implemented\! See below for an example of how to implement an IPM in
this framework.

Next on the implementation to-do list are the general versions of these
(still density
independent).

``` r
# Example of the setup for a simple IPM without density dependence or environmental
# stochasticity

library(ipmr)

data_list <- list(s_int = 2.2,
                  s_slope = 0.25,
                  g_int = 0.2,
                  g_slope = 1.02,
                  sd_g = 0.7,
                  f_r_int = 0.03,
                  f_r_slope = 0.015,
                  f_s_int = 1.3,
                  f_s_slope = 0.075,
                  mu_fd = 0.5,
                  sd_fd = 0.2)

s <- function(sv1, params) {
  1/(1 + exp(-(params[1] + params[2] * sv1)))
}


g <- function(sv1, sv2, params) {
  mu <- params[1] + params[2] * sv1
  dnorm(sv2, mean = mu, sd = params[3])
}

f_r <- function(sv1, params) {
  1/(1 + exp(-(params[1] + params[2] * sv1)))
}

f_s <- function(sv1, params) {
  exp(params[1] + params[2] * sv1)
}

f_d <- function(sv2, params) {
  dnorm(sv2, mean = params[1], sd = params[2])
}

fec <- function(sv1, sv2, params) {
  f_r(sv1, params[1:2]) * f_s(sv1, params[3:4]) * f_d(sv2, params[5:6])
}

b  <- seq(0, 50, length.out = 101)
d1 <- d2 <- (b[2:101] + b[1:100]) * 0.5
h  <- d1[3] - d1[2]


G  <- outer(d1, d2, FUN = g, params = c(data_list$g_int,
                                        data_list$g_slope,
                                        data_list$sd_g))
G2 <- G/matrix(as.vector(apply(G, 2, sum)),
               nrow = length(d1),
               ncol = length(d1),
               byrow = TRUE)

S  <- s(d1, c(data_list$s_int, data_list$s_slope))

P  <- h *  t(S * t(G2))

Fm <- h * outer(d1, d2, FUN = fec, params = unlist(data_list[6:11]))

K <- P + Fm

lambda_usr <- Re(eigen(K)$values[1])
w_usr <- Re(eigen(K)$vectors[ , 1])


# User specified functions can be passed to make_ipm(usr_funs = list(my_fun = my_fun)).
# inv_logit is a simple example, but more complicated ones can be specified as well. 

inv_logit <- function(int, slope, sv) {
  return(1/(1 + exp(-(int + slope * sv))))
}

state_list <- list(c('dbh'))

x <- init_ipm('simple_di_det') %>%
  define_kernel(
    # Name of the kernel
    name = "P",
    # The type of transition it describes (e.g. continuous - continuous, discrete - continuous)
    family = "CC",
    # The formula for the kernel. s_g_mult() is a helper function to make sure the
    # survival vector is correctly aligned with the kernel.
    formula = s_g_mult(s, g),
    # A named set of expressions for the vital rates it includes. 
    
    s = inv_logit(s_int, s_slope, dbh_1), # note the use of user-specified function here
    g = dnorm(dbh_2, mu_g, sd_g),
    mu_g = g_int + g_slope * dbh_1,
    data_list = list(s_int = 2.2,
                     s_slope = 0.25,
                     g_int = 0.2,
                     g_slope = 1.02,
                     sd_g = 0.7),
    states = state_list,
    has_hier_effs = FALSE,
    # The evict_fun argument can take any function. ipmr provides built in
    # truncated density functions, but a user specified one will work as well, 
    # provided all parameters are provided in the data_list argument.
    evict = TRUE,
    evict_fun = truncated_distributions(g,
                                        n_mesh_p = 100)) %>%
  
  # Define the fecundity kernel
  define_kernel(
    name = 'F',
    formula = f_r * f_s * f_d,
    family = 'CC',
    f_r = inv_logit(f_r_int, f_r_slope, dbh_1),
    f_s = exp(f_s_int + f_s_slope * dbh_1),
    f_d = dnorm(dbh_2, mu_fd, sd_fd),
    data_list = list(f_r_int = 0.03,
                     f_r_slope = 0.015,
                     f_s_int = 1.3,
                     f_s_slope = 0.075,
                     mu_fd = 0.5,
                     sd_fd = 0.2),
    states = state_list,
    evict = FALSE
  ) %>%
  # K kernels get their own special define_k function. Rather than use the formula
  # parameter, it simply takes ..., allowing you to specify the form of the iteration
  # kernel and the population vector at T + 1 simulataneously. This is likely to
  # be more useful for stochastic simulation models than deterministic models
  define_k(
    name = "K",
    K = P + F,
    family = "IPM",
    states = state_list,
    evict = FALSE # this is dealt with in the sub-kernels, so no need to do so again.
  ) %>%
  define_impl(
    kernel_impl_list =  make_impl_args_list(
      kernel_names = c("K", "P", "F"),
      int_rule = rep('midpoint', 3),
      dom_start = rep('dbh', 3),
      dom_end = rep('dbh', 3)
    )
  ) %>% 
  define_domains(
    dbh = c(0, # the first entry is the lower bound of the domain.
            50, # the second entry is the upper bound of the domain.
            100 # third entry is the number of meshpoints for the domain.
    )
  ) %>%
  make_ipm(usr_funs = list(inv_logit = inv_logit))

lambda_ipmr <- Re(eigen(x$iterators$K)$values[1])
w_ipmr <- Re(eigen(x$iterators$K)$vectors[ , 1])

lambda_ipmr - lambda_usr
```

## Simple, density independent, stochastic kernel resampled models

These models are typically the result of vital rate models that are fit
in a mixed effects framework (e.g. multiple sites or multiple years of
data). They have a special syntax that mirrors the mathematical notation
of these models (and has the side effect of saving you a considerable
amount of copying/pasting/typing in general).

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
the
framework.

``` r
# rlang is a useful shortcut for splicing named values into lists. purrr is used to manipulate said lists.
# This is intended to simulate a monocarpic perennial life history where flowering is always fatal.
# Note that this means the survival function also includes the probability of reproduction function.

library(rlang)
library(ipmr)
library(purrr)

# define functions for target ipm

# Survival - logistic regression
s <- function(sv1, params, r_effect) {
  1/(1 + exp(-(params[1] + params[2] * sv1 + r_effect))) *
    (1 - f_r(sv1, params[3:4]))
}

# Growth
g <- function(sv1, sv2, params, r_effect) {
  mu <- params[1] + params[2] * sv1 + r_effect
  dnorm(sv2, mean = mu, sd = params[3])
}

# probability of reproducing
f_r <- function(sv1, params) {
  1/(1 + exp(-(params[1] + params[2] * sv1)))
}

# offspring production
f_s <- function(sv1, params, r_effect) {
  exp(params[1] + params[2] * sv1 + r_effect)
}

# offspring size distribution
f_d <- function(sv2, params) {
  dnorm(sv2, mean = params[1], sd = params[2])
}

# constructor function for the F kernel
fec <- function(sv1, sv2, params, r_effect) {
  f_r(sv1, params[1:2]) * f_s(sv1, params[3:4], r_effect) * f_d(sv2, params[5:6])
}


set.seed(50127)


# Define some fixed parameters
data_list = list(s_int = 1.03,
                 s_slope = 2.2,
                 g_int = 8,
                 g_slope = 0.92,
                 sd_g = 0.9,
                 f_r_int = 0.09,
                 f_r_slope = 0.05,
                 f_s_int = 0.1,
                 f_s_slope = 0.005,
                 mu_fd = 9,
                 sd_fd = 2)

# Now, simulate some random intercepts for growth, survival, and offspring production
g_r_int <- rnorm(5, 0, 0.3)
s_r_int <- rnorm(5, 0, 0.7)
f_s_r_int <- rnorm(5, 0, 0.2)

nms <- paste("r_", 1:5, sep = "")

names(g_r_int) <- paste('g_', nms, sep = "")
names(s_r_int) <- paste('s_', nms, sep = "")
names(f_s_r_int) <- paste('f_s_', nms, sep = "")

# The !!! operator used inside of list2 from rlang takes the named vector
# and converts it to a named list. This can be spliced into the data list 
# to rapidly make a parameter set suitable for usage in the data_list argument
# of define_kernel

g_params <- list2(!!! g_r_int)
s_params <- list2(!!! s_r_int)
f_s_params <- list2(!!! f_s_r_int)

params <- splice(data_list, g_params, s_params, f_s_params)


b <- seq(0.2, 40, length.out = 101)
sv1 <- sv2 <- (b[2:101] + b[1:100]) * 0.5
h <- sv1[2] - sv1[1]

# repetitive to demonstrate the typical kernel construction process.

g_1 <- outer(sv1, sv2, FUN = g, params = c(params$g_int,
                                               params$g_slope,
                                               params$sd_g),
                 r_effect = params$g_r_1)

g_2 <- outer(sv1, sv2, FUN = g, params = c(params$g_int,
                                               params$g_slope,
                                               params$sd_g),
                 r_effect = params$g_r_2)
g_3 <- outer(sv1, sv2, FUN = g, params = c(params$g_int,
                                               params$g_slope,
                                               params$sd_g),
                 r_effect = params$g_r_3)

g_4 <- outer(sv1, sv2, FUN = g, params = c(params$g_int,
                                               params$g_slope,
                                               params$sd_g),
                 r_effect = params$g_r_4)

g_5 <- outer(sv1, sv2, FUN = g, params = c(params$g_int,
                                               params$g_slope,
                                               params$sd_g),
                 r_effect = params$g_r_5)

g_1 <- truncated_distributions(g_1, n_mesh_p = 100)
g_2 <- truncated_distributions(g_2, n_mesh_p = 100)
g_3 <- truncated_distributions(g_3, n_mesh_p = 100)
g_4 <- truncated_distributions(g_4, n_mesh_p = 100)
g_5 <- truncated_distributions(g_5, n_mesh_p = 100)

s_1 <- s(sv1, c(params$s_int, params$s_slope,
                params$f_r_int, params$f_r_slope), params$s_r_1)
s_2 <- s(sv1, c(params$s_int, params$s_slope,
                params$f_r_int, params$f_r_slope), params$s_r_2)
s_3 <- s(sv1, c(params$s_int, params$s_slope,
                params$f_r_int, params$f_r_slope), params$s_r_3)
s_4 <- s(sv1, c(params$s_int, params$s_slope,
                params$f_r_int, params$f_r_slope), params$s_r_4)
s_5 <- s(sv1, c(params$s_int, params$s_slope,
                params$f_r_int, params$f_r_slope), params$s_r_5)

P_1 <- t(s_1 * t(g_1)) * h
P_2 <- t(s_2 * t(g_2)) * h
P_3 <- t(s_3 * t(g_3)) * h
P_4 <- t(s_4 * t(g_4)) * h
P_5 <- t(s_5 * t(g_5)) * h

# These are not corrected for eviction, but they probably should be
F_1 <- h * outer(sv1, sv2, FUN = fec,
                 params = unlist(params[6:11]),
                 r_effect = params$f_s_r_1)

F_2 <- h * outer(sv1, sv2, FUN = fec,
                 params = unlist(params[6:11]),
                 r_effect = params$f_s_r_2)

F_3 <- h * outer(sv1, sv2, FUN = fec,
                 params = unlist(params[6:11]),
                 r_effect = params$f_s_r_3)

F_4 <- h * outer(sv1, sv2, FUN = fec,
                 params = unlist(params[6:11]),
                 r_effect = params$f_s_r_4)

F_5 <- h * outer(sv1, sv2, FUN = fec,
                 params = unlist(params[6:11]),
                 r_effect = params$f_s_r_5)


K_1 <- P_1 + F_1
K_2 <- P_2 + F_2
K_3 <- P_3 + F_3
K_4 <- P_4 + F_4
K_5 <- P_5 + F_5

lambdas <- c(Re(eigen(K_1)$values[1]),
             Re(eigen(K_2)$values[1]),
             Re(eigen(K_3)$values[1]),
             Re(eigen(K_4)$values[1]),
             Re(eigen(K_5)$values[1]))


## ipmr version

# define the levels of the hierarchical variable and save them in a named
# list that corresponds to the suffix in the kernel notation
hier_levels <- list(yr = 1:5)

# additional usr_funs to be passed into make_ipm()
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
    
    name = 'P_yr',
    formula = t(s_yr * t(g_yr)) ,
    family = "CC",
    s_yr = inv_logit_r(ht_1, s_int, s_slope, s_r_yr) * 
      (1 - inv_logit(ht_1, f_r_int, f_r_slope)),
    g_yr = dnorm(ht_2, mean = mu_g_yr, sd = sd_g),
    mu_g_yr = g_int + g_slope * ht_1 + g_r_yr,
    data_list = params,
    states = list(c('ht')),
    has_hier_effs = TRUE,
    levels_hier_effs = hier_levels,
    evict = TRUE,
    # Note that the suffix is appended here since the growth kernel also has a random intercept.
    evict_fun = truncated_distributions(g_yr,
                                        n_mesh_p = 100)
  ) %>%
  define_kernel(
    "F_yr",
    formula = f_r * f_s_yr * f_d,
    family = "CC",
    f_r = inv_logit(ht_1, f_r_int, f_r_slope),
    f_s_yr = pois_r(ht_1, f_s_int, f_s_slope, f_s_r_yr),
    f_d = dnorm(ht_2, mean = mu_fd, sd = sd_fd),
    data_list = params,
    states = list(c('ht')),
    has_hier_effs = TRUE,
    levels_hier_effs = hier_levels,
    evict = FALSE) %>%
  define_k(
    'K_yr',
    K_yr = P_yr + F_yr,
    family = "IPM",
    data_list = params,
    states = list(c("ht")),
    has_hier_effs = TRUE,
    levels_hier_effs = hier_levels,
    evict = FALSE
  ) %>%
  define_impl(
    make_impl_args_list(
      kernel_names = c("K_yr", "P_yr", "F_yr"),
      int_rule = rep("midpoint", 3),
      dom_start = rep("ht", 3),
      dom_end = rep("ht", 3)
    )
  ) %>%
  define_domains(ht = c(0.2, 40, 100)) %>%
  make_ipm(usr_funs = list(inv_logit = inv_logit,
                           inv_logit_r = inv_logit_r,
                           pois_r = pois_r))


ks          <- monocarp_sys$iterators
lambda_ipmr <- vapply(ks, function(x) Re(eigen(x)$values[1]), numeric(1))

lambda_ipmr - lambdas
```

## Simple, density independet, parameter resampled models

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

data_list <- list(s_slope = 0.3,
                  g_slope = 0.99,
                  g_sd = 0.2,
                  f_r_slope = 0.03,
                  f_s_slope = 0.001,
                  f_d_mu = 1.1,
                  f_d_sd = 0.1)

# Simulate some random parameters to define the multivariate distribution
# to sample from. In user-specified models, these will likely come from mixed
# effects models fit to real data.
r_means <- c(s_int_yr = 0.8,
             g_int_yr = 0.1,
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
    'P',
    formula = s_g_mult(s, g),
    family = 'CC',
    # Note here that the growth and survival intercepts are prefaced with env_params$...
    # This is because the mvt_wrapper function is named env_params in the call
    # to define_env_state(). env_params is not a reserved word, so you can call
    # it whatever you like.
    g_mu = env_params$g_int_yr + g_slope * surf_area_1,
    s = inv_logit(env_params$s_int_yr, s_slope, surf_area_1),
    g = dnorm(surf_area_2, g_mu, g_sd),
    data_list = data_list,
    states = list(c('surf_area')),
    has_hier_effs = FALSE,
    evict = TRUE,
    evict_fun = truncated_distributions(g,
                                        100)
  ) %>%
  define_kernel(
    'F',
    formula = f_r * f_s * f_d,
    family = 'CC',
    f_r = inv_logit(env_params$f_r_int_yr, f_r_slope, surf_area_1),
    f_s = exp(env_params$f_s_int_yr + f_s_slope * surf_area_1),
    f_d = dnorm(surf_area_2, f_d_mu, f_d_sd),
    data_list = data_list,
    states = list(c('surf_area')),
    has_hier_effs = FALSE,
    evict = FALSE
  ) %>%
  define_k(
    'K',
    
    # Note that here, we specify both the form of the iteration kernel and
    # the iteration procedure. right_mult() is a helper function to make sure
    # population states are multiplied by the kernels correctly.
    K = P + F,
    n_surf_area_t_1 = right_mult(K, n_surf_area_t),
    family = 'IPM',
    data_list = data_list,
    states = list(c('surf_area')),
    has_hier_effs = FALSE,
    evict = FALSE
  ) %>%
  define_impl(
    make_impl_args_list(
      kernel_names = c('P', "F", "K"),
      int_rule = rep('midpoint', 3),
      dom_start = rep('surf_area',3),
      dom_end = rep('surf_area', 3)
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
  make_ipm(usr_funs = list(inv_logit = inv_logit,
                           mvt_wrapper = mvt_wrapper),
           iterate = TRUE,
           iterations = 10)

pop_state_ipmr <- param_resamp_model$pop_state$pop_state_surf_area
lambda_ipmr <- numeric(iterations)

for(i in seq(2, dim(pop_state_ipmr)[2], 1)) {
  lambda_ipmr[(i - 1)] <- sum(pop_state_ipmr[ ,i]) / sum(pop_state_ipmr[ ,(i - 1)])
}

lambda_ipmr
```
