Introduction
------------

OpenSAFELY is a new secure analytics platform for electronic health
records in the NHS, created to deliver urgent research during the global
COVID-19 emergency. It is now successfully delivering analyses across
more than 24 million patients’ full pseudonymised primary care NHS
records, with more to follow shortly.

All our analytic software is open for security review, scientific
review, and re-use. OpenSAFELY uses a new model for enhanced security
and timely access to data: we don’t transport large volumes of
potentially disclosive pseudonymised patient data outside of the secure
environments managed by the electronic health record software company;
instead, trusted analysts can run large scale computation across near
real-time pseudonymised patient records inside the data centre of the
electronic health records software company.

This document is intended as a short walkthrough of the OpenSAFELY
platform. Please visit
[docs.opensafely.org](https://docs.opensafely.org/en/latest/) for more
comprehensive documentation, and
[opensafely.org/](https://opensafely.org/) for any other info.

The examples in this document are all available in the
[os-demo-research](https://github.com/opensafely/os-demo-research)
github repository.

### Technical pre-requisites

OpenSAFELY maintains extremely high standards for data privacy, whilst
ensuring complete computational and analytical transparency. As such
there are some technical pre-requisites that users must satisfy to use
the platform. These include installing and using git and python, though
typically only a narrow set of actions are required and guide-rails are
provided. These details can be read in the documentation pages, starting
[here](https://docs.opensafely.org/en/latest/min_req/). The walkthrough
assumes that the technical set-up has been completed.

### Key concepts

-   A **study definition** specifies the patients you want to include in
    your study and defines the variables that describe them. Study
    definitions are defined as a Python script. that relies heavily on
    an easily-readable functions API.
-   The **cohort extractor** uses the study definition to create a
    dataset for analysis. This is either:
    -   A dummy dataset used for developing and testing analysis code on
        the user’s own machine. Users have control over the
        characteristics of each dummy variable, which are defined inside
        the study definition.
    -   A real dataset created from the OpenSAFELY database, used for
        the analysis proper. Real datasets never leave the secure
        server, only summary data and other outputs that are derived
        from them can be released (after disclosivity checks). It also
        performs other useful tasks like importing codelists and
        generating *Measures* (see below).
-   A **Codelist** is a collection of clinical codes that define a
    particular condition, event or diagnosis.
-   The **project pipeline** defines dependencies within the project’s
    analytic code. For example `make_chart.R` depends on
    `process_data.R`, which depends on the study dataset having been
    extracted. This reduces redundancies by only running scripts that
    need to be run.
-   The **job runner** runs the actions defined in the project pipeline
    using real data.

### Workflow

For researchers using the platform, the OpenSAFELY workflow for a single
study is typically as follows:

1.  **Create a git repository** from the template provided and clone it
    on your local machine
2.  **Create a Study Definition** that specifies what data you want to
    extract from the database:
    -   Specify the patient population (dataset rows) and variables
        (dataset columns)
    -   Specify the expected distributions of these variables for use in
        dummy data
    -   Specify the codelists required by the study definition, as
        hosted by codelists.opensafely.org, and import them to the repo
3.  **Generate the dummy data**
4.  **Develop analysis scripts** using the dummy dataset in R, Stata, or
    Python. This will include:
    -   importing and processing the dataset(s) created by the cohort
        extractor
    -   importing any other external files needed for analysis
    -   generating analysis outputs like tables and figures
    -   generating log files to debug the scripts when run on the real
        data
5.  **Create a project pipeline** to run steps 3 and 4
6.  **Run the analysis scripts on the real data** via the job runner
7.  **Check the output for disclosivity** within the server
8.  **Release the outputs** via github

Steps 2-5 can all be progressed on your local machine without accessing
the real data. These steps are iterative and should proceed with
frequent git commits and code reviews where appropriate. These steps are
automatically tested against dummy data every time a new version of the
repository is saved (“pushed”) to GitHub.

Let’s start with a simple example.

Example 1 — STP patient population
----------------------------------

This example introduces study definitions, expectations and dummy data,
and project pipelines. We’re going to use OpenSAFELY to find out how
many patients are registered at a TPP practice within each STP
(Sustainability and Transformation Partnership) on 1 January 2020.

### Create a git repository

Go to the [research template
repository](https://github.com/opensafely/research-template) and click
`Use this template`. Follow this instructions
[here](https://docs.opensafely.org/en/latest/project_setup/).

### Create a Study Definition

Your newly-created repo will contain a template the study definition.
Edit this to suit your needs. In this example, it’s a very simple — the
entire file, called
[`study_definition_1_stppop.py`](https://github.com/opensafely/os-demo-research/blob/master/analysis/study_definition_1_stppop.py),
looks like this:

``` python
## LIBRARIES

# cohort extractor
from cohortextractor import (StudyDefinition, patients)

# dictionary of STP codes (for dummy data)
from dictionaries import dict_stp

# set the index date
index_date = "2020-01-01"

## STUDY POPULATION

study = StudyDefinition(

    default_expectations = {
        "date": {"earliest": index_date, "latest": "today"}, # date range for simulated dates
        "rate": "uniform",
        "incidence": 1
    },
    
    # This line defines the study population
    population = patients.registered_as_of(index_date),

    # this line defines the stp variable we want to extract
    stp = patients.registered_practice_as_of(
        index_date,
        returning="stp_code",
        return_expectations={
            "category": {"ratios": dict_stp},
        },
    ),
)
```

Let’s break it down:

``` python
from cohortextractor import (StudyDefinition, patients)
```

This imports the required functions from the OpenSAFELY
`cohortextractor` library, that you will have previously installed.

``` python
from dictionaries import dict_stp
```

This imports a dictionary of STP codes for creating the STP dummy data.

``` python
index_date = "2020-01-01"
```

This defines the registration date that we’re interested in.

We then use the `StudyDefinition()` function to define the cohort
population and the variables we want to extract.

``` python
default_expectations = {
    "date": {"earliest": index_date, "latest": "today"}, # date range for simulated dates
    "rate": "uniform",
    "incidence": 1
},
```

This defines the default expectations which are used to generate dummy
data. This just says that we expect date variables to be uniformly
distributed between the index date and today’s date. For this study
we’re not producing any dates.

``` python
population = patients.registered_as_of(index_date)
```

This says that we want to extract information only for patients who were
registered at a practice on the 1 January 2020. There will be one row
for each of these patients in the extracted dataset. Note that
`population` is a reserved variable name for `StudyDefinition` which
specifies the study population — we don’t have to do any additional
filtering/subsetting on this variable.

``` python
stp = patients.registered_practice_as_of(
  index_date,
  returning="stp_code",
  return_expectations={
    "incidence": 0.99,
    "category": {"ratios": dict_stp},
  },
)
```

This says we want to extract the STP for each patient (or more strictly,
the STP of each patients’ practice). Here we also use the
`returning_expectations` argument, which specifies how the `stp`
variable will be distributed in the dummy data. option, The
`"incidence": 0.99"` line says that we expect an STP code to be
available for 99% of patients. The `"category": {"ratios": dict_stp}`
line uses the pre-defined STP dictionary `dict_stp` (imported earlier)
to define the expected distribution of STPs. This is currently set to a
uniform distribution, so that each STP is equally likely to appear in
the dataset.

This study definition uses two in-built variable extractor functions in
OpenSAFELY’s `patients` library, `patients.registered_as_of()` and
`patients._practice_as_of()`. There are many more such functions, like
`patients.age()`, `patients.with_these_clinical_events()`, and
`patients.admitted_to_icu()`, which are all documented here.

### Generate dummy data

Now that we’ve defined the study cohort, we can generate a dummy
dataset. Assuming you have the correct technical set-up on your local
machine, this is simply a case of submitting the following command in a
Terminal that can find `cohortextractor` (you can use
`cohortextractor --help` to find out more about this command, the
available options, defaults, etc.):

    cohortextractor generate_cohort --study-definition study_defintion_1_stppop.py --expectations-population 10000 --output-dir=output/cohorts

This will create a file `input_1_stppop.csv` in the `/output/cohorts/`
folder with `10000` rows.

### Develop analysis scripts

We now have a dummy dataset so we can begin to develop and test our
analysis code. For this example, we just want to import the dataset,
count the number of STPs, and output to a file. These steps are outlined
code-block below, and are available in the file
[`/analysis/1-plot-stppop.R`](https://github.com/opensafely/os-demo-research/blob/master/analysis/1-plot-stppop.R).

First we import and process the data (click “code” to the right to show
the code):

``` r
## import libraries
library('tidyverse')
library('sf')
## import data
df_input <- read_csv(
  here::here("output", "cohorts", "input_1_stppop.csv"), 
  col_types = cols(
    patient_id = col_integer(),
    stp = col_character()
  )
)
# import the STP shapefile for the map
# from https://openprescribing.net/api/1.0/org_location/?format=json&org_type=stp
# not importing directly from the URL because no access on the server, so have copied into the repo
sf_stp <- st_read(here::here("lib", "STPshapefile.json"))

# count STP for each registered patient
df_stppop = df_input %>% count(stp, name='registered')

#combine with STP geographic data
sf_stppop <- sf_stp %>% 
  left_join(df_stppop, by = c("ons_code" = "stp")) %>%
  mutate(registered = if_else(!is.na(registered), registered, 0L))
```

Then we create a map:

``` r
plot_stppop_map <- sf_stppop %>%
ggplot() +
  geom_sf(aes(fill=registered), colour='black') +
  scale_fill_gradient(limits = c(0,NA), low="white", high="blue")+
  labs(
    title="TPP-registered patients per STP",
    subtitle= "as at 1 January 2020",
    fill = NULL)+
  theme_void()+
  theme(
    legend.position=c(0.1, 0.5)
  )
plot_stppop_map
```

![](Rdemo_files/figure-markdown_github/stppop.map-1.png)

Or a bar chart if you prefer (click “code” to the right to show the
code):

``` r
plot_stppop_bar <- sf_stppop %>%
  mutate(
    name = forcats::fct_reorder(name, registered, median, .desc=FALSE)
  ) %>%
  ggplot() +
  geom_col(aes(x=registered/1000000, y=name, fill=registered), colour='black') +
  scale_fill_gradient(limits = c(0,NA), low="white", high="blue", guide=FALSE)+
  labs(
    title="TPP-registered patients per STP",
    subtitle= "as at 1 January 2020",
    y=NULL,
    x="Registered patients\n(million)",
    fill = NULL)+
  theme_minimal()+
  theme(
    panel.grid.major.y = element_blank(),
    panel.grid.minor.y = element_blank(),
    plot.title.position = "plot",
    plot.caption.position =  "plot"
  )
plot_stppop_bar
```

![](Rdemo_files/figure-markdown_github/stppop.bar-1.png)

This is just dummy data. But we can easily rerun the code above on the
real dataset. For this, we use the OpenSAFELY Job Runner.

### Create a project pipeline

OpenSAFELY uses a Job Runner to run analysis scripts on the real data.
This includes extracting data from the OpenSAFELY database as defined in
a study definition, and running R or Stata scripts on that data. To do
this, we define the Project Pipeline, which is a set of actions written
in a file called `project.yaml`, which lives in the root directory of
the repo. This is best demonstrated with an example:

``` yaml

version: '3.0'

expectations:
  population_size: 10000

actions:

  generate_cohort_stppop:
    run: cohortextractor:latest generate_cohort --study-definition study_definition_1_stppop --output-dir=output/cohorts
    outputs:
      highly_sensitive:
        cohort: output/cohorts/input_1_stppop.csv

  plot_stppop:
    run: r:latest analysis/1-plot-stppop.R
    needs: [generate_cohort_stppop]
    outputs:
      moderately_sensitive:
        log: output/logs/log-1-plot-stppop.txt
        figure1: output/plots/plot_stppop_map.png
        figure2: output/plots/plot_stppop_bar.png
        
        
  run_all:
    needs: [plot_stppop]
    run: cohortextractor:latest --version
    outputs:
      moderately_sensitive:
        whatever: project.yaml
```

The file begins with some housekeeping that specifies which version to
use and the size of the dummy dataset used for testing.

The main part of the yaml file is the `actions` section. In this example
we have three actions:

-   `generate_cohort_stppop` is the action to extract the dataset from
    the database. The `run` section gives the extraction command
    (`generate_cohort`), which study definition to use
    (`--study-definition study_definition_1_stppop`) and where to put
    the output (`--output-dir = output/cohorts`). The `outputs` section
    declares that the expected output is `highly_sensitive` and
    shouldn’t be considered for release.
-   `plot_stppop` is the action that runs the R script. The `run`
    section tells it which script to run. The `needs` section says that
    this action cannot be run without having first run the
    `geenrate_cohort_stppop` action. The `outputs` sections declares
    that the expected outputs (a log file for debugging, and the two
    plots) are `moderately_sensitive` and should be considered for
    release after disclosivity checks.
-   `run_all` is an action necessary for testing. By setting the `needs`
    section to include all actions, it will run all other actions and
    test that they can complete successfully.

### Run the analysis scripts on the real data

Now that the `project.yaml` pipeline is defined, it can be run via the
Job Runner at `http://jobs.opensafely.org/jobs/` as follows:

-   create a workspace for the repo + branch, and select the appropriate
    backend (in this case `TPP`)
-   click `add job` and select which actions to run, then submit.

This will create the required outputs on the server, which can be
reviewed on the server and released via github if they are deemed
non-disclosive (there are detailed reviewing guidelines for approved
researchers).

### Check the outputs for disclosivity

This is a manual step that must be carried out entirely in the server.
Instructions for this process are available for those with the
appropriate permissions. Essentially, the step ensures that the outputs
do not contain any potentially disclosive information due to small
patient counts.

### Release the outputs

Once results have been checked they can be released via GitHub. This
step must also occur in the server. The released outputs are put in the
[`/released-output`](https://github.com/opensafely/os-demo-research/tree/master/released-ouput)
folder.

In this example, the map looks like this:

[<img src="../released-ouput/plots/plot_stppop_map.png" title="registration count by STP" style="width:50.0%" />](https://github.com/opensafely/os-demo-research/blob/master/released-ouput/plots/plot_stppop_map.png)

and the bar chart looks like this:

[<img src="../released-ouput/plots/plot_stppop_bar.png" title="registration count by STP" style="width:70.0%" />](https://github.com/opensafely/os-demo-research/blob/master/released-ouput/plots/plot_stppop_bar.png)

Here we’re just linking to the released `.png` files in the repo
(clicking the image takes you there). We could have also released the
underlying STP count data and developed an analysis script directly on
this dataset, provided it is non-disclosive. However, typically only the
most high-level aggregated datasets are suitable for public release.

Example 2 — Covid versus non-covid deaths
-----------------------------------------

This example introduces codelists. We’re going to use OpenSAFELY to look
at the frequency of covid-related deaths compared with non-covid deaths
between 1 January 2020 and 30 September 2020, and see how this differs
by age and sex. For this, we’ll need death dates for anyone in the
database who died during the period of interest, and we’ll need a way to
identify whether these deaths were covid-related or not.

### Study definition

The study definition for this task is available at
[`/analysis/study_definition_2_deaths.py`](https://github.com/opensafely/os-demo-research/blob/master/analysis/study_definition_2_deaths.py),
and can be viewed by clicking `code` to the right.

``` python
## LIBRARIES

# cohort extractor
from cohortextractor import (
    StudyDefinition,
    Measure,
    patients,
    codelist_from_csv,
    codelist,
    filter_codes_by_category,
    combine_codelists
)

## CODELISTS
# All codelist are held within the codelist/ folder.
codes_ICD10_covid = codelist_from_csv(
    "codelists/opensafely-covid-identification.csv", 
    system = "icd10", 
    column = "icd10_code"
)

## STUDY POPULATION
# Defines both the study population and points to the important covariates

index_date = "2020-01-01"
end_date = "2020-09-30"


study = StudyDefinition(
        # Configure the expectations framework
    default_expectations={
        "date": {"earliest": index_date, "latest": end_date},
        "rate": "uniform",
    },

    index_date = index_date,

    # This line defines the study population
    population = patients.satisfying(
        """
        (sex = 'F' OR sex = 'M') AND
        (age >= 18 AND age < 120) AND
        (NOT died) AND
        (registered)
        """,
        
        registered = patients.registered_as_of(index_date),
        died = patients.died_from_any_cause(
            on_or_before=index_date,
            returning="binary_flag",
        ),
    ),

    age = patients.age_as_of(
        index_date,
        return_expectations={
            "int": {"distribution": "population_ages"},
            "incidence": 1
        },
    ),

    sex = patients.sex(
        return_expectations={
            "category": {"ratios": {"M": 0.49, "F": 0.51}},
            "incidence": 1
        }
    ),
    
    date_death = patients.died_from_any_cause(
        between = [index_date, end_date],
        returning = "date_of_death",
        date_format = "YYYY-MM-DD",
        return_expectations = {
            "incidence": 0.2,
        },
    ),

    death_category = patients.categorised_as(
        {
            "covid-death": "died_covid",
            "non-covid-death": "(NOT died_covid) AND died_any",
            "" : "DEFAULT"
        },

        died_covid = patients.with_these_codes_on_death_certificate(
            codes_ICD10_covid,
            returning = "binary_flag",
            match_only_underlying_cause = False,
            between = [index_date, end_date],
        ),

        died_any = patients.died_from_any_cause(
            between = [index_date, end_date],
            returning = "binary_flag",
        ),

        return_expectations = {
            "category": {"ratios": {"": 0.8, "covid-death": 0.1, "non-covid-death": 0.1}}, 
            "incidence": 1
        },
    ),

    

)
```

As before, we first import libraries and dictionaries. This time we also
import the codelist for identifying covid-related deaths. This uses data
from death certificates, which are coded using ICD-10 codes. The covid
codes in this system are `U071` and `U072`, and have been collected in a
codelist at
[codelists.opensafely.org](https://codelists.opensafely.org/codelist/opensafely/covid-identification/2020-06-03).
To import the codelists, put the codelist url in the
`codelists/codelists.txt` file in the repo, then run the following
command:

    cohortextractor update_codelists

This will create a `.csv` for each codelist, which are imported to the
study definition using

``` python
codes_ICD10_covid = codelist_from_csv(
    "codelists/opensafely-covid-identification.csv", 
    system = "icd10", 
    column = "icd10_code"
)
```

Then as before, we define the cohort population and the variables we
want to extract within `StudyDefinition()`. The variables we extract
this time are `age`, `sex`, `date_death`, and `death_category`. These
use some new extractor functions and some more expectation definitions,
whose details can be found in the documentation
[here](https://docs.opensafely.org/en/latest/study_def_intro/).

### Results

Once again, we use the dummy data to develop the analysis script then
create a `project.yaml` to run the entire study on the real data. The
analysis script is available in the file
[`/analysis/2-plot-deaths.R`](https://github.com/opensafely/os-demo-research/blob/master/analysis/2-plot-deaths.R).
This script is run by the Job Runner via the `project.yaml` file, then
the outputs are reviewed in the server and released via github. The
final graph looks like this:

[<img src="../released-ouput/plots/plot_deaths.png" title="covid-related deaths" style="width:80.0%" />](https://github.com/opensafely/os-demo-research/blob/master/released-ouput/plots/plot_deaths.png)

Here you can see the familiar covid mortality bump during the first wave
of the pandemic. There is also a bump in non-covid deaths, suggesting
that identification of covid-related deaths may not be 100% sensitive.

Example 3 — Primary care activity throughout the pandemic.
----------------------------------------------------------

In our final example, we introduce the Measures framework. This enables
the extraction of multiple study cohorts each covering different time
periods, and calculates a set of statistics for each period. We’ll look
at the frequency of cholesterol and INR (International Normalised Ratio,
which measures how long it takes for blood to clot) measurements
recorded in the Primary Care record, by practice and by STP.

### Study Definition

The entire study definition is available at
[`/analysis/study_definition_3_activity.py`](https://github.com/opensafely/os-demo-research/blob/master/analysis/study_definition_3_activity.py),
and can be viewed by clicking `code` to the right.

``` python
## LIBRARIES

# cohort extractor
from cohortextractor import (
    StudyDefinition,
    Measure,
    patients,
    codelist_from_csv,
    codelist,
    filter_codes_by_category,
    combine_codelists
)

## CODELISTS
# All codelist are held within the codelist/ folder.
codes_cholesterol = codelist_from_csv(
    "codelists-local/cholesterol-measurement.csv", 
    system = "ctv3", 
    column = "id"
)

codes_inr = codelist_from_csv(
    "codelists-local/international-normalised-ratio-measurement.csv", 
    system = "ctv3", 
    column = "id"
)


# dictionary of STP codes (for dummy data)
from dictionaries import dict_stp


## STUDY POPULATION

index_date = "2020-01-01"

study = StudyDefinition(
        # Configure the expectations framework
    default_expectations={
        "date": {"earliest": index_date, "latest": "today"},
        "rate": "uniform",
        "incidence": 1
    },

    index_date = index_date,

    # This line defines the study population
    population = patients.satisfying(
        """
        (age >= 18 AND age < 120) AND
        (NOT died) AND
        (registered)
        """,
        
        died = patients.died_from_any_cause(
            on_or_before=index_date,
            returning="binary_flag"
        ),
        registered = patients.registered_as_of(index_date),
        age=patients.age_as_of(index_date),
    ),

    ### geographic/administrative groups
    practice = patients.registered_practice_as_of(
         index_date,
         returning = "pseudo_id",
         return_expectations={
             "int": {"distribution": "normal", "mean": 100, "stddev": 20}
         },
    ),

    stp = patients.registered_practice_as_of(
        index_date,
        returning="stp_code",
        return_expectations={
            "category": {"ratios": dict_stp},
        },
    ),

    cholesterol = patients.with_these_clinical_events(
        codes_cholesterol,
        returning = "number_of_episodes",
        between = ["index_date", "index_date + 1 month"],
        return_expectations={
            "int": {"distribution": "normal", "mean": 2, "stddev": 0.5}
        },
    ),

    inr = patients.with_these_clinical_events(
        codes_inr,
        returning = "number_of_episodes",
        between = ["index_date", "index_date + 1 month"],
        return_expectations={
            "int": {"distribution": "normal", "mean": 3, "stddev": 0.5}
        },
    ),
)


measures = [
    Measure(
        id="cholesterol_practice",
        numerator="cholesterol",
        denominator="population",
        group_by="practice"
    ),
    Measure(
        id="cholesterol_stp",
        numerator="cholesterol",
        denominator="population",
        group_by="stp"
    ),
    Measure(
        id="inr_practice",
        numerator="inr",
        denominator="population",
        group_by="practice"
    ),
    Measure(
        id="inr_stp",
        numerator="inr",
        denominator="population",
        group_by="stp"
    ),
]
```

We’ll pick out some key parts:

``` python
population = patients.satisfying(
    """
    (age >= 18 AND age < 120) AND
    (NOT died) AND
    (registered)
    """,
    died = patients.died_from_any_cause(
      on_or_before = index_date,
      returning = "binary_flag"
    ),
    registered = patients.registered_as_of(index_date),
    age = patients.age_as_of(index_date),
)
```

The population declaration says that for each period, we want the set of
adult patients who are both alive and registered at the index date.

``` python
cholesterol = patients.with_these_clinical_events(
    codes_cholesterol,
    returning = "number_of_episodes",
    between = ["index_date", "index_date + 1 month"],
    return_expectations={
        "int": {"distribution": "normal", "mean": 2, "stddev": 0.5}
    },
),

inr = patients.with_these_clinical_events(
    codes_inr,
    returning = "number_of_episodes",
    between = ["index_date", "index_date + 1 month"],
    return_expectations={
        "int": {"distribution": "normal", "mean": 3, "stddev": 0.5}
    },
),
```

Then we want to extract the number of cholesterol- or INR-measurement
“episodes” recorded during the month beginning the index date. The
`codes_cholesterol` and the `codes_inr` codelists are defined similarly
to the `codes_ICD10_covid` in example 2. Using expectations, we say the
dummy values for these variables will be a low-valued integer.

``` python
measures = [
    Measure(
        id="cholesterol_practice",
        numerator="cholesterol",
        denominator="population",
        group_by="practice"
    ),
    Measure(
        id="cholesterol_stp",
        numerator="cholesterol",
        denominator="population",
        group_by="stp"
    ),
    Measure(
        id="inr_practice",
        numerator="inr",
        denominator="population",
        group_by="practice"
    ),
    Measure(
        id="inr_stp",
        numerator="inr",
        denominator="population",
        group_by="stp"
    ),
]
```

Finally, we define the measures that we want to calculate. Here we want
four measures, one for each combination of cholesterol/inr and
practice/STP. For more details on Measures, see the documentation
[here](https://docs.opensafely.org/en/latest/measures/).

### Generating measures

As before, we generate dummy data using the `generate_cohort` command in
the `cohortextractor`, but this time, we include an `--index-date-range`
option so that it extracts a new cohort for each date specified, as
follows:

    cohortextractor generate_cohort --study-definition study_definition_3_activity --expectations-population 10000 --index-date-range "2020-01-01 to 2020-09-01 by month"

Here we go from 1 January 2020 to 1 September 2020 in monthly
increments. These dates are passed to the `index_date` variable in the
study definition. This will produce a set of
`input_3_activity_<date>.csv` files, each with `10000` rows of dummy
data.

An additional step is now needed to generate the measures. This is done
as follows:

    cohortextractor generate_measures --study-definition study_definition_3_activity

which will produce a set of files of the form `measure_<id>.csv`.

### Results

Now we have the extracted outputs we can develop our analysis script.
The analysis script for this can be found at
[`/analysis/3-plot-activity.R`](https://github.com/opensafely/os-demo-research/blob/master/analysis/3-plot-activity.R).

Let’s look at the number of measurements in each STP for cholesterol:

[<img src="../released-ouput/plots/plot_each_cholesterol_stp.png" style="width:80.0%" />](https://github.com/opensafely/os-demo-research/blob/master/released-ouput/plots/plot_each_cholesterol_stp.png)

and for INR:

[<img src="../released-ouput/plots/plot_each_inr_stp.png" style="width:80.0%" />](https://github.com/opensafely/os-demo-research/blob/master/released-ouput/plots/plot_each_inr_stp.png)

Clearly there is a substantial dip in activity for cholesterol which
corresponds neatly to the first pandemic wave. This activity dip is
similar across all STPs. However, for INR the decline is less
pronounced.

We can also look at deciles of measurement activity for each GP practice
for cholesterol:

[<img src="../released-ouput/plots/plot_quantiles_cholesterol_stp.png" style="width:80.0%" />](https://github.com/opensafely/os-demo-research/blob/master/released-ouput/plots/plot_quantiles_cholesterol_practice.png)

and for INR:

[<img src="../released-ouput/plots/plot_quantiles_inr_stp.png" style="width:80.0%" />](https://github.com/opensafely/os-demo-research/blob/master/released-ouput/plots/plot_quantiles_inr_practice.png)

Future developments on the OpenSAFELY roadmap
---------------------------------------------

-   Better dummy data, respecting between-variable dependencies and
    providing pre-loaded dictionaries
-   Automated comparisons between dummy data and real data
-   Automated disclosivity checks
-   Fast patient matching algorithms for example for case-control
    studies
-   More comprehensive population coverage
-   More external datasets
-   Live dashborads of health service activity
-   etc, etc
