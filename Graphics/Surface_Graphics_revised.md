Revised Graphics for Surface Data from Casco Bay Monitoring
================
Curtis C. Bohlen, Casco Bay Estuary Partnership
9/18/2021

-   [Introduction](#introduction)
-   [Load Libraries](#load-libraries)
-   [Part 1: Load and Organize Data](#part-1-load-and-organize-data)
    -   [Establish Folder Reference](#establish-folder-reference)
    -   [Primary Data](#primary-data)
        -   [Remove 2020 only data](#remove-2020-only-data)
    -   [Add Station Names](#add-station-names)
    -   [Address Secchi Censored
        Values](#address-secchi-censored-values)
    -   [Create Recent Data](#create-recent-data)
    -   [Create Labels Tibble](#create-labels-tibble)
    -   [Reorganize and Select Data](#reorganize-and-select-data)
-   [Part 2: Faceted Graphics](#part-2-faceted-graphics)
-   [Examples](#examples)
    -   [Fixing the Chlorophyll Y Axis](#fixing-the-chlorophyll-y-axis)
    -   [Jitter Plot](#jitter-plot)

<img
    src="https://www.cascobayestuary.org/wp-content/uploads/2014/04/logo_sm.jpg"
    style="position:absolute;top:10px;right:50px;" />

# Introduction

This Notebook analyzes FOCB’s “Surface” data. These data are pulled from
long term monitoring locations around the Bay. Data, as suggested by the
name of the data subset, is collected at the surface of the water.
(Other data, from depth, are also available, but not evaluated here.)

This notebook includes code only for developing final revisions of
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
library(readxl)

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

the_data <- read_excel(fpath, skip=2, col_names = mynames)
rm(mynames)
```

### Remove 2020 only data

``` r
the_data <- the_data %>%
select(-c(fdom:winddir))
```

## Add Station Names

``` r
sibfldnm <- 'Derived_Data'
parent   <- dirname(getwd())
sibling  <- file.path(parent,sibfldnm)

fn    <- 'FOCB Monitoring Sites SHORT NAMES.xlsx'
fpath <- file.path(sibling,fn)
loc_data <- read_excel(fpath) %>%
  select(Station_ID, Alt_Name) %>%
  rename(station = Station_ID,
         station_name = Alt_Name)

the_data <- the_data %>%
  left_join(loc_data, by = 'station') %>%
  relocate(station_name, .after = station) %>%
  
  relocate(year, .after = dt) %>%
  relocate(month, .after = year)
rm(loc_data)
```

Our data contains two stations that are not associated with locations
that were included in our spatial data. Neither matter for the current
analysis. They will get filtered out when we select data to describe
recent conditions.

## Address Secchi Censored Values

Note we do NOT use maximum likelihood estimators of censored Secchi
depth, as censored values are relatively rare, and do not affect median
values, which are our focus here. The key goal is to include
observations with Secchi Depth on the bottom, and include information on
when the Secchi Depth may be biased.

``` r
the_data <- the_data %>%
  mutate(secchi_2 = if_else(secchi == "BSV", water_depth, as.numeric(secchi)),
         bottom_flag = secchi == "BSV") %>%
  relocate(secchi_2, .after = secchi) %>%
  relocate(bottom_flag, .after = sample_depth)  # we want an ID column
#> Warning in replace_with(out, !condition, false, fmt_args(~false), glue("length
#> of {fmt_args(~condition)}")): NAs introduced by coercion
```

Warnings are caused because some entries in Secchi that could not be
converted to numeric values. However, the warnings appear to have all
been triggered by existing NAs, not problems importing numeric values.
(Not shown.)

## Create Recent Data

In 2015, we presented marginal means and standard errors for sixteen
different regions of the Bay. This time, we have fewer monitoring
stations, and choose to present results for each monitoring location
individually.

We filter to the last five FULL years of data, 2015 through 2019, and
add a transformed chlorophyll value, to facilitate plotting.

``` r
recent_data <- the_data %>%
  filter(year > 2014 & year < 2020) %>%
  mutate(station = fct_reorder(station, temperature, median, na.rm = TRUE),
         station_name = fct_reorder(station_name, 
                                    temperature, median, na.rm = TRUE)) %>%
  mutate(chl_log1p = log1p(chl))
rm(the_data)
```

## Create Labels Tibble

To run parallel analyses on nested tibbles, we will need to reorganize
the data so that we can analyze along parameters. We add a list of
labels and measurement units to simplify later labeling of plots.

c(‘temperature’, ‘salinity’, ‘do’,‘pH’, ‘secchi\_2’, ‘chl\_log1p’))

``` r
units <- tibble(parm = c('temperature', 
                              'salinity', 'do',
                              'pctsat', 'pH', 'secchi_2', 
                              'chl', 'chl_log1p'),
                label = c( "Temperature",
                         "Salinity", "Dissolved Oxygen",
                         "Percent Saturation", "pH", "Secchi Depth",
                         "Chlorophyll A", "Chlorophyll A"),
                units = c(paste0("\U00B0", "C"),
                          'PSU', 'mg/l',
                          '', '', 'm', 
                          'ug/l', 'ug/l'),
                fancy_label =  c(
                  expression("Temperature (" * degree * "C)"),
                  expression("Salinity" ~ "(PSU)"),
                  expression("Dissolved Oxygen" ~ "(mg/l)"),
                  expression("Percent" ~ " Saturation"), 
                  expression("pH"),
                  expression("Secchi" ~ "Depth" ~ "(m)"), 
                  expression("Chlorophyll A (" * mu * "g/l)"),
                  expression("Chlorophyll A (" * mu * "g/l)")))
```

## Reorganize and Select Data

This is perhaps counter intuitive, because we “unnest” the nested data
here. An alternative would be to start from `recent_data` and pivot to
long form, but this ensures we are looking at the exact same data used
in our other plots.

``` r
data_long <- recent_data %>%
  select(-dt, -year, -time, -sample_depth, -secchi) %>%
  relocate(water_depth, .after = month) %>%
  pivot_longer(c(secchi_2:chl_log1p), 
               names_to = 'parm', 
               values_to = 'value') %>%
  mutate(bottom_flag = if_else(parm=='secchi_2', bottom_flag, FALSE)) %>%
  filter(! is.na(value)) %>%
  filter(parm %in% c('temperature', 'salinity', 'do', 
                          'pH', 'secchi_2', 'chl_log1p')) %>%
  mutate(parm = factor(parm,
                            levels = c('temperature', 'salinity', 'do', 
                                       'pH', 'secchi_2', 'chl_log1p'))) %>%
  mutate(fancy_parm = factor(parm, levels = units$parm,
                             labels = units$fancy_label))
```

# Part 2: Faceted Graphics

It would be nice to wrap those graphics up in a faceted format,
principally to simplify axis labels.

We need to supress some legend entries, and only show an entry for
“secchi on bottom”. Interestingly, since we first developed these
graphics, we updated `ggplot2` several times, and it appears the updates
altered the behavior of `scale_color_manual()`. Luckily the desired
behavior can be generated with `scale_color_discrete()`.

# Examples

Note we use `type = cbep_colors()` wit
h`scale_color_discrete(), not`values =
cbep\_colors()`, as we did with`scale\_color\_manual()\`.

``` r
test <- ggplot(mtcars) +
  geom_point(aes(wt, mpg,colour = factor(cyl)), size = 3)
test +
  scale_color_discrete(breaks=c(4,8), 
                       name="Cylinders",
                       type = cbep_colors())
test +
  scale_color_manual(breaks=c(4,6,8),
                     name="Cylinders",
                     values = cbep_colors())

# But when we omit something from breaks, it gets replaced with the NA color
test +
  scale_color_manual(breaks=c(4,8), 
                     name="Cylinders",
                     values = cbep_colors())
```

<img src="Surface_Graphics_revised_files/figure-gfm/examples-1.png" width="30%" style="display: block; margin: auto;" /><img src="Surface_Graphics_revised_files/figure-gfm/examples-2.png" width="30%" style="display: block; margin: auto;" /><img src="Surface_Graphics_revised_files/figure-gfm/examples-3.png" width="30%" style="display: block; margin: auto;" />

## Fixing the Chlorophyll Y Axis

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

#### Create Functions

``` r
prefered_breaks =  c(0, 1, 5, 10, 50)

my_breaks_fxn <- function(lims) {
  #browser()
  if(max(lims) < 5.1) {
    # Then we're looking at our transformed Chl data
  a <- prefered_breaks
    return(log(a +1))
  }
  else {
    return(labeling::extended(lims[[1]], lims[[2]], 5))
  }
}
```

We are cheating a bit here by plotting transformed data, but providing
labels that are back transformed. That work is conducted by the labeling
function.

``` r
my_label_fxn <- function(brks) {
  # frequently, brks is passed with NA in place of one or more 
  # of the candidate brks, even after I pass a vector of breaks.
  # In particular, "pretty" breaks outside the range of the data
  # are dropped and replaced with NA.
  a <- prefered_breaks
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

## Jitter Plot

``` r
p_jit <- ggplot(data_long, aes(x = station_name, y = value)) +
  geom_jitter(aes(color = bottom_flag), width = 0.3, height = 0, 
              alpha = 0.15) +
  stat_summary(fun.data = function(.x) median_hilow(.x, conf.int = .5),
               fill = cbep_colors()[3], color = "black",
               size = .4, shape = 22) +
  xlab('') +
  ylab('') +
  
  scale_y_continuous (breaks = my_breaks_fxn, labels = my_label_fxn) +
  
  theme_cbep(base_size = 12) +
  theme(axis.text.x = element_text(angle = 90,
                                   size = 8,
                                   hjust = 1,
                                   vjust = 0),
        axis.ticks.length.x = unit(0, 'cm')) +
  theme(legend.position = 'bottom')
```

``` r
p_jit +
  scale_color_discrete(type = cbep_colors()[c(1,4)], 
                     name = '', breaks = 'TRUE', label = 'Secchi On Bottom') +
  guides(color = guide_legend(override.aes = list(size = 3, alpha = 1))) +
  facet_wrap(~fancy_parm, nrow = 6,
             scale = 'free_y',
             labeller = label_parsed)
```

<img src="Surface_Graphics_revised_files/figure-gfm/jitter_colors, -1.png" style="display: block; margin: auto;" />

``` r
 ggsave('figures/surface_6_jitter_long_revised.pdf', device = cairo_pdf, 
        width = 3.5, height = 9.5)
```

``` r
p_jit +
  scale_color_discrete(type = cbep_colors()[c(1,1)]) +
  theme(legend.position = "none") +
  
  facet_wrap(~fancy_parm, nrow = 6,
             scale = 'free_y',
             labeller = label_parsed)
```

<img src="Surface_Graphics_revised_files/figure-gfm/jitter_no_colors, -1.png" style="display: block; margin: auto;" />

``` r
 ggsave('figures/surface_6_jitter_long_revised_no_color.pdf', device = cairo_pdf, 
        width = 3.5, height = 9.5)
```
