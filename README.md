# mntoha-data-release
Thermal Optical Habitat Area data release

## Organization

Though we don't strictly adhere to these divisions, here's the general idea for the output folders:

* `out` contains git-committable yml or ind files that describe the existence of data files
* `out_data` contains data files that are directly posted to SB as-is
* `tmp` contains data files that we should be able to delete to save space without disrupting much pipeline progress. These may also be posted to SB as-is

## PGDL outputs

PGDL outputs are built and posted directly from Tallgrass.

```sh
ssh tallgrass.cr.usgs.gov
cd /caldera/projects/usgs/water/iidd/datasci/lake-temp/mntoha-data-release
source activate mntoha_release
git pull
```

These targets should be built in the `lake-temperature-neural-networks` project:
```
2_model/log/pgdl_outputs.ind # or otherwise be sure the batch is complete
3_assess/out/posthoc_config.csv
3_assess/log/preds_pretrain.ind
3_assess/log/preds_holdout.ind
3_assess/log/preds_final.ind
```
and these targets should be built in `lake-temperature-model-prep` (I recommend a local build/Drive pull and then `scp`):
```
1_crosswalk_fetch/out/canonical_lakes_sf.rds # for site grouping (phase 5 and SB posting)
7b_temp_merge/out/temp_data_with_sources.feather # for 6_evaluation
```
(see the READMEs in those repos for instructions on how to create and use a conda environment to build the above targets.)

Results can now be packaged and posted using this repo (but run `mkdir log`, `mkdir out_data`, `mkdir out`, and `mkdir tmp` first if needed). These targets take <2.5 minutes apiece.
```sh
Rscript -e "library(scipiper); scmake('out_data/pgdl_config.csv', force=TRUE)" > log/pgdl_config.out 2>&1
nohup Rscript -e "library(scipiper); scmake('tmp/5_pgdl_pretrain_predictions_zips.yml')" > log/5_pgdl_pretrain_predictions_zips.out 2>&1 &
nohup Rscript -e "library(scipiper); scmake('out/5_pgdl_predictions_zips.yml')" > log/5_pgdl_predictions_zips.out 2>&1 &
nohup Rscript -e "library(scipiper); scmake('out/5_pgdl_test_predictions_zips.yml')" > log/5_pgdl_test_predictions_zips.out 2>&1 &
```

Then evaluate results like so, in R. (I tried running `R_ZIPCMD=$(which zip)` before launching an Rscript command, but that didn't seem to be effective; `Sys.setenv` does the trick, though.)
```sh
nohup Rscript -e "library(scipiper); Sys.setenv('R_ZIPCMD' = system('which zip', intern=TRUE)); scmake('out_data/pgdl_evaluation.csv')" > log/6_pgdl_evaluation.out 2>&1 &
```

Then post results in an interactive R session (to login to SB) with these commands:
```R
R
library(scipiper)
sbtools::authenticate_sb('cidamanager')
scmake('03_config_sb_data_pgdl')
scmake('05_predictions_sb_pgdl')
scmake('06_evaluation_sb_data_pgdl')
```

### Tallgrass configuration for PGDL outputs

```sh
conda update -n base -c defaults conda
conda create -n mntoha_release
source activate mntoha_release
conda install -c conda-forge proj4 gdal r-rgdal r-devtools r-maps r-mapdata r-maptools r-rgeos r-rjsonio r-RcppCNPy r-ggplot2 r-sf r-lwgeom r-dplyr r-tidyr r-readr r-progress r-BH r-hms r-generics r-lubridate r-feather r-plyr r-reticulate python
module load netcdf/gcc/64/4.6.1
```

Install more packages. Choose option 3 to not update already-installed packages.
```r
devtools::install_github('richfitz/remake')
devtools::install_github('USGS-R/scipiper')
install.packages(c('dataRetrieval', 'sbtools'))
devtools::install_github('USGS-R/meddle')
devtools::install_github('GLEON/GLMr')
devtools::install_github('USGS-R/glmtools')
```

dssecrets: On your desktop/laptop, paste https://github.com/USGS-CIDA/dssecrets/archive/master.tar.gz into a browser to download. Move to Tallgrass like so:
```sh
scp ~/Downloads/dssecrets-master.tar.gz tallgrass.cr.usgs.gov:/caldera/projects/usgs/water/iidd/datasci/lake-temp/mntoha-data-release/tmp
```
Then install when back on Tallgrass
```r
devtools::install_github("gaborcsardi/secret")
install.packages("tmp/dssecrets-master.tar.gz", repos = NULL)
```

After doing all of the above, you should be able to just load the environment with
```sh
source activate mntoha_release
```
in future sessions.

## Habitat metrics

Habitat metrics (TOHA + a number of annual metrics) are built on and should be posted directly from Yeti. The `log/07_habitat_xml_sb.csv` target should be built separately, on a machine that can build `spatial_metadata` from `1_spatial`.

```sh
ssh yeti.cr.usgs.gov
cd /cxfs/projects/usgs/water/iidd/data-sci/lake-temp/mntoha-data-release
```

These targets should be built in the `lake-temperature-out` project:
```
3_summarize/out/3_summarize_zip_pb0_toha.yml
3_summarize/out/annual_metrics_pb0.csv
3_summarize/out/3_summarize_zip_pgdl_toha.yml
3_summarize/out/annual_metrics_pgdl.csv
```

Then post results in an interactive R session (to login to SB) with the following commands. Commit the changes to the two CSV targets. Note that `gdal/3.1.0 proj/7.0.1` is part of the modules loaded. This is different from `lake-temperature-out` and needed to use meddle.
```R
module load legacy R/3.6.3 tools/nco-4.7.8-gnu tools/netcdf-c-4.3.2-intel gdal/3.1.0 proj/7.0.1
R
library(scipiper)
sbtools::authenticate_sb('cidamanager')
scmake('log/07_habitatPB_sb.csv')
scmake('log/07_habitatPGDL_sb.csv')
```

If you are only building those two targets and have not built any other part of the pipeline, you will need to adjust the `include` part of `remake.yml`. First, comment out `6_evaluation.yml` because it will fail on missing the file `src/scratch_plots.R`. Then, add `1_spatial.yml` so that it doesn't fail on `spatial_metadata` being an implicitly created target that must be a file. Now you can run the code above without errors. Revert these changes once done.

```
include:
  - 1_spatial.yml
  #- 6_evaluation.yml
  - 7_habitat.yml
```
