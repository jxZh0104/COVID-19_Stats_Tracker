> Originally developed in 2020. Cleaned up in 2026. Minor code edits and README writing was assisted by AI (Claude) to improve documentation and readability.

# COVID-19 Analysis Pipeline

A small pipeline that downloads the Johns Hopkins CSSE COVID-19 time-series data, cleans it, stores it as CSV files and in an SQLite database, and then lets you forecast case counts, draw animated maps, and look up numbers in simple GUIs. It works on **confirmed cases** only: worldwide by country, US by state, China by province, plus daily case counts for the city of Berkeley.

## What the pipeline does

The pipeline takes the Johns Hopkins (JHU CSSE) time series of confirmed cases, cleans it into tidy tables, and then lets you forecast, map and look up the numbers. It has five steps, and each step uses the output of the one before:

| Step | Script | Reads | Produces |
| --- | --- | --- | --- |
| 1. Download | `download-latest-JHU.sh` | the [CSSEGISandData/COVID-19](https://github.com/CSSEGISandData/COVID-19) repository on GitHub | `jhu/` (raw CSV files) |
| 2. Clean | `COVID-proj.R` | `jhu/`, `countries.csv`, `us_states.csv` | `COVID-19_cleaned_cases.RData` |
| 3. Build tables | `database.py` | the `.RData` file and the City of Berkeley open-data portal | `df_files/`, `data_entries/`, `sqlite3/COVID_database.db` |
| 4. Analyze | `Analysis_Script.py` | `df_files/` | forecast plots (PNG) and animated maps (HTML) |
| 5. Look up | `GUI.py`, `usGUI.py`, `globalGUI.py` | `sqlite3/COVID_database.db` | interactive windows |

### The data

Each dataset is a table with one row per region and one column per date (after the columns for name, latitude and longitude). It comes in two flavours:

- **total**: cumulative confirmed cases up to each date.
- **new**: daily new cases, the day-to-day difference of the totals.

The regions come at three levels: the **world** by country, the **US** by state and **China** by province. That gives six datasets, named `<global|us|china>_counts_<total|new>`, for example `us_counts_new`. Each one also has a `_geo` version (for example `us_counts_new_geo`) that keeps only the regions with known coordinates. The maps and the GUIs use the `_geo` versions. On top of these, the city of **Berkeley** has a small table of daily and cumulative cases.

### Step by step

#### 1. Download (`download-latest-JHU.sh`)

Makes a shallow `git clone` of the JHU repository into `jhu/` (replacing any earlier copy) and removes its `.git` folder. Only two files are used, both in `jhu/csse_covid_19_data/csse_covid_19_time_series/`: `time_series_covid19_confirmed_global.csv` and `time_series_covid19_confirmed_US.csv`.

#### 2. Clean (`COVID-proj.R`)

The JHU files have one row per province or region (world) and one row per county (US). The script adds these up to one row per country and one row per US state, attaches latitude and longitude from two lookup tables (`countries.csv` and `us_states.csv`), and computes the daily new cases. It saves four tables (total and new, for the world and for the US) in `COVID-19_cleaned_cases.RData`.

#### 3. Build tables (`database.py`)

Reshapes the four tables to *name, latitude, longitude, one column per date*, builds the China tables from the JHU rows for China's provinces (Taiwan is included as one of them), and creates the `_geo` versions. It writes:

- `df_files/`: one CSV per table. The CSVs are not rebuilt if the folder was already built today.
- `data_entries/`: one text file per dataset listing its region names. It is created if missing, and the automation scripts loop over these files.
- `sqlite3/COVID_database.db`: one table per `_geo` dataset (for example `global_counts_total` or `us_counts_new`), plus a `Berkeley` table of daily and cumulative cases downloaded from the City of Berkeley open-data portal.

#### 4. Analyze (`Analysis_Script.py`)

Reads the tables in `df_files/` and does what you ask for through command-line options:

| Option | What it does |
| --- | --- |
| `--forecast True` | Fits a curve to one region's history and extends it 10 days, with a prediction interval estimated by bootstrap: a logistic curve for cumulative counts (`*_total`) and a Gaussian curve for daily counts (`*_new`). The figure shows the whole history next to the last 30 days. Saved as `Forecast_Plot_<Name>_Total.png` or `Forecast_Plot_<Name>_New.png`. |
| `--forecast True --arima True` | Daily counts only. Fits an automatically chosen ARIMA model with weekly seasonality to the first 90% of the series, then forecasts the remaining days plus 10 more, with a confidence band. Saved as `Forecast_Plot_ARIMA_<Name>_New.png`. |
| `--map True` | Animated Plotly map of the counts over time, with a date slider: the world by country, the US by state, China by province. Needs a `_geo` dataset. `--scale` chooses a `continuous` (default) or `log` colour scale. Saved as `<region>_map_<total or new>_<scale>scale.html`, for example `global_map_total_logscale.html`. |
| `--print True` | Prints the selected table. |

The options that go with these:

- `--df` selects the dataset, for example `us_counts_new` (use a `_geo` name for maps).
- `--name` selects the region (use underscores instead of spaces). If the optional `fuzzywuzzy` package is installed, a name that is close to a valid one is matched automatically.
- `--show True` shows the result on screen, and `--save <folder>` saves it there.

Two examples:

```bash
# 10-day ARIMA forecast of daily new cases in California, shown on screen
python3 Analysis_Script.py --df us_counts_new --name California --forecast True --show True --arima True

# animated world map of cumulative cases on a log colour scale, saved in the current folder
python3 Analysis_Script.py --df global_counts_total_geo --map True --scale log --save .
```

#### 5. Look up (`GUI.py`, `usGUI.py`, `globalGUI.py`)

Small tkinter windows on top of the SQLite database. Enter a region, choose `New` or `Total`, and enter a date (`mm-dd-yy`) to see the number of confirmed cases, or plot the whole series. `GUI.py` also covers the Berkeley cases, `usGUI.py` is for US states and `globalGUI.py` for countries.

### Wrapper scripts

`download-latest-JHU.sh` is step 1 on its own. The other three scripts chain several steps:

- **`COVID-Analysis.sh`** is an interactive menu. Answer `true` to each prompt you want to run:
  - download the data and clean it (steps 1 and 2);
  - update the database (step 3);
  - open the GUI;
  - daily check on California: prints the daily new US cases, then an ARIMA forecast for California;
  - daily check on US cases: the same for the whole US, from the world table.

  If you skip the US check, it also offers to run `Analysis_Script.py` with arguments you type in (a forecast or a map).
- **`COVID-Analysis-Automation.sh`** is a batch run. It can download and clean the data first, then recreates the `automation/` folder (an existing one is deleted), writes the maps of all six `_geo` datasets on both colour scales to `automation/maps/`, and, for each dataset you confirm, saves a forecast plot for every region in `automation/forecasts/<dataset>/`.
- **`COVID-Analysis-Automation-Forecasts.sh`** runs only the per-region forecasts of the script above.

## Before you start

- **Tools:** `git`, R with the `tidyverse` package, and Python 3 with the packages imported by `utils.py` (`pyreadr`, `pandas`, `numpy`, `matplotlib`, `scipy`, `basemap`, `IPython`, `plotly`, `pycountry`, `pmdarima`, `statsmodels`) plus `tkinter` for the GUIs. `fuzzywuzzy` is optional.
- **Not included in this repository:**
  - `us_states.csv` and `countries.csv`, the latitude/longitude lookup tables read by `COVID-proj.R` (the comments in that script cite their sources).
  - `china-geojson-master/china (geojson).json`, needed for the China map (see the comment in `utils.py`).
- **Run order:** `Analysis_Script.py` and the GUIs need the output of `database.py`. It runs after the R step, and `COVID-Analysis.sh` does it when you answer `true` to "Update database?".
- **Data freshness:** the upstream JHU repository is archived (read-only) and was last updated in June 2023, so "latest data" means that final snapshot.

## How to run

0. Navigate to the current folder.
1. Run `bash download-latest-JHU.sh` in terminal to gather latest data.
2. Run `Rscript COVID-proj.R` in terminal for preliminary cleaning and gathering of gathered data.
3. Run `python3 database.py` in terminal to build the datasets and the SQLite database (make sure all the necessary modules have already been installed). It creates the `df_files/`, `data_entries/` and `sqlite3/` folders itself.
4. Run `python3 Analysis_Script.py` with the arguments you want in terminal to analyze and plot data, for example `python3 Analysis_Script.py --df us_counts_new --name California --forecast True --show True --arima True`.

   If you wish to personalize which data you want to analyze, run `python3 Analysis_Script.py -h` to see the necessary arguments to input.

**OR**

Simply run `bash COVID-Analysis.sh`.

- `GUI.py` generates a simple GUI for querying number of confirmed cases and can plot cases in Berkeley.
- `usGUI.py` generates a simple GUI for querying number of confirmed cases in different states of US and can plot them.
- `globalGUI.py` generates a simple GUI for querying number of confirmed cases in different countries and can plot them.
