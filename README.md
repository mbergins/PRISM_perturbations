# Kinase Inhibitor Cell Viability Modeling

This repository contains the source code used to organize the data and build models associated with:

Berginski ME, Joisa C, et al. In process

## Overall Code Organization, Philosophy and Prerequisites

Nearly all of the code in this respository is written in R, almost all using tidyverse-esque idioms. All of the computational modeling code uses the tidymodel framework to faciliate model testing and usage. If you want to get a full handle on this codebase and these concepts are a bit unknown, you would be well served to work through [R for Data Science](https://r4ds.had.co.nz/) and the [tidymodels tutorial](https://www.tidymodels.org/start/models/). Hopefully my coding style is clear enough to make it possible to follow along without these resources, but they are the first places I would send someone who was interested in digging into the code. As for the actual code, I generally work in Rmarkdown documents except in cases where I'm going to need supercomputing resources.

Otherwise, everything in this respository was written using Rstudio, but I don't think Rstudio is strickly necessary to run or modify the code. Typically, when I'm working on this project the first thing I do is open the R project file present in the top level directory. 

The raw data used for this work is too large to fit into this repository, but I've made a copy of the data organized into the directory structure the rest of the code expects on zenodo. A majority of the data sets were downloaded from the [depmap data portal](https://depmap.org/portal/download/) and compressed, with the remainder coming from journal supplemental data sections. These files should end up in a top level "/data" directory.

As for the overall organization of the code, there are three primary divisions:

* Code for organizing the raw data: located in [`src/data_organization`](src/data_organization)
* Code for testing models: this is a majority of the rest of the code, so in the modeling section I'll point towards specific locations to look for code related to the models in the paper
* Code for organizing the validation data: located in [`src/validation_screen`](src/validation_screen)

Before diving into the rest of the code, you should take a look at and run [`package_check.R`](src/package_check.R). This is a script that uses the pacman library to check for library installations and if missing installs them. It also installs two packages from github that I maintain ([DarkKinaseTools](https://github.com/IDG-Kinase/DarkKinaseTools) and [BerginskiRMisc](https://github.com/mbergins/BerginskiRMisc)). 

I've only ever tested this code on Linux (Ubuntu) and the supercomputing cluster at UNC. I think a majority of the code will work on other platforms, but I haven't tested it.

# Data Organization Code

This section of the code base deals with organizing, reformatting and otherwise preparing the raw downloaded data sets for the modeling sections of the code. In general, I tried to divide the code into sections that directly touch the raw data (this code) and code that only deals with the output of this section (the rest of the code). As mentioned above, all the code is in [`src/data_organization`](src/data_organaization) and the following Rmarkdown scripts contain the critical code:

* [`Klaeger Data`](src/data_organization/process_klaeger_data/klaeger_data_processing.Rmd): This code takes supplemental table 2 from Klaeger 2017 and converts it into a machine learning ready RDS file. There are quite a few decisions that went into getting the data into shape and this is all documented in the text surrounding the code.
* [`PRISM Viability`](src/data_organization/prep_PRISM_for_ML/prep_PRISM_for_ML.Rmd): This code takes the responce curve values provided by the PRISM consortium and imputes cell viability values at all of the concentrations used by Klaeger et al.
* [`DepMap CRISPR-KO`](src/data_organization/prep_depmap_data_for_ML/prep_depmap_for_ML.Rmd): The data from the DepMap consortium is very well organized, so not much was really necesary here. I simplified the names provided for each gene and filtered out a few lines with unexpected NAs.
* [`CCLE Data Sets`](src/data_organization/prep_CCLE_data_for_ML/prep_CCLE_for_ML.Rmd): The data from the CCLE is similarly well organized, so all that was really needed was some gene name simplification.
* [`CCLE Proteomics`](src/data_organization/prep_CCLE_proteomics_data/prep_CCLE_proteomics_data.Rmd): The code takes table S2 from the CCLE proteomics project, available from the Gygi lab [website](https://gygi.hms.harvard.edu/publications/ccle.html), and converts it into a machine learning ready RDS file. This part of the processing pipeline is a bit more complex as we need to deal with a larger amount of missing readings here and the associated imputation.

All the other script files in the [`data organization`](src/data_organaization) are related to other ideas we've had that weren't important in the paper. I thought about scrubbing all the unnecessary files out, but maybe someone will find something interesting in them.

## Reproducibility

I've made a simple script that runs the above scripts in sequence to produce all the files needed for the modelling effort ([`reproduce_data_org.R`](src/data_organization/reproduce_data_org.R)). On my computer (Ryzen 7 5800x) it took about 3 minutes and used 4.7 GB of RAM.

# Modeling Code

As I mentioned above, all the modeling code is written using the tidymodels framework and each modeling attempt has two parts. The first part of the modeling code calculates the correlation coefficients between cell viability and each of the potential model features using the cross validation folds if to divide the data if requested. The second part of the modeling method then uses those correlation values and a given number of features to then build and test the model. Since the primary model we selected in the paper was a random forest with 500 features selected from the kinase activation states and gene expression, I'll use those files as examples.

## Sample Modeling Code Discussion

* [`get_feat_cor.R`](src/build_ML_models_expression_regression/get_feat_cor.R): This script reads in the data set from the data organization section and finds the correlations between cell viability and each of the features. In the case of this file, it only reads and searches the Klaeger activation states and gene expression, but this varies depending on the data sets the model includes. The feature correlation calculations are calculated in a cross validation aware fashion. This script also contains the code for making the cross validation fold assignments and building the full data file needed to make the final model that includes all of the data. Normally, with new model testing, I run this code once to make the CV fold assignments (saved to disk) to ensure that I don't accidentally use multiple cross validation assignments. You might be wondering why I don't just the built in CV code in tidymodels and it's because tidymodels doesn't support out of the differing feature selection across folds and this also makes it possible to run all the CV folds independently on the supercomputer.
* [`build_random_forest_models_no_tune.R`](src/build_ML_models_expression_regression/build_random_forest_models_no_tune.R): Hopefully you will forgive my overly wordy file names, but as you can hopefully guess, this script builds a random forest model for CV testing without tuning. It takes two parameters, the cross validation fold ID to run on and the number of features to include in the model, although the code will also run without specifying any parameter with a default of 100 features and CV fold ID 1. The other model types ([`XGBoost`](src/build_ML_models_expression_regression/build_xgboost_models_no_tune.R) and [`linear regression`](src/build_ML_models_expression_regression/build_linear_models.R)) get their own files that look very similar to this one (thanks tidymodels).

There are two other smaller scripts used to make the modeling run [`send_feat_cor_cmd.R`](src/build_ML_models_expression_regression/send_feat_cor_cmd.R) and [`send_model_cmd.R`](src/build_ML_models_expression_regression/send_model_cmd.R). These scripts simply build the commands needed to run the feature correlations and modeling runs and send them into the computational queue on the supercomputer at UNC. If you have access to a slurm cluster these will probably work, but if not you can use them as a template to run your models locally or on whatever supercomputing system you use.



## Reproducibility

# Validation Data Code

## Reproducibility
