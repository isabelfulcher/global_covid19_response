-   [Global COVID-19 Response](#global-covid-19-response)
    -   [Table of Contents](#table-of-contents)
    -   [About:](#about)
    -   [Goals:](#goals)
    -   [Modelling technique:](#modelling-technique)
        -   [Facility-level models:](#facility-level-models)
        -   [District and county-level
            models:](#district-and-county-level-models)
        -   [Missing data considerations:](#missing-data-considerations)
    -   [Overview of folders and files:](#overview-of-folders-and-files)
        -   [data](#data)
        -   [R](#r)
        -   [figures](#figures)
        -   [Example](#example)

    liberia_shape <- readOGR("../global_covid19_ss/liberia/data/shape/liberia_fixed/Export_Output_2.shp")

    ## OGR data source with driver: ESRI Shapefile 
    ## Source: "/Users/nicholekulikowski/Dropbox/global_covid19_ss/liberia/data/shape/liberia_fixed/Export_Output_2.shp", layer: "Export_Output_2"
    ## with 15 features
    ## It has 4 fields

Global COVID-19 Response
========================

*Last updated: 15 Sep 2020*

Table of Contents
-----------------

-   [About](#About)
-   [Goals](#Goals)
-   [Modelling technique](#Modelling-technique)
    -   [Facility-level models](#Facility-level-models)
    -   [District and county-level
        models](#District-and-county-level-models)
    -   [Missing data considerations](#Missing-data-considerations)
-   [Overview of folders and files](#Overview-of-folders-and-files)

About:
------

This repository contains code to follow the Data Processing Pipeline for
the Global Covid-19 Syndromic Surveillance Team - a partnership between
sites at Partners in Health, the Global Health Research Core at Harvard
Medical School, and Brigham and Women’s Hospital. The data has been
modified to respect the privacy of our sites, in hopes that other groups
can benefit from the functions we have written.

This repository contains data, code, and other items needed to reproduce
this work. Outputs include figures, tables, and Leaflet maps. Further
explanation of outputs and their construction is given in the “Overview
of folders and files” section, which includes detailed explanations of
the functions we have written.

Goals:
------

The main goal of the Global COVID-19 Syndromic Survillance Team is to
monitor changes in indicators that may signal changes in COVID-19 case
numbers in health systems from our eight partnering countries: Haiti,
Lesotho, Liberia, Madagascar, Malawi, Mexico, Peru, and Rwanda. This is
accomplished through establishing a baseline using prior data, and
monitoring for deviations for relevant indicators. The data
visualization tools created using our functions allow identification of
local areas that are experiencing upticks in COVID-19-related symptoms.

Modelling technique:
--------------------

The process starting with the raw data and finishing with the various
outputs is referred to as the Data Processing Pipeline (see Figure 1
below):

After data has been cleaned, it is processed according to the level it
is available at (either on a facility of county/district basis) for each
indicator. This is done by taking data from a historic baseline period,
and then projecting it into the evaluation period. This then is compared
to the observed counts/proportions. A 95% confidence interval has been
chosen, and we have defined the baseline period to be data from January
2016-December 2019.

The functions included in this repository focus on the modelling and
processing stages.

### Facility-level models:

For facility-level assessments, we fit a generalized linear model with
negative binomial distribution and log-link to estimate expected monthly
counts. Only data from the baseline period will be used to estimate the
expected counts:
$$ \\log(E\[Y | year, t \]) = \\beta\_0 + \\beta\_1year + \\sum\_{k=1}^{3} \\beta\_{k1} cos(2 \\pi kt/12) + \\beta\_{k2} sin(2 \\pi kt/12) $$
 where Y indicates monthly indicator count, t indicates the cumulative
month number. The year term captures trend, and the harmonic term
captures seasonality. This model is an adaptation of that proposed by
Dan Weinberger lab (CITE). If data is available on a more granular
level, then weekly or daily terms could be added to the equation to
capture other types of trend. To calculate the prediction intervals, we
used ciTools R package
(<a href="https://cran.r-project.org/web/packages/ciTools/ciTools.pdf" class="uri">https://cran.r-project.org/web/packages/ciTools/ciTools.pdf</a>).

For proportions, in which the numerator is indicator counts and the
denominator is outpatient visits, we produced similar prediction
intervals using the following procedure: we performed a parametric
bootstrap procedure that generates random monthly indicator counts from
the prediction intervals described above and kept the total outpatient
visits fixed. This gives empirical estimates and prediction intervals
for proportions. If there were missing values for the monthly outpatient
visit count, instead of deleting those months and doing a complete-case
analysis which would waste existing indicator count data, we performed
an imputation procedure as follows: first, we fit the aforementioned
model for outpatient visits instead of indicator counts, and using that
model’s estimates, imputed the missing denominator values. Then, we can
do the parametric bootstrap procedure with the additional step of
randomly imputing missing denominator values in order to account for
variation and uncertainty in these imputed outpatient values.

### District and county-level models:

In Liberia, it was also of interest to perform syndromic surveillance at
the district and county-level. If there was no missing data, one could
simply sum the ARI counts across all facilities within a district (or
county) and fit the above model. However, the Liberia data contains
months with missing counts at the facility-level. We used a parametric
bootstrap to impute the missing values from the facility-level models in
the previous section. We drew realizations of the ARI counts for each
month and each facility and then summed these values for a district (or
county) level estimate. We repeated this procedure 500 times and took
the 2.5th and 97.5th percentiles to create 95% prediction intervals. For
region-level proportions, the number of outpatient visits can be summed
across facilities and a proportion can be computed. If there are missing
values in the outpatient visits, another step can be included in the
above parametric bootstrap procedure where missing outpatient visits are
generated from fitting the above model and where Y indicates monthly
outpatient visit count.

Alternatively, one could fit a generalized linear mixed model using the
above equation with a random effect terms for each facility within the
region. The region-level count estimates can then be obtained by
integrating over the random effects distribution. Ultimately, we did not
choose this model due to its lack of flexibility in dealing with missing
data.
$$ \\log(E\[Y\_j | year, t \]) = \\beta\_0 ^\* + \\beta\_1^\*year + \\sum\_{k=1}^{3} \\beta\_{k1}^\* cos(2 \\pi kt/12) + \\beta\_{k2}^\* sin(2 \\pi kt/12) + \\gamma \_{0j} $$

### Missing data considerations:

We excluded facilities from our analysis for two reasons: (1) missing
dates in the baseline period (creation of the expected counts model) (2)
missing observed counts in the evaluation period.

For the first reason, facilities with high levels of missing data (more
than 20% of baseline dates missing) were excluded. Although there are
statistical methods that can handle missing time series data, we decided
to only include sites that demonstrated ability to collect data over
time. A complete case (time) analysis was conducted with included
facilities, which assumes that the counts were missing completely at
random (MCAR). Specifically, we assumed the reason for missing counts
was independent of time and count value. If the MCAR assumption was
violated and we had information about the missing data mechanism, one
could impute values for the missing data and report unbiased expected
counts and correct inference.

For the second reason, facilities with ANY missing monthly data during
the evaluation period (January 2020 onward) were removed. As the
syndromic surveillance exercise hinges on comparing the observed counts
to the expected and flagging for deviations, we require complete
observed data during this period. In this context, it would be invalid
to impute observed counts based on information from the baseline period.
In theory, one could attempt to impute the observed count based on
information during the evaluation period.

Overview of folders and files:
------------------------------

### data

This folder contains example data used to demonstrate functions.
\#\#\#\# data.example\_singlecounty.rds The facility-level dataset used
to demonstrate the functions throughout this repository. Note- specific
names and numbers have been altered to respect the privacy of our sites.

### R

This folder contains the functions used to create the key data
visualization figures and maps.

### figures

This folder contains figures that have been included in README.md.

### Example

#### Loading Data and Functions

    source("R/model_functions.R")
    source("R/model_figures.R")

    data <- readRDS("data/data_example_singlecounty.rds")
    head(data)

    ## # A tibble: 6 x 10
    ##   date       county district facility indicator_count… indicator_count…
    ##   <date>     <chr>  <chr>    <chr>               <dbl>            <dbl>
    ## 1 2016-01-01 Count… Distric… Facilit…               29               15
    ## 2 2016-01-01 Count… Distric… Facilit…               40                7
    ## 3 2016-01-01 Count… Distric… Facilit…               46               13
    ## 4 2016-01-01 Count… Distric… Facilit…              114               32
    ## 5 2016-01-01 Count… Distric… Facilit…               38               17
    ## 6 2016-01-01 Count… Distric… Facilit…               43               17
    ## # … with 4 more variables: indicator_count_ari_over5 <dbl>,
    ## #   indicator_denom <dbl>, indicator_denom_under5 <dbl>,
    ## #   indicator_denom_over5 <dbl>

The data loaded here are taken from a county in Liberia and perturbed
slightly. The indicator of interest is acute respiratory infections,
disaggregated by age, and we also see total outpatient visits
(indicator\_denom)–a measure of healthcare utilization–disaggregated by
age.

#### Example 1: Single Facility

We take an example facility–“Facility K”, run the facility-specific
model, and look at the results through the counts and proportion lenses.

    # Declare this for all functions
    extrapolation_date <- "2020-01-01"

    # Run Facility-level Model
    example_1_results <- fit.site.specific.denom.pi(data=data,
                                  site_name="Facility K",
                                  extrapolation_date="2020-01-01",
                                  indicator_var="indicator_count_ari_total",
                                  denom_var="indicator_denom", 
                                  site_var="facility",
                                  date_var="date",
                                  R=500)

##### Single Facility Counts Results

    plot_heatmap(df = example_1_results,
                 indicator = "indicator_count_ari_total",
                 extrapolation_date = "2020-01-01")

![](README_files/figure-markdown_strict/unnamed-chunk-6-1.png)

    plot_site(input = example_1_results,
              site_name = "Facility K")

![](README_files/figure-markdown_strict/unnamed-chunk-6-2.png)

##### Single Facility Proportions Results

    plot_heatmap_prop(df = example_1_results,
                 indicator = "indicator_count_ari_total",
                 extrapolation_date = "2020-01-01")

![](README_files/figure-markdown_strict/unnamed-chunk-7-1.png)

    plot_site_prop(input = example_1_results,
              site_name = "Facility K")

![](README_files/figure-markdown_strict/unnamed-chunk-7-2.png)

#### Example 2: All Facilities

We repeat the process above for all indicators and all facilites. In
this example dataset, there are 25 facilites, 3 syndromic surveillance
indicators–ARI, ARI under 5, ARI over 5–and 3 denominator
indicators–total denominator, denominator under 5, denominator over 5.
Note these results are needed for the subsequent county-level analyses.

    #get all sites

    all_sites <- data %>% distinct(facility) %>% pull()

    # loop over all syndromic surveillance indicators and facilities

    lapply(c("indicator_count_ari_total","indicator_count_ari_under5","indicator_count_ari_over5"), function(y){

      if ("under5" %in% y){
        do.call(rbind, lapply(all_sites,function(x)
          fit.site.specific.denom.pi(data=data,
                                  site_name=x,
                                  extrapolation_date=extrapolation_date,
                                  indicator_var=y,
                                  denom_var="indicator_denom_under5",
                                  site_var="facility",
                                  date_var="date",
                                  R=500)))
      }
      else if ("over5" %in% y){
        do.call(rbind, lapply(all_sites,function(x)
          fit.site.specific.denom.pi(data=data,
                                  site_name=x,
                                  extrapolation_date=extrapolation_date,
                                  indicator_var=y,
                                  denom_var="indicator_denom_over5",
                                  site_var="facility",
                                  date_var="date",
                                  R=500)))
      }
      else {
        do.call(rbind, lapply(all_sites,function(x)
          fit.site.specific.denom.pi(data=data,
                                  site_name=x,
                                  extrapolation_date=extrapolation_date,
                                  indicator_var=y,
                                  denom_var="indicator_denom",
                                  site_var="facility",
                                  date_var="date",
                                  R=500)))
      }
    }) -> facility.list


    # loop over all denominator(outpatient) indicators and facilities

    lapply(c("indicator_denom","indicator_denom_under5","indicator_denom_over5"), function(y){

        do.call(rbind, lapply(all_sites,function(x)
          fit.site.specific.denom.pi(data=data,
                                     site_name=x,
                                     extrapolation_date=extrapolation_date,
                                     indicator_var=y,
                                     site_var="facility",
                                     date_var="date",
                                     counts_only=TRUE)))

    }) -> facility.list.denom



    names(facility.list) <- c("indicator_count_ari_total","indicator_count_ari_under5","indicator_count_ari_over5")
    names(facility.list.denom) <- c("indicator_denom","indicator_denom_under5","indicator_denom_over5")

    plot_facet(input = facility.list[["indicator_count_ari_total"]] %>% dplyr::rename(facet_var=`site`),
               counts_var = "est_raw_counts")

    ## Warning in max(ids, na.rm = TRUE): no non-missing arguments to max; returning -
    ## Inf

![](README_files/figure-markdown_strict/unnamed-chunk-9-1.png)

#### Example 3: County-level

Now we run the county-level model for the ARI indicator. The same can of
course be done for the other indicators of interest.

    county_results <- fit.cluster.pi(data = data,
                               indicator_var = "indicator_count_ari_total",
                               denom_var = "indicator_denom",
                               date_var = "date",
                               denom_results_all = facility.list.denom, 
                               indicator_results_all= facility.list, 
                               counts_only=FALSE,
                               n_count_base = 0,
                               p_miss_base = 0.2,
                               p_miss_eval = 0.5,
                               R=250)

    county_test <- readRDS("../global_covid19_ss/liberia/results/liberia_county_ari_total_final_08_25_2020.rds")

    plot_heatmap_county(df = county_results %>% mutate(site = "County Alpha"),
                 indicator = "indicator_count_ari_total",
                 extrapolation_date = "2020-01-01")

![](README_files/figure-markdown_strict/unnamed-chunk-11-1.png)

    plot_site_county(input = county_results %>% mutate(site = "County Alpha"),
              site_name = "County Alpha")

![](README_files/figure-markdown_strict/unnamed-chunk-11-2.png)

    # for figures, for first part, use output to make 2 figures, tile plot and standard thing
    # nicole using real data maps
    # save output for single facility and multiple facilities to create figures later..another folder called results or output like results_example1 or something
    # example 1 facility counts, example 2 facility proportions, example 3 multiple fac, facet wrap for all of em, example 4, county counts, tile thing
    # separate r script for figures (heat map, time series single, time series facet wrap)
    #
    #
