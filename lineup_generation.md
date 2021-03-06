Figures for USCOTS 2019 Poster on Visual Inference
================
Adam Loy
May 2019

Set up
------

``` r
library(Sleuth3)  # for data sets
library(dplyr)    # tools for data manipulation
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
library(infer)    # tools for simulation-based inference
library(ggplot2)  # plotting tools
library(nullabor) # lineup tools
library(broom)
```

Intro to permutation testing example
------------------------------------

``` r
set.seed(403)
orig_data <- cbind(case0101, replicate = 20)

writing_sim <- 
  case0101 %>%
  specify(Score ~ Treatment) %>% 
  hypothesize(null = "independence") %>%
  generate(reps = 19, type = "permute") %>%
  as_tibble() %>%
  bind_rows(orig_data) %>%
  mutate(.id = sample(20, size = 20, replace = FALSE) %>% rep(each = nrow(case0101)))

writing_sim_means <- writing_sim %>%
  group_by(.id, Treatment) %>%
  summarize(mean = mean(Score))

diff_means_plot <- ggplot(data = case0101) +
  geom_boxplot(mapping = aes(x = Treatment, y = Score, fill = Treatment), alpha = 0.5) +
  geom_point(data = case0101 %>% group_by(Treatment) %>% summarize(Score = mean(Score)), aes(x = Treatment, y = Score)) +
  scale_fill_viridis_d() +
  scale_color_viridis_d() +
  theme_bw() +
  # coord_flip() +
  theme(legend.position = "none")

ggsave(diff_means_plot, filename = "diff_means_plot.pdf", height = 2.5, width = 2.5)

permute_plot <- writing_sim %>%
  ggplot() +
  geom_boxplot(mapping = aes(x = Treatment, y = Score, fill = Treatment), alpha = 0.5) +
  geom_point(data = writing_sim_means, mapping = aes(x = Treatment, y = mean, group = Treatment)) +
  facet_wrap(~.id, ncol = 5) +
  scale_fill_viridis_d() +
  scale_color_viridis_d() +
  theme_bw() +
  # coord_flip() +
  theme(legend.position = "none")

ggsave(permute_plot, filename = "permute_lineup.pdf")
```

    ## Saving 7 x 5 in image

Linear regression diagnostics
-----------------------------

``` r
data("ex0826", package = "Sleuth3")
mod <- lm(log(Metab) ~ log(Mass), data = ex0826)
# mod <- lm(Yield ~ Rainfall + I(Rainfall^2), data = ex0915)
lm_lineup <- replicate(19, expr = simulate(mod), simplify = FALSE)
lm_lineup[[20]] <- data.frame(sim_1 = log(ex0826$Metab))

lm_lineup <- lapply(lm_lineup, FUN = function(x) {
  broom::augment(lm(x[[1]] ~ log(Mass), data = ex0826))
}) 

lm_lineup <- lm_lineup %>% 
  bind_rows() %>%
  mutate(.sample = rep(1:20, each = nrow(ex0826)),
         .id = sample(20, size = 20, replace = FALSE) %>% rep(., each = nrow(ex0826))) 

residual_lineup <- lm_lineup %>%
  ggplot(aes(x = .fitted, y = .resid)) +
  geom_hline(yintercept = 0, linetype = 2) +
  geom_point(shape = 1) +
  facet_wrap(~ .id) +
  labs(x = "Fitted values", y = "Residuals") +
  theme_bw()

ggsave(residual_lineup, filename = "residual_lineup.pdf")
```

    ## Saving 7 x 5 in image

``` r
observed_residual <- augment(mod) %>%
  ggplot(aes(x = .fitted, y = .resid)) +
  geom_hline(yintercept = 0, linetype = 2) +
  geom_point(shape = 1) +
  labs(x = "Fitted values", y = "Residuals") +
  theme_bw()

ggsave(observed_residual, filename = "observed_residual.pdf", height = 2, width = 2)
```

Interpreting Q-Q plots
----------------------

``` r
set.seed(1523613829)
n <- 30
obs_sim <- rchisq(n, df = 2) 
norm_sims <- lapply(1:19, function(x) rnorm(n))
qq_lineup <- data.frame(x = c(unlist(norm_sims), scale(obs_sim)), .sample = rep(1:20, each = n)) %>%
  mutate(.id = sample(20, size = 20, replace = FALSE) %>% rep(., each = n))

qqplot_lineup <- qq_lineup %>%
  ggplot(aes(sample = x)) +
  geom_qq() + 
  geom_qq_line(linetype = 2) +
  facet_wrap(~.id, ncol = 5) +
  theme_bw() + 
  labs(x = "N(0, 1) quantiles", y = "Sample quantiles")

ggsave(qqplot_lineup, filename = "qqplot_lineup.pdf")
```

    ## Saving 7 x 5 in image

``` r
n <- 10
obs_sim2 <- rt(n, df = 2) 
norm_sims2 <- lapply(1:19, function(x) rnorm(n))
qq_lineup2 <- data.frame(x = c(unlist(norm_sims2), obs_sim2), .sample = rep(1:20, each = n)) %>%
  mutate(.id = sample(20, size = 20, replace = FALSE) %>% rep(., each = n))

qq_lineup2 %>%
  ggplot(aes(sample = x)) +
  geom_qq() + 
  geom_qq_line(linetype = 2) +
  facet_wrap(~.id, ncol = 4) +
  theme_bw()  + 
  labs(x = "N(0, 1) quantiles", y = "Sample quantiles")
```

![](lineup_generation_files/figure-markdown_github/unnamed-chunk-5-1.png)

Logistic regression diagnositcs
-------------------------------

``` r
set.seed(354798)
data("Donner", package="vcdExtra")
blog_mod <- glm(survived ~ age + sex, data = Donner, family = binomial)

glm_lineup <- replicate(19, expr = simulate(blog_mod), simplify = FALSE)
glm_lineup[[20]] <- data.frame(sim_1 = Donner[["survived"]])

glm_lineup <- lapply(glm_lineup, FUN = function(x) {
  broom::augment(glm(x[[1]] ~ age + sex, data = Donner, family = binomial), type.residuals = "pearson")
}) 

glm_lineup <- glm_lineup %>% 
  bind_rows() %>%
  mutate(.sample = rep(1:20, each = nrow(Donner)),
         .id = sample(20, size = 20, replace = FALSE) %>% rep(., each = nrow(Donner))) 

logistic_residuals <- glm_lineup %>%
  ggplot(aes(x = .fitted, y = .resid)) +
  geom_hline(yintercept = 0, linetype = 2) +
  geom_point(shape = 1) +
  facet_wrap(~ .id) +
  labs(x = "Linear predictor", y = "Pearson residuals") +
  theme_bw()

ggsave(logistic_residuals, filename = "logistic_residuals.pdf")
```

    ## Saving 7 x 5 in image
