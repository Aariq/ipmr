
ipmr
====

Not yet functional. See below for an example of what the workflow will look like and open issues to suggest alternatives.

``` r
# Example of the API for a simple IPM without density dependence or environmental
# stochasticity

my_ipm <- init_ipm(class = 'simple-ndd-det') %>%
  add_kernel(
    # Name of the kernel
    name = "P",
    # The type of transition it describes (e.g. continuous - continuous, discrete - continuous)
    family = "IPM_CC",
    # The formula for the kernel. don't forget to transpose the growth matrix when multiplying survival
    formula = t(s * t(g)),
    # A named set of expressions for the vital rates it includes
    s = 1/(1 + exp(-(bs_0 + bs_1 * size_1))),
    g = dnorm(size_2, mean_g, sd_g),
    mean_g = bg_0 + bg_1 * size_1,
    # A list containing named values for each constant 
    data_list = list(bs_0 = 0.2, # could also be coef(my_surv_mod)[1]
                     bs_1 = 0.978,
                     bg_0 = 0.78,
                     bg_1 = 0.967,
                     sd_g = 3.4),
    # The state variable for time T
    domain_start = size_1,
    # The state variable for T+1. for IPM_CC's, this will almost always be the same 
    # as domain_start. Scroll down to see examples of when it isn't
    domain_end = size_2,
    # The type of eviction correction. Set to NA if you wish to correct on your own
    evict_type = 'truncated_distributions'
    ) %>%
  add_kernel(
    name = "F",
    family = "IPM_CC",
    formula = r_s * r_n * r_p,
    r_s = dnorm(size_2, recr_mean, recr_sd),
    r_n = exp(br_0 + br_1 * size_1),
    r_p = 1/(1 + exp(-(bp_0 + bp_1 * size_1))),
    data_list = list(recr_mean = 0.67,
                     recr_sd = 0.3,
                     br_0 = 2.9,
                     br_1 = 6.5,
                     bp_0 = 0.65,
                     bp_1 = 0.72),
    domain_start = size_1,
    domain_end = size_2,
    evict_type = 'truncated_distributions'
  ) %>%
  add_K(
    formula = P + F
  ) %>%
  make_ipm(
    # each element of make_domain_list takes lower, upper, and the number of meshpoints
    domain_list = make_domain_list(size = c(0.01, 100, 200)),
    # The rule to use for numerically approximating the integral
    integration_rule = "midpoint"
  )
```
