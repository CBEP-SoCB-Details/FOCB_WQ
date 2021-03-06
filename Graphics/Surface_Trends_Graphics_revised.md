Revised Graphics of Water Quality Trends from Friends of Casco Bay Data
================
Curtis C. Bohlen, Casco Bay Estuary Partnership
9/18/2021

-   [Introduction](#introduction)
-   [Load Libraries](#load-libraries)
-   [Part 1: Load and Organize Data](#part-1-load-and-organize-data)
    -   [Establish Folder Reference](#establish-folder-reference)
    -   [Primary Data](#primary-data)
        -   [Remove 2020 only data](#remove-2020-only-data)
        -   [Add Seasonal Factor](#add-seasonal-factor)
        -   [Address Secchi Censored
            Values](#address-secchi-censored-values)
        -   [Limit Chlorophyll to Three Long-Term
            Stations](#limit-chlorophyll-to-three-long-term-stations)
        -   [Select Parameters](#select-parameters)
    -   [Create Trend Data](#create-trend-data)
    -   [Construct Nested Tibble](#construct-nested-tibble)
-   [Part 2: Modeling](#part-2-modeling)
    -   [Models](#models)
    -   [Deriving Annotations from general
        results.](#deriving-annotations-from-general-results)
    -   [Marginal Means](#marginal-means)
        -   [Limit Forecasts Chlorophyll](#limit-forecasts-chlorophyll)
-   [Part 3: Graphics](#part-3-graphics)
    -   [Facet Graphics](#facet-graphics)
        -   [Data Set up](#data-set-up)
        -   [Fixing the Chlorophyll Y
            Axis](#fixing-the-chlorophyll-y-axis)
    -   [Assembling the Final Graphics](#assembling-the-final-graphics)
        -   [Base Graphic](#base-graphic)
        -   [Define Annotations](#define-annotations)
        -   [Original Annotation
            Placement](#original-annotation-placement)
        -   [Revised Annotation
            Placement](#revised-annotation-placement)

<img
    src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
    style="position:absolute;top:10px;right:50px;" />

# Introduction

This Notebook analyzes FOCB’s “Surface” data. These data are pulled from
long term monitoring locations around the Bay.

This notebook includes code only for developing the final revisions of
graphics for the final draft layout of the State of Casco Bay chapter on
bay water quality.

# Load Libraries

``` r
library(tidyverse)
#> Warning: package 'tidyverse' was built under R version 4.0.5
#> -- Attaching packages --------------------------------------- tidyverse 1.3.1 --
#> v ggplot2 3.3.5     v purrr   0.3.4
#> v tibble  3.1.4     v dplyr   1.0.7
#> v tidyr   1.1.3     v stringr 1.4.0
#> v readr   2.0.1     v forcats 0.5.1
#> Warning: package 'ggplot2' was built under R version 4.0.5
#> Warning: package 'tibble' was built under R version 4.0.5
#> Warning: package 'tidyr' was built under R version 4.0.5
#> Warning: package 'readr' was built under R version 4.0.5
#> Warning: package 'dplyr' was built under R version 4.0.5
#> Warning: package 'forcats' was built under R version 4.0.5
#> -- Conflicts ------------------------------------------ tidyverse_conflicts() --
#> x dplyr::filter() masks stats::filter()
#> x dplyr::lag()    masks stats::lag()
library(lemon)   # A few enhancements for ggplot graphics.
#> Warning: package 'lemon' was built under R version 4.0.5
#> 
#> Attaching package: 'lemon'
#> The following object is masked from 'package:purrr':
#> 
#>     %||%
#> The following objects are masked from 'package:ggplot2':
#> 
#>     CoordCartesian, element_render
library(readxl)

library(mgcv)     # For `gam()` models, used here for hierarchical models
#> Loading required package: nlme
#> 
#> Attaching package: 'nlme'
#> The following object is masked from 'package:dplyr':
#> 
#>     collapse
#> This is mgcv 1.8-36. For overview type 'help("mgcv-package")'.
library(emmeans)
#> Warning: package 'emmeans' was built under R version 4.0.5
library(ggpmisc)  # Allows absolute positioning of annotations and text
#> Warning: package 'ggpmisc' was built under R version 4.0.5
#> Loading required package: ggpp
#> Warning: package 'ggpp' was built under R version 4.0.5
#> 
#> Attaching package: 'ggpp'
#> The following object is masked from 'package:ggplot2':
#> 
#>     annotate
library(CBEPgraphics)
load_cbep_fonts()
theme_set(theme_cbep())
```

# Part 1: Load and Organize Data

## Establish Folder Reference

``` r
sibfldnm <- 'Original_Data'
parent   <- dirname(getwd())
sibling  <- file.path(parent,sibfldnm)

dir.create(file.path(getwd(), 'figures'), showWarnings = FALSE)
```

## Primary Data

We specify column names because FOCB data has a row of names, a row of
units, then the data. This approach is simpler than reading names from
the first row and correcting them to be R syntactic names.

``` r
fn    <- 'FOCB Surface All Current Sites With BSV Data.xlsx'
fpath <- file.path(sibling,fn)

mynames <- c('station', 'dt', 'time', 'sample_depth',
             'secchi', 'water_depth','temperature', 'salinity',
             'do', 'pctsat', 'pH', 'chl', 
             'month', 'year', 'fdom', 'bga', 
             'turbidity', 'blank', 'clouds', 'wndspd',
             'winddir'
             ) 

the_data <- read_excel(fpath, skip=2, col_names = mynames) %>%
  mutate(month = factor(month, levels = 1:12, labels = month.abb)) %>%
  relocate(month, year, .after = dt)

rm(mynames)
```

### Remove 2020 only data

``` r
the_data <- the_data %>%
select(-c(fdom:winddir))
```

### Add Seasonal Factor

``` r
the_data <- the_data %>%
  mutate(season_3 =  fct_collapse(month, 
                                 Spring = c('Apr', 'May'), 
                                 Summer = c('Jun','Jul', 'Aug'),
                                 Fall   =  c('Sep', 'Oct'))) %>%
  relocate(season_3, .after = month)
```

### Address Secchi Censored Values

``` r
the_data <- the_data %>%
  mutate(secchi_2 = if_else(secchi == "BSV", water_depth, as.numeric(secchi)),
         bottom_flag = secchi == "BSV") %>%
  relocate(secchi_2, .after = secchi) %>%
  relocate(bottom_flag, .after = year)
#> Warning in replace_with(out, !condition, false, fmt_args(~false), glue("length
#> of {fmt_args(~condition)}")): NAs introduced by coercion
```

### Limit Chlorophyll to Three Long-Term Stations

``` r
the_data <- the_data %>%
  mutate(chl = if_else(station %in% c('P5BSD', 'P6FGG', 'P7CBI'),
                                   chl, NA_real_)) %>%
  mutate(chl_log1p = log1p(chl))
```

### Select Parameters

We will plot six parameters, omitting percent saturation, and plotting
chlorophyll on a transformed y axis.

``` r
the_data <- the_data %>%
  select(-chl, -pctsat)
```

## Create Trend Data

First, we create a tibble containing information on years in which each
station was sampled.

``` r
years_data <- the_data %>%
  group_by(station, year) %>%
  summarize(yes = ! all(is.na(temperature)),
            .groups = 'drop_last') %>%
  summarize(years = sum(yes, na.rm = TRUE),
            recent_years =  sum(yes & year > 2014, na.rm = TRUE),
            .groups = 'drop')
```

Then we identify stations with at least 10 years of data, and at least
three years of data from the last five years, and use that list to
select data for trend analysis. Finally, we adjust the levels in the
`station` and `station_name` variables.

``` r
selected_stations <- years_data %>%
  filter(years> 9, recent_years >2) %>%
  pull(station)

trend_data <- the_data %>%
  filter(station %in% selected_stations) %>%
  mutate(station = fct_drop(station)) %>%
  mutate(station = fct_reorder(station, temperature, mean, na.rm = TRUE)) 
rm(selected_stations, years_data)
rm(the_data)
```

``` r
length(unique(trend_data$station))
#> [1] 17
```

We are reduced to only 17 stations with long-term records for trend
analysis. We noted above that we have limited chlorophyll data before
the last couple of years. We address that momentarily.

## Construct Nested Tibble

``` r
units <- tibble(parm = c('temperature', 'salinity', 'do', 
                          'pH', 'secchi_2', 'chl_log1p'),
                label = c("Temperature","Salinity", "Dissolved Oxygen",
                         "pH", "Secchi Depth", "Chlorophyll A"),
                units = c(paste0("\U00B0", "C"),'PSU', 'mg/l','','m',
                          'ug/l'))

nested_data <- trend_data %>%
  select(-dt, -time, -sample_depth, 
         -secchi) %>%
  relocate(water_depth, .after = month) %>%
  pivot_longer(c(secchi_2:chl_log1p), 
               names_to = 'parm', 
               values_to = 'value') %>%
  mutate(bottom_flag = if_else(parm=='secchi_2', bottom_flag, FALSE)) %>%
  filter(! is.na(value)) %>%

  group_by(parm) %>%
  nest() %>%
  left_join(units, by = 'parm') %>%
   mutate(parm = factor(parm,
                       levels = c('temperature', 'salinity', 'do', 
                                  'pH', 'secchi_2', 'chl_log1p')))
```

# Part 2: Modeling

We treat stations as random exemplars of possible stations, and thus
rely on hierarchical models. We use a GAM model with a random factor
smoothing term. We restrict ourselves to linear trends by year.

The seasonal models were motivated by two dimensional tensor smooth GAM
models developed in “Surface\_Analysis\_Trends.Rmd”. We look at other
interaction models in “Surface\_Seasonal\_Trends.Rmd”

## Models

``` r
nested_data <- nested_data %>%
  mutate(lmer_s = map(data, function(df) gam(value ~ year * season_3 + 
                                              s(station, bs = 're'), 
                                            data = df)))
names(nested_data$lmer_s) <- nested_data$parm
```

``` r
nested_data <- nested_data %>%
  mutate(s_three_slopes = map(lmer_s,
                      function(mod) summary(emtrends(mod, 
                                                     ~ season_3, 
                                                     var = "year"))))
names(nested_data$s_three_slopes) <- nested_data$parm
```

We also want to identify which seasons have significant slopes. We can
figure that out (approximately) by checking if sign of the upper and
lower 95% confidence intervals are the same or not.

``` r
which_sig <- function(upper, lower) {
  sig_list <- which(sign(upper) == sign(lower))
  c('Spring', 'Summer', 'Fall')[sig_list]
}

nested_data <- nested_data %>%
  mutate(seas_change = map(s_three_slopes,
                      function(df) which_sig(df$upper.CL, df$lower.CL)))
names(nested_data$seas_change) <- nested_data$parm
```

## Deriving Annotations from general results.

We want to say the following: \* Temperatures: Highlight overall trend.
Interaction is significant, but small.  
\* “Increasing \~ 0.56 degrees C per decade”  
\* Salinity: ANOVA not significant. Possible weak summer trend.  
\* “No trend”  
\* DO: No significant terms by ANOVA, so don’t want to interpret
marginal means.  
\* “No trend”  
\* pH: Highlight different seasonal patterns that cancel each other
out.  
\* “Spring increase, summer decrease”.  
\* Secchi: Show seasonal data and highlight seasonal trend, with
declining. water quality in summer and fall, but little change in
spring.  
\* “Decrease Summer and Fall”  
\* Chlorophyll: Overall trend, but strong seasonal differences.  
\* “Decrease in spring and summer”

## Marginal Means

These means are all marginal to the identity of the stations. It’s worth
noting that differences in slope that jump out in these graphics often
vanish into relative insignificance when plotted against the source
data. Also, some “statistically significant” differences in seasonal
slopes are pretty small, on pretty small slopes, and thus are probably
not worth interpreting.

Here we use the `emmip()` function, which produces an interaction plot,
but suppress the plots, as we want to add related data to plots of our
own.

``` r
nested_data <- nested_data %>%
  mutate(emmi_3 = map(lmer_s, function(mod) emmip(mod, season_3 ~ year,
                                                         at = list(year = 1993:2020),
                                                         plotit = FALSE)))
```

### Limit Forecasts Chlorophyll

The code we just ran generated “predictions” for chlorophyll for years
in which we actually have no chlorophyll data. We could run `emmip()`
again, but it is also easy to simply select the right rows from the
existing predictions.

``` r
row <- nested_data[nested_data$parm == 'chl_log1p',]
preds <- row$emmi_3[[1]]
preds <- preds %>% filter(year > 2000)

nested_data$emmi_3[nested_data$parm == 'chl_log1p'] <- list(preds)
```

# Part 3: Graphics

## Facet Graphics

We want to generate facet graphics rather than individual graphics. The
facet graphics should mimic the organization of the graphics showing
recent conditions. That means we want to order parameters in the same
way, and use the same `log(x+1)` transformed axis for Chlorophyll.

Facet graphics are built off of data frames with an indicator variable
that assigns data to facets. So this will require rethinking the
organization of data. We need to be able to plot both raw data and
prediction lines. That means we will assemble facet graphics from two
data frames, one of data, and one of predictions.

### Data Set up

``` r
list_df <- nested_data$data
names(list_df) <-  nested_data$parm
data_sel <- bind_rows(list_df, .id = 'parm') %>%
  filter(parm %in% c('temperature', 'salinity', 'do', 
                          'pH', 'secchi_2', 'chl_log1p')) %>%
  mutate(parm = factor(parm,
                            levels = c('temperature', 'salinity', 'do', 
                                       'pH', 'secchi_2', 'chl_log1p'))) %>%
  mutate(simple_lab = factor(parm,
                            levels = c('temperature', 'salinity', 'do', 
                                       'pH', 'secchi_2', 'chl_log1p'),
                            labels = c("Temperature (°C)", 
                                       'Salinity (PSU)', 
                                       'Dissolved Oxygen (mg/l)',
                                       'pH',
                                       'Secchi Depth (m)',
                                       'Chlorophyll A (ug/l)'))) %>%
  mutate(full_lab = factor(parm, 
                            levels =  c("temperature", 'salinity', 'do', 
                                        'pH', 'secchi_2', 'chl_log1p'),
                            labels = c(expression("Temperature (" * degree * "C)"), 
                                       expression('Salinity' ~ '(PSU)'), 
                                       expression('Dissolved' ~ 'Oxygen' ~ '(mg/l)'),
                                       expression('pH'),
                                       expression('Secchi' ~ 'Depth' ~ '(m)'),
                                       expression('Chlorophyll A (' * mu * 'g/l)'))))

list_df <- nested_data$emmi_3
names(list_df) <-  nested_data$parm
predicts_sel <- bind_rows(list_df, .id = 'parm') %>%
  filter(parm %in% c('temperature', 'salinity', 'do', 
                          'pH', 'secchi_2', 'chl_log1p')) %>%
  mutate(parm = factor(parm,
                            levels = c('temperature', 'salinity', 'do', 
                                       'pH', 'secchi_2', 'chl_log1p'))) %>%
  mutate(simple_lab = factor(parm,
                            levels = c('temperature', 'salinity', 'do', 
                                       'pH', 'secchi_2', 'chl_log1p'),
                            labels = c("Temperature (°C)", 
                                       'Salinity (PSU)', 
                                       'Dissolved Oxygen (mg/l)',
                                       'pH',
                                       'Secchi Depth (m)',
                                       'Chlorophyll A (ug/l)'))) %>%
  mutate(full_lab = factor(parm, 
                            levels =  c("temperature", 'salinity', 'do', 
                                        'pH', 'secchi_2', 'chl_log1p'),
                            labels = c(expression("Temperature (" * degree * "C)"), 
                                       expression('Salinity' ~ '(PSU)'), 
                                       expression('Dissolved' ~ 'Oxygen' ~ '(mg/l)'),
                                       expression('pH'),
                                       expression('Secchi' ~ 'Depth' ~ '(m)'),
                                       expression('Chlorophyll A (' * mu * 'g/l)'))))
```

### Fixing the Chlorophyll Y Axis

The Chlorophyll values we model and plot are actually a transformation
of the  
raw chlorophyll observations. As such, the values plotted are
meaningless to potential readers. We need to provide an axis that
indicates the non-linearity of the Y axis.

However, it **is** possible to define axis breaks and labels using
functions.  
We can create a function to produce appropriate breaks and another to
provide labels for the Chlorophyll plot, if only we can construct a
function that provides different values depending on the sub-plot we are
in.

We can tell which sub-plot we are in (here….) by looking at the axis
limits, which are passed as the argument to the `breaks` function.

Luckily, in our use case, the upper axis limit for the (transformed)
chlorophyll data is lower than any of the others. The upper limit is
around 5, while the next lowest maximum figure is well over 5.

A similar problem (and solution) faces setting the labels for the
breaks. but here, since we will have SET the breaks, we can look to see
if we have those specific breaks, and if so, set the labels
appropriately. It’s slightly trickier than that, but that is the basic
concept.

#### Create Breaks Function

``` r
preferred_breaks =  c(0, 1,  5, 10, 50)

my_breaks_fxn <- function(lims) {
  #browser()
  if(max(lims) < 6) {
    # Then we're looking at our transformed Chl data
  a <- preferred_breaks
    return(log(a +1))
  }
  else {
    # Return the default breaks, by calling the default breaks fxn
    return(labeling::extended(lims[[1]], lims[[2]], 5))
  }
}
```

#### Create Label Function

We are cheating a bit here by plotting transformed data, but providing
**labels** that are back transformed. That work is conducted by the
labeling function.

``` r
my_label_fxn <- function(brks) {
  # frequently, brks contians NA in place of one or more 
  # of the candidate brks, even after we pass a vector of specific breaks.
  # In particular, "pretty" breaks outside the range of the data
  # are dropped and replaced with NA.
  a <- preferred_breaks
  b <- round(log(a+1), 3)
  real_breaks = round(brks[! is.na(brks)], 3)
  if (all(real_breaks %in% b)) {
    # then we have our transformed Chl data
    return(a)
  }
  else {
    return(brks)
  }
}
```

## Assembling the Final Graphics

### Base Graphic

``` r
p_jit <- ggplot(data_sel, aes(x = year, y = value)) +
    geom_jitter(aes(color = season_3), width = 0.3, height = 0, 
                alpha = 0.1) +
    geom_line(data = predicts_sel, 
             mapping = aes(x = year, y = yvar, color = tvar),
             size = 1) +
    xlab('') +
    ylab('') +

    scale_y_continuous (breaks = my_breaks_fxn, labels = my_label_fxn) +
  
    theme_cbep(base_size = 12) +
    theme(axis.text.x = element_text(size = 10),
          legend.position = 'bottom')
```

### Define Annotations

``` r
annotations <- c(paste0(" +0.56","\U00B0", "C per decade"),
                 "No trend",
                 "No trend",
                 "Spring increase, summer decrease",
                 "Decrease summer and fall",
                 "Decrease spring and summer")
nested_data$annot <- annotations
```

### Original Annotation Placement

Most annotations can go in upper right corner. We start with that, then
modify as needed.

``` r
ann_xloc <- rep('left', length(nested_data$parm))
ann_yloc <- rep('bottom', length(nested_data$parm))

ann_xloc[1] <- 'right'  # temperature
ann_xloc[5] <- 'right'  # Secchi
ann_yloc[5] <- 'top'    # Secchi
ann_xloc[6] <- 'right'  # Chlorophyll
ann_yloc[6] <- 'top'    # Chlorophyll

annot <- tibble(full_lab = factor(levels(data_sel$full_lab), 
                              levels = levels(data_sel$full_lab)),
                simple_lab = factor(levels(data_sel$simple_lab), 
                              levels = levels(data_sel$simple_lab)),
                annot = annotations, 
                ann_xloc = ann_xloc, ann_yloc = ann_yloc)
```

#### Original Figure

``` r
p1 <- p_jit +
  scale_color_manual(values = cbep_colors2()[c(1,2,4)],
                     name = '',
                     guide = guide_legend(override.aes = list(alpha = 1))) +
  facet_wrap(~full_lab, nrow = 6,
               scale = 'free_y',
               labeller = label_parsed) +
  geom_text_npc(data = annot, 
            mapping = aes(npcx = ann_xloc, npcy = ann_yloc, label = annot),
            hjust = 'inward',
            size = 3.25) +
  theme(axis.ticks = element_line(size = .5))
p1
```

<img src="Surface_Trends_Graphics_revised_files/figure-gfm/original-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/trends_original.pdf', device = cairo_pdf, width = 3.5, height = 9.5)
```

\#\#\#\#Original Figure with Tick Marks Package ‘lemon’ offers an
alternative, where we can keep tick marks.

``` r
p1a <- p_jit+
  scale_color_manual(values = cbep_colors2()[c(1,2,4)],
                     name = '',
                     guide = guide_legend(override.aes = list(alpha = 1))) +
  facet_rep_wrap(~full_lab, nrow = 6,
               scales = 'free_y',
               labeller = label_parsed) +
  geom_text_npc(data = annot, 
            mapping = aes(npcx = ann_xloc, npcy = ann_yloc, label = annot),
            hjust = 'inward',
            size = 3.25) +
  theme(axis.ticks = element_line(size = .5))
p1a
```

<img src="Surface_Trends_Graphics_revised_files/figure-gfm/color_annot_tick-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/trends_original_tick.pdf', device = cairo_pdf, width = 3.5, height = 9.5)
```

#### Revised Colors

``` r
p2 <- p_jit +
  scale_color_manual(values = cbep_colors2()[c(1,5,2)],
                     name = '',
                     guide = guide_legend(override.aes = list(alpha = 1))) +
  facet_wrap(~full_lab, nrow = 6,
               scale = 'free_y',
               labeller = label_parsed) +
  geom_text_npc(data = annot, 
            mapping = aes(npcx = ann_xloc, npcy = ann_yloc, label = annot),
            hjust = 'inward',
            size = 3.25)+
  theme(axis.ticks = element_line(size = .5))

p2
```

<img src="Surface_Trends_Graphics_revised_files/figure-gfm/revised_colors-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/trends_revised_colors.pdf', device = cairo_pdf, width = 3.5, height = 9.5)
```

#### Revised Colors with Tick Marks

``` r
p2a <- p_jit+
  scale_color_manual(values = cbep_colors2()[c(1,5,2)],
                     name = '',
                     guide = guide_legend(override.aes = list(alpha = 1))) +
  facet_rep_wrap(~full_lab, nrow = 6,
               scales = 'free_y',
               labeller = label_parsed) +
  geom_text_npc(data = annot, 
            mapping = aes(npcx = ann_xloc, npcy = ann_yloc, label = annot),
            hjust = 'inward',
            size = 3.25)+
  theme(axis.ticks = element_line(size = .5))
p2a
```

<img src="Surface_Trends_Graphics_revised_files/figure-gfm/revised_color_tick-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/trends_revised_color_tick.pdf', device = cairo_pdf, width = 3.5, height = 9.5)
```

### Revised Annotation Placement

``` r
ann_xloc <- rep('left', length(nested_data$parm))
ann_yloc <- rep('bottom', length(nested_data$parm))

ann_xloc[1] <- 'right'  # temperature
#ann_xloc[5] <- 'right'  # Secchi
#ann_yloc[5] <- 'top'    # Secchi
ann_xloc[6] <- 'right'  # Chlorophyll
ann_yloc[6] <- 'top'    # Chlorophyll

annot <- tibble(full_lab = factor(levels(data_sel$full_lab), 
                              levels = levels(data_sel$full_lab)),
                simple_lab = factor(levels(data_sel$simple_lab), 
                              levels = levels(data_sel$simple_lab)),
                annot = annotations, 
                ann_xloc = ann_xloc, ann_yloc = ann_yloc)
```

#### Figure with Revised Annotation Placement

``` r
p3 <- p_jit +
  scale_color_manual(values = cbep_colors2()[c(1,2,4)],
                     name = '',
                     guide = guide_legend(override.aes = list(alpha = 1))) +
  facet_wrap(~full_lab, nrow = 6,
               scale = 'free_y',
               labeller = label_parsed) +
  geom_text_npc(data = annot, 
            mapping = aes(npcx = ann_xloc, npcy = ann_yloc, label = annot),
            hjust = 'inward',
            size = 3.25)+
  theme(axis.ticks = element_line(size = .5))
p3
```

<img src="Surface_Trends_Graphics_revised_files/figure-gfm/revised_annot-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/trends_revised_annot.pdf', device = cairo_pdf, width = 3.5, height = 9.5)
```

#### Figure with Revised Annotation Placement and Ticks

``` r
p3a <- p_jit+
  scale_color_manual(values = cbep_colors2()[c(1,2,4)],
                     name = '',
                     guide = guide_legend(override.aes = list(alpha = 1))) +
  facet_rep_wrap(~full_lab, nrow = 6,
               scales = 'free_y',
               labeller = label_parsed) +
  geom_text_npc(data = annot, 
            mapping = aes(npcx = ann_xloc, npcy = ann_yloc, label = annot),
            hjust = 'inward',
            size = 3.25)+
  theme(axis.ticks = element_line(size = .5))
p3a
```

<img src="Surface_Trends_Graphics_revised_files/figure-gfm/revised_annot_tick-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/trends_revised_annot_tick.pdf', device = cairo_pdf, width = 3.5, height = 9.5)
```

#### Figure with Revised Colors and Annotation Placement

``` r
p4<- p_jit +
  scale_color_manual(values = cbep_colors2()[c(1,5,2)],
                     name = '',
                     guide = guide_legend(override.aes = list(alpha = 1))) +
  facet_wrap(~full_lab, nrow = 6,
               scale = 'free_y',
               labeller = label_parsed) +
  geom_text_npc(data = annot, 
            mapping = aes(npcx = ann_xloc, npcy = ann_yloc, label = annot),
            hjust = 'inward',
            size = 3.25)+
  theme(axis.ticks = element_line(size = .5))
p4
```

<img src="Surface_Trends_Graphics_revised_files/figure-gfm/revised_color_annot-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/trends_revised_colors_annot.pdf', device = cairo_pdf, width = 3.5, height = 9.5)
```

#### Figure with Revised Colors, Annotation Placement and Ticks

``` r
p4a <- p_jit+
  scale_color_manual(values = cbep_colors2()[c(1,5,2)],
                     name = '',
                     guide = guide_legend(override.aes = list(alpha = 1))) +
  facet_rep_wrap(~full_lab, nrow = 6,
               scales = 'free_y',
               labeller = label_parsed) +
  geom_text_npc(data = annot, 
            mapping = aes(npcx = ann_xloc, npcy = ann_yloc, label = annot),
            hjust = 'inward',
            size = 3.25)+
  theme(axis.ticks = element_line(size = .5))
p3a
```

<img src="Surface_Trends_Graphics_revised_files/figure-gfm/revised_colors_annot_tick-1.png" style="display: block; margin: auto;" />

``` r
ggsave('figures/trends_revised_colors_annot_tick.pdf', device = cairo_pdf, width = 3.5, height = 9.5)
```

#### Original Figure with Grid Lines (Not Run)

``` r
p1+
theme(panel.grid.major.x = element_line(size = 0.5, color = 'gray75'),
      axis.ticks.x = element_line(size = 0.5, color = 'gray75')
      )
ggsave('figures/trends_original_grid.pdf', device = cairo_pdf, width = 3.5, height = 9.5)
```
