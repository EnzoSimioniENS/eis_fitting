# EIS Fitting

> **Recommended environment**: this workflow is recommended for a Linux-like environment. On Windows, using WSL is advised to avoid path, dependency, and terminal-behaviour issues

Object-oriented Python workflow for preprocessing, fitting, analysing, and plotting electrochemical impedance spectroscopy (EIS) data from conductive hydrogel experiments.

The project was developed for automated fitting of conductive hydrogel impedance spectra but could be used with other materials given the appropriate equivalent circuits. It currently supports raw data preprocessing, optional automatic outlier removal, Lin-KK validation, equivalent-circuit fitting, fitted-parameter extraction, fitted-parameter error filtering, derived conductivity calculations, and generation of Bode, Nyquist, residual, and parameter-summary plots.

> **NB**: this repository only contains the documentation of the project for now, the rest of the code will be made public after the first publication.

---

## 1. Project overview

The workflow is organised around a small set of object-oriented components:

```text
main.py
    │
    ▼
global.json
    │
    ▼
EISPipeline
    │
    ├── ConfigLoader
    │       ├── reads the root-level global configuration
    │       ├── reads experiment-level metadata.json files
    │       ├── reads reusable method files from data/_methods/
    │       └── merges these settings into an Experiment object
    │
    ├── EISPreprocessor
    │       ├── cleans raw EIS files
    │       ├── converts BioLogic .mpr files when needed
    │       ├── splits frequency sweeps in distinct files
    │       └── can create pre-fitting Bode/Nyquist plots for manual preprocessing
    │
    ├── EISFitter
    │       ├── optionally removes outliers before fitting
    │       ├── optionally validates spectra with Lin-KK before fitting
    │       ├── fits selected equivalent circuits
    │       ├── saves one fitted-parameter CSV per circuit
    │       ├── calculates chi-square and reduced chi-square
    │       ├── calculates residuals and fitted-parameter uncertainties
    │       └── rejects non-identifiable fits when parameter error exceeds the configured threshold
    │
    └── EISAnalyser / EISPlotter
            ├── calculates derived quantities such as selected conductivities
            ├── creates per-sample R/Q or R/C parameter bar charts
            ├── creates residual plots when enabled
            └── creates one experiment-level summary plot per conductivity type
```

The aim is to keep each class responsible for one task.

| Class | Role |
|---|---|
| `EISPipeline` | Coordinates the complete workflow |
| `ConfigLoader` | Loads `global.json`, experiment `metadata.json`, and reusable method files |
| `Experiment` | Stores merged experiment settings, paths, selected circuits, geometry, and run options |
| `EISPreprocessor` | Converts, cleans, splits, and prepares raw EIS data |
|`PreprocessPlotter`| Plots Bode (phase and modulus) with visible points for manual removal of outliers |
| `EISFitter` | Fits equivalent circuits, calculates fit metrics, and saves fitted parameters |
| `FitMetrics` | Calculates residuals, chi-square, and reduced chi-square |
| `FitPlotter` | Creates Bode, Nyquist, and residual plots of experimental vs fitted data |
| `EISAnalyser` | Adds derived quantities, especially conductivities requested in the circuit mapping |
| `AnalysisPlotter` | Creates parameter and conductivity bar charts |
| `FigureSaver` | Saves figures according to global saving preferences, e.g. PNG, SVG, transparent background, dpi |

---

## 2. Installation and execution

### 2.1 Clone the repository:

```bash
git clone git@github.com:<your-username>/eis_fitting.git
cd eis_fitting
```

### 2.2 Create and activate a virtual environment (recommended):

Using conda:

```bash
conda create -n eis_fitting python=3.11
conda activate eis_fitting
```

### 2.3 Install dependencies

Upgrade `pip`:

```bash
pip install --upgrade pip
```

Install the project dependencies (`requirements.txt` can be deleted afterwards):

```bash
pip install -r requirements.txt
```

If the package structure already supports editable installation, install it locally with:

```bash
pip install -e .
```

If not, run from the repository root so that the local imports work correctly.

### 2.4 Running the project

Before running the workflow, make sure that the data folder, `metadata.json`, `_methods/` files, `global.json`, and circuit definitions follow the structure described in the sections below. Then, from `projects/` run :

```bash
python -m eis_fitting.main
```


---

## 3. Data organisation

The project is designed around:

1. a root-level `global.json` file in the project repository
2. one `_methods/` folder with the data, containing reusable method files
3. experiment folders organised by user/project/experiment

Current architecture:

```text
eis_fitting/
├── main.py
├── global.json
├── pipeline.py
├── circuits.py
|
├── config/
├── core/
├── preprocessing/
├── fitting/
├── analysis/
└── plotting/

data/
├── _methods/
│   └── conductive_hydrogel.json
│
└── <user>/<project>/<experiment>/
    ├── metadata.json
    ├── sample_1.csv
    ├── sample_2.csv
    └── ...
```

Example:

```text
data/
├── _methods/
│   └── conductive_hydrogel.json
│
└── Bel/eis_flex/AB-01-001/
    ├── AB-01-001_01_01.csv
    ├── AB-01-001_01_02.csv
    └── metadata.json
```

The experiment folder should contain only:

- the experiment `metadata.json`
- raw experimental files, such as `.csv` or `.mpr`

It should not contain manually created fitting, analysis, or plotting folders. These are created automatically by the pipeline.

The `_methods/` folder stores reusable method-level scientific settings, such as sample geometry. Figure-saving preferences are not stored in method files anymore; they are controlled globally from `global.json`.

---

## 4. Local mode and workspace mode

The workflow can be used in two data-management modes (see section `5.1 Global configuration file`).

### 4.1 Workspace mode (recommended)

In workspace mode, raw data and processed data are separated.

The raw data stay in:

```text
data/<user>/<project>/<experiment>/
```

and the processed outputs are written to a separate workspace:

```text
workspace/processed/<user>/<project>/<experiment>/
```

Example:

```text
raw data:
data/Bel/eis_flex/AB-01-001/

processed data:
workspace/processed/Bel/eis_flex/AB-01-001/
```

The processed experiment folder then contains:

```text
workspace/processed/Bel/eis_flex/AB-01-001/
├── csv/
│   ├── AB-01-001_01_01.csv
│   └── AB-01-001_01_02.csv
│
└── fits/
    ├── AB-01-001_circuit_2.csv
    └── plots/
        ├── AB-01-001_01_01/
        ├── AB-01-001_01_02/
        └── AB-01-001_circuit_2_Ri_conductivity_all_samples.png
```

Workspace mode is recommended for larger projects because the original raw data remain untouched and all generated files are stored separately.

### 4.2 Local mode

In local mode, processed files are written inside the experiment folder itself.

Example:

```text
data/
└── Bel/eis_flex/AB-01-001/
    ├── AB-01-001_01_01.csv
    ├── AB-01-001_01_02.csv
    ├── metadata.json
    └── processed/
        ├── csv/
        └── fits/
            ├── AB-01-001_circuit_2.csv
            └── plots/
```

This mode keeps the raw and processed data together. It is simple and useful for small projects or quick testing.

---

## 5. Metadata, method files, and global configuration

The current architecture uses three JSON layers:

1. a root-level **global configuration file**
2. an experiment-level **metadata file**
3. a reusable **method file**

The separation is:

```text
global.json      → paths, run options, plotting options, save preferences
metadata.json    → experiment description, method name, selected circuits
_methods/*.json  → reusable scientific method settings, (only cell geometry for now)
```

This avoids repeating geometry and run options in every experiment folder.

---

### 5.1 Global configuration file

The root-level `global.json` file is placed next to `main.py`. It controls where the data are read from, where outputs are written, which workflow steps are enabled, how spectra are validated before fitting, how non-identifiable fits are rejected, and how figures are saved.

Example:

```json
{
    "paths": {
        "data_path": "/home/enzos/data/",
        "user": "enzo",
        "project": "test",
        "output_mode": "workspace",
        "output_path": "/home/enzos/data_processed"
    },
    "run": {
        "pre_plot": false,
        "plot": true,
        "compute_mean_parameters": false
    },
    "outlier_removal": {
        "enabled": true,
        "threshold": 1.5,
        "features": [
            "log_modulus",
            "phase"
        ]
    },
    "validation": {
        "display_chi_square": true,
        "fitting_frequency_range": [
            10,
            1e6
        ],
        "plot_residuals": true,
        "lin_kk": {
            "enabled": true,
            "plot": true,
            "reject_on_fail": true,
            "c": 0.85,
            "max_M": 50,
            "fit_type": "complex",
            "add_cap": true
        },
        "error_percent_param_threshold": 200
    },
    "plotting": {
        "R_and_Q": {
            "left_ylim": [
                1e-1,
                1e15
            ],
            "right_ylim": [
                1e-10,
                1e-1
            ]
        },
        "conductivities": {
            "left_ylim": [
                null,
                null
            ],
            "normalize": true,
            "cmap_name": "viridis",
            "log_scale": false,
            "legend": {
                "rotation": 0,
                "ha": "center",
                "fontsize": 14
            }
        }
    },
    "save_preferences": {
        "transparent": false,
        "png": true,
        "svg": false,
        "dpi": 250
    },
    "full_terminal": true
}
```

#### `paths`

| Field | Meaning |
|---|---|
| `paths.data_path` | Root folder containing the raw data and the `_methods/` folder |
| `paths.methods_path` | Optional folder containing reusable method files. If omitted, this can default to `<data_path>/_methods` |
| `paths.user` | User/person folder inside `data_path` |
| `paths.project` | Project folder inside `data_path/user` |
| `paths.output_mode` | Either `"local"` or `"workspace"` |
| `paths.output_path` | Root output folder used in workspace mode. Required if `output_mode` is `"workspace"` |

#### `run`

| Field | Meaning |
|---|---|
| `run.pre_plot` | If `true`, preprocess only and create preliminary plots. No fitting or analysis is performed afterwards |
| `run.fit` | Optional. If `true`, fitting is allowed when no fitted CSVs exist yet. If omitted, the workflow can default to fitting enabled |
| `run.plot` | Master switch for plot generation. If `false`, the workflow can still fit and write CSV files without creating figures |
| `run.compute_mean_parameters` | If `true`, appends one `MEAN` row to each circuit parameter CSV. The mean is computed over successful fits only. Parameter uncertainties are propagated from individual fitted standard deviations using $u(\bar{x}) = \sqrt{\sum_i u_i^2}/n$. Since $\chi^2$ and $\chi^2_{\mathrm{red}}$ do not have individual fitted uncertainties, their uncertainty is estimated from the sample-to-sample standard deviation across successful fits. |

#### `outlier_removal`

The `outlier_removal` block controls automatic removal of points before fitting.

```json
"outlier_removal": {
    "enabled": true,
    "threshold": 1.5,
    "features": [
        "log_modulus",
        "phase"
    ]
}
```

| Field | Meaning |
|---|---|
| `outlier_removal.enabled` | If `true`, the fitter removes outliers before applying the fitting frequency range and before equivalent-circuit fitting |
| `outlier_removal.threshold` | Quantile/IQR threshold. A value of `1.5` corresponds to the standard IQR outlier rule; larger values such as `2.0` or `3.0` are more conservative |
| `outlier_removal.features` | List of features used to detect outliers. Supported entries can include `log_modulus`, `phase`, `real`, and `imag` depending on the fitter implementation |

The IQR rule removes points outside:

$$
Q_1 - t \times IQR
\quad \text{to} \quad
Q_3 + t \times IQR
$$

where $t$ is `outlier_removal.threshold`.

The recommended default is to use:

```json
"features": [
    "log_modulus",
    "phase"
]
```

because these two quantities are usually the most convenient for identifying isolated spikes in Bode plots.

The parameter CSV can record how many points were removed, for example:

```text
n_points_before_outlier_removal
n_points_after_outlier_removal
n_invalid_points_removed
n_quantile_outliers_removed
n_outliers_removed
```

#### `validation`

The `validation` block contains settings used to validate spectra, select the fitted frequency range, display fit-quality metrics, and reject poorly identifiable fits.

| Field | Meaning |
|---|---|
| `validation.display_chi_square` | If `true`, fitted Bode/Nyquist plots display chi-square and reduced chi-square information |
| `validation.fitting_frequency_range` | Frequency window used for fitting, in Hz. Use `null` to fit the full spectrum, or `[f_min, f_max]` to restrict the fit. One-sided ranges such as `[10, null]` or `[null, 1000000]` are also supported |
| `validation.plot_residuals` | If `true`, residual plots are generated when fit plotting is enabled |
| `validation.error_percent_param_threshold` | Maximum allowed fitted-parameter percentage error. If one fitted parameter has a percentage error above this threshold, the fit is rejected |
| `validation.lin_kk` | Nested block controlling Lin-KK validation |

For example:

```json
"error_percent_param_threshold": 200
```

means that a fit is rejected if any fitted parameter has:

```text
<parameter>_percent_error > 200
```

This replaces a hard-coded rejection limit and makes the identifiability criterion explicit from `global.json`.

#### `validation.lin_kk`

The Lin-KK block controls Kramers-Kronig consistency validation before equivalent-circuit fitting.

```json
"lin_kk": {
    "enabled": true,
    "plot": true,
    "reject_on_fail": true,
    "c": 0.85,
    "max_M": 50,
    "fit_type": "complex",
    "add_cap": true
}
```

| Field | Meaning |
|---|---|
| `validation.lin_kk.enabled` | If `true`, Lin-KK validation is performed before fitting |
| `validation.lin_kk.plot` | If `true`, Lin-KK validation plots are generated when the plotter supports them |
| `validation.lin_kk.reject_on_fail` | If `true`, equivalent-circuit fitting is rejected when Lin-KK validation fails |
| `validation.lin_kk.c` | Under/over-fitting control parameter used by `impedance.py` Lin-KK validation. The common default is `0.85` |
| `validation.lin_kk.max_M` | Maximum number of RC elements considered by the Lin-KK validation model |
| `validation.lin_kk.fit_type` | Which part of the impedance is used during Lin-KK fitting. Typical values are `"real"`, `"imag"`, or `"complex"` |
| `validation.lin_kk.add_cap` | If `true`, adds an additional capacitance term in the Lin-KK validation model. This can help for spectra with strongly capacitive low-frequency behaviour |

Lin-KK validation checks whether the measured spectrum is consistent with the assumptions required for EIS interpretation: linearity, causality, and stability. It does not prove that a chosen equivalent circuit is physically correct; it only checks whether the spectrum itself is internally consistent enough to be interpreted.

If `reject_on_fail` is enabled and Lin-KK fails, the fit row is saved with:

```text
fit_success = False
error = Fit rejected: Lin-KK validation failed (...)
```

and no fitted plots are generated for that sample/circuit.

#### `plotting`

Manual plot settings for analysis plots. Each entry corresponds to a figure type.

For R/Q or R/C bar plots:

```json
"R_and_Q": {
    "left_ylim": [1e-1, 1e15],
    "right_ylim": [1e-10, 1e-1]
}
```

| Field | Meaning |
|---|---|
| `plotting.R_and_Q.left_ylim` | Limits for the left axis, usually resistance |
| `plotting.R_and_Q.right_ylim` | Limits for the right axis, usually CPE magnitude or capacitance |

For conductivity summary plots:

```json
"conductivities": {
    "left_ylim": [
        null,
        null
    ],
    "normalize": true,
    "cmap_name": "viridis",
    "log_scale": false,
    "legend": {
        "rotation": 0,
        "ha": "center",
        "fontsize": 14
    }
}
```

| Field | Meaning |
|---|---|
| `plotting.conductivities.left_ylim` | Y-axis limits for conductivity summary plots. Use `[null, null]` for automatic scaling |
| `plotting.conductivities.normalize` | If `true`, conductivity values are normalized, typically by the maximum value in the plotted conductivity series |
| `plotting.conductivities.cmap_name` | Matplotlib colormap used to colour bars across samples, for example `"viridis"`, `"Blues"`, `"magma"`, or `"cividis"` |
| `plotting.conductivities.log_scale` | If `true`, the conductivity y-axis is plotted on a logarithmic scale |
| `plotting.conductivities.legend.rotation` | Rotation angle of the sample labels on the x-axis |
| `plotting.conductivities.legend.ha` | Horizontal alignment of the sample labels, for example `"center"`, `"right"`, or `"left"` |
| `plotting.conductivities.legend.fontsize` | Font size of the sample labels |

Use `null` in JSON to keep automatic Matplotlib scaling.

#### `save_preferences`

| Field | Meaning |
|---|---|
| `save_preferences.transparent` | If `true`, figures are saved with a transparent background |
| `save_preferences.png` | If `true`, figures are saved as PNG files |
| `save_preferences.svg` | If `true`, figures are saved as SVG files |
| `save_preferences.dpi` | Resolution used when saving raster figures such as PNG. Defaults to `300` if not specified. Lower values speed up plotting and reduce file size; higher values are useful for publication-quality figures |

#### `full_terminal`

If `true`, the workflow prints more information about selected experiments, paths, outlier removal, skipped fits, failed fits, and generated outputs.

---

### 5.2 Experiment metadata file

Each experiment must contain a `metadata.json` file.

Example:

```json
{
    "method": "conductive_hydrogel",
    "description": "0.1% w/v PEDOT:PSS, no Upy",
    "circuits": [
        "ionic_1"
    ]
}
```

The `method` field points to a method file in:

```text
data/_methods/
```

For example:

```json
"method": "conductive_hydrogel"
```

loads:

```text
data/_methods/conductive_hydrogel.json
```

Run options such as `pre_plot` and `plot`, validation options such as `plot_residuals`, `display_chi_square`, `fitting_frequency_range`, and Lin-KK settings, and analysis options such as conductivity plot settings are not stored in each experiment metadata but are controlled globally from `global.json`.

| Field | Meaning |
|---|---|
| `method` | Name of the method file to load from `data/_methods/`, without the `.json` extension |
| `description` | Free-text description of the experiment |
| `circuits` | List of circuit names to fit. These names must exist in the circuit dictionary |

---

### 5.3 Method file

Method files are stored in:

```text
data/_methods/
```

Example:

```text
data/_methods/conductive_hydrogel.json
```

Example method file:

```json
{
    "geometry": {
        "thickness_cm": 0.1,
        "SD_thickness_cm": 0.01,
        "area_cm2": 1.0,
        "SD_area_cm2": 0.2
    }
}
```

| Field | Meaning |
|---|---|
| `geometry.thickness_cm` | Sample thickness in cm |
| `geometry.SD_thickness_cm` | Uncertainty of thickness measurement |
| `geometry.area_cm2` | Electrode/sample area in cm² |
| `geometry.SD_area_cm2` | Uncertainty over area measurement |

---

## 6. Running the workflow

### 6.1 `main.py` file

The workflow is launched from `main.py`. The current structure keeps `main.py` short, loading `global.json`, creating the pipeline, and running the project specified in the global configuration.

The person, project, input path, output mode, and output path are read from `global.json`, so they do not need to be edited directly in `main.py`.

The run behaviour (see following subsections) is controlled by `global.json` and by whether processed results already exist.

### 6.2 Preprocessing-only mode

If:

```json
"pre_plot": true
```

then the pipeline only preprocesses the data and generates preliminary plots.

Typical use:

```text
raw files
    ↓
cleaned CSVs
    ↓
pre-fit Bode/Nyquist plots
```

This mode is intended for manual inspection and cleaning. After checking the preliminary plots, the user can manually remove bad points from the cleaned CSV files before running the workflow without any more pre-plotting to proceed with the fitting.

### 6.3 Fitting mode

If:

```json
"pre_plot": false,
"fit": true
```

and no fit results are available yet, the pipeline runs fitting and analysis.

Typical use:

```text
processed CSVs
    ↓
optional outlier removal and Lin-KK validation
    ↓
equivalent-circuit fitting
    ↓
fit metrics and residuals
    ↓
parameter CSVs
    ↓
fitted Bode/Nyquist/residual plots
    ↓
analysis plots
```

### 6.4 Fitting without plotting

If plotting is disabled in `global.json`, the fitter can generate fitted-parameter CSV files without producing figures.

```json
"plot": false
```

This is useful for quick batch fitting or when only the numerical results are needed.

### 6.5 Re-plotting without refitting

If fit CSV files already exist, the pipeline does not need to refit the data. To regenerate plots without refitting, delete only the plot folder:

```bash
rm -rf <processed_experiment>/fits/plots
```

Then make sure plotting is enabled in `global.json`:

```json
"plot": true
```

and rerun the workflow.

The fitted parameter CSV files are kept, while the plotter can recreate the Bode, Nyquist, residual, and analysis plots with new plotting limits, saving format, dpi, ...

### 6.6 Typical workflow

1. Set `pre_plot=true` and run the workflow to generate preliminary plots.
2. Inspect the cleaned CSV files and remove bad points if needed.
3. Set `pre_plot=false`, `fit=true`, and `plot=true`.
4. Run the workflow to fit the selected circuits and generate plots.
5. Inspect Nyquist, Bode, residual plots, Lin-KK results, parameter errors, and conductivity summaries.
6. To change plotting style only, delete `fits/plots/` and rerun with `plot=true`.

---

## 7. Output structure

A typical processed experiment has the following structure:

```text
processed/<person>/<project>/<experiment>/
├── csv/
│   ├── sample_1.csv
│   ├── sample_2.csv
│   └── ...
│
└── fits/
    ├── <experiment>_<circuit_1>.csv
    ├── <experiment>_<circuit_2>.csv
    └── plots/
        ├── sample_1/
        │   ├── circuit_1/
        │   │   ├── sample_1_circuit_1_bode_fit.png
        │   │   ├── sample_1_circuit_1_nyquist_fit.png
        │   │   ├── sample_1_circuit_1_residuals.png
        │   │   └── sample_1_circuit_1_R_and_Q.png
        │   |
        │   └── circuit_2/
        │
        ├── sample_2/
        │
        ├── all_runs/
        │   └── combined plots, if enabled
        │
        ├── analysis/
        │   └── optional analysis-level plots
        │
        ├── <experiment>_<circuit_1>_Ri_conductivity_all_samples.png
        └── <experiment>_<circuit_1>_Rbulk_conductivity_all_samples.png
```

If a fit is rejected because Lin-KK validation fails or because one or more parameter percentage errors exceed `validation.error_percent_param_threshold`, the parameter CSV records the failure. In that case, no Bode, Nyquist, residual, or bar plots are generated for that failed fit. Depending on when the output folders are created, an empty plot folder may remain; this is expected and indicates that the fit was skipped after validation.

--- 

## 8. Equivalent circuits

Equivalent circuits are defined in the circuit dictionary, usually in a file such as:

```text
projects/eis_fitting/circuits.py
```

Each circuit entry contains:

- a circuit string in `impedance.py` syntax
- an initial guess for each component parameter to be used by the complex, non-linear, least squares fitting method
- a mapping dictionary for readable parameter names, colours, plotting order, and conductivity calculation

Example:

```python
CIRCUITS = {
    "circuit_1": {
        "string": "p(R_1-CPE_1,R_2)",
        "guess": [
            1000,  # R_1 = R_i
            1e-6,  # CPE_1_0 =Q_DL
            0.9,  # CPE_1_1 = alpha_DL
            1e7,  # R_2 = R_e
        ],
        "mapping": {
            "R_1": {
                "name": "R_i",
                "colour": "#80D4F8",
                "compute_conductivity": True,
                "order": 0,
            },
            "R_2": {
                "name": "R_e",
                "colour": "#F37F8C",
                "order": 1,
            },
            "CPE_1_0": {
                "name": "Q_DL",
                "colour": "#7C84C2",
                "order": 0,
            },
            "CPE_1_1": {
                "name": "alpha_DL",
                "colour": "#7C84C2",
                "order": 0,
            },
        },
    },
    ...
}
```

In this example, conductivities are computed only for `R_i` because it is the only resistance with:

```python
"compute_conductivity": True
```

Plotting examples :

<p align="center">
  <img src="./resources/DK_circuit_1_bar.png" height="300">
  <img src="./resources/circuit_2_Ri_conductivity_all_samples.png" height="300">
</p>

> Note that the names displayed in the parameter bars do not follow the `impedance.py` string naming convention directly. They use the readable names defined in the circuit mapping.

### Circuit syntax

The circuit syntax follows the `impedance.models.circuits.circuits.CustomCircuit` convention.

| Syntax | Meaning |
|---|---|
| `R_1-R_2` | Series connection |
| `p(R_1,CPE_1)` | Parallel connection |
| `CPE_1` | Constant phase element with two parameters: $Q$: `CPE_1_0` and $\alpha$: `CPE_1_1` from $Z_{CPE}=\frac{1}{Q(j\omega)^{\alpha}}$ |

### Mapping fields

| Field | Meaning |
|---|---|
| `name` | Readable parameter name written to the fitted parameter CSV. |
| `colour` | Colour used in parameter bar plots. |
| `order` | Plotting order for parameters of the same family. |
| `compute_conductivity` | Optional boolean. If `true`, conductivity is calculated from this resistance. |

The `compute_conductivity` field is only meaningful for resistance elements.

--- 

## 9. Fitting

For each selected circuit and each sample, the fitter:

1. reads the processed experimental EIS CSV
2. builds the complex impedance array
3. optionally removes outliers using the `outlier_removal` block from `global.json`
4. optionally restricts the data to `validation.fitting_frequency_range`
5. optionally validates the spectrum using Lin-KK validation
6. creates a `CustomCircuit`
7. fits the circuit using the supplied initial guesses
8. predicts the fitted impedance
9. extracts fitted parameters
10. calculates standard deviations and percentage errors where possible
11. calculates residuals, chi-square, and reduced chi-square through `fitting/fit_metrics.py`
12. rejects the fit if any fitted parameter has a percentage error above `validation.error_percent_param_threshold`
13. saves the result to a circuit-specific CSV
14. generates fitted Bode, Nyquist, residual, Lin-KK, and analysis plots if plotting is enabled and the fit passed validation

The fitted-parameter error threshold is used to reject visually good but non-identifiable fits. A fit can have a good chi-square and scattered residuals while still having poorly determined parameters. In that situation, the equivalent circuit may reproduce the spectrum, but the individual elements should not be interpreted physically.

Rejected fits are stored in the CSV with messages such as:

```text
fit_success = False
error = Fit rejected: parameter uncertainty above 200% for ...
```

or:

```text
fit_success = False
error = Fit rejected: Lin-KK validation failed (...)
```

and are not plotted.

--- 

## 10. Fit-quality metrics and residuals

The fitting workflow evaluates the quality of each equivalent-circuit fit using two complementary approaches:

1. numerical fit-quality metrics, mainly `chi_square` and `reduced_chi_square`
2. residual plots, which show how the fitting error is distributed across frequency

These metrics are useful for comparing fits, but they should not be interpreted as proof that a circuit is physically unique or correct.

### 10.1 Complex impedance and residuals

For each frequency point, the experimental impedance is complex:

$$
Z_{\mathrm{exp}} = Z_{\mathrm{exp}}' + jZ_{\mathrm{exp}}''
$$

and the fitted circuit predicts:

$$
Z_{\mathrm{fit}} = Z_{\mathrm{fit}}' + jZ_{\mathrm{fit}}''
$$

The residual is the difference between the experimental and fitted impedance:

$$
r = Z_{\mathrm{exp}} - Z_{\mathrm{fit}}
$$


Because EIS data are complex, the residual contains both a real and an imaginary contribution:

$$
r(\omega) =
r'(\omega)
+
jr''(\omega)
$$

The `fit_metrics.py` module centralises these calculations. It is responsible for computing:

* complex residuals
* relative real residuals
* relative imaginary residuals
* absolute residuals
* `chi_square`
* `reduced_chi_square`

### 10.2 Chi-square

The chi-square value, $\chi^2$, measures the overall discrepancy between the experimental impedance spectrum and the fitted circuit response.

Since impedance values can vary by several orders of magnitude across the frequency range, this project uses a relative chi-square by default. The squared difference between experimental and fitted impedance is normalized by the magnitude of the experimental impedance:

$$
\chi^2 =
\sum_{\omega}
\frac{
\left|Z_{\mathrm{exp}}(\omega) - Z_{\mathrm{fit}}(\omega)\right|^2
}{
\left|Z_{\mathrm{exp}}(\omega)\right|^2
}
$$

This normalization prevents high-impedance points from dominating the fit-quality metric.

This is also consistent with the fitting procedure used in `fitting/fitter.py`, where the circuit is fitted using modulus weighting:

```python
circuit.fit(freq, z, weight_by_modulus=True)
```

In practical terms:

* a lower `chi_square` means that the fitted curve is numerically closer to the experimental data
* a higher `chi_square` means that the fitted curve deviates more strongly from the experimental data

However, raw `chi_square` depends on the number of data points and the number of fitted parameters. For this reason, it is mainly useful when comparing fits with the same circuit and the same frequency range.

---

### 10.3 Reduced chi-square

The reduced chi-square, $\chi^2_{\mathrm{red}}$, normalizes $\chi^2$ by the number of degrees of freedom of the fit:

$$
\chi^2_{\mathrm{red}} =
\frac{\chi^2}{\nu}
$$

where $\nu$ is the number of degrees of freedom of the fit.

For complex EIS data, each frequency point contributes two fitted values:

* the real impedance, $Z'$
* the imaginary impedance, $Z''$

Therefore, if there are $N$ valid frequency points and $p$ fitted circuit parameters, the number of degrees of freedom is calculated as:

$$
\nu = 2N - p
$$

Thus:

$$
\chi^2_{\mathrm{red}} =
\frac{\chi^2}{2N - p}
$$

The reduced chi-square is more useful than raw chi-square when comparing candidate circuits with different numbers of fitted parameters. A more complex circuit can often reduce raw $\chi^2$ simply because it has more adjustable parameters. Dividing by the degrees of freedom partially accounts for this increase in model complexity.

In the workflow, `reduced_chi_square` is therefore the preferred numerical metric for comparing different circuit fits to the same dataset.

---

### 10.4 Residual plots

While `chi_square` and `reduced_chi_square` give a single numerical estimate of fit quality, residual plots show where and how the model fails across the frequency range.

Residual plots are generated when the following option is set in `global.json`:

```json
"validation": {
    "plot_residuals": true
}
```

The residual plots show, across the whole frequency range, the relative deviation between the experimental and fitted impedance (normalized by the magnitude of the experimental impedance).

These values are expressed as percentages, making it easier to compare residuals across frequency ranges with very different impedance magnitudes.

A good residual plot should show points randomly scattered around zero, without a clear frequency-dependent structure. A structured residual pattern indicates systematic error in the fit.

<p align="center">
  <img src="./resources/DK_circuit_1_residuals.png" height="300">
</p>

Systematic residual patterns are important warning signs. For example:

| Residual behaviour                              | Possible interpretation                                                                                     |
| ----------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Random scatter around zero                      | The circuit captures the main impedance response reasonably well.                                           |
| Wave-like or sinusoidal residuals               | The circuit is missing one or more frequency-dependent processes.                                           |
| Residuals mostly above or below zero            | The model is biased and systematically over- or underestimates the data.                                    |
| Large low-frequency residuals                   | Possible missing electrode polarization, diffusion, drift, or slow relaxation.                              |
| Large high-frequency residuals                  | Possible missing contact resistance, inductive artefact, cable effect, or high-frequency measurement error. |
| Small residuals but very large parameter errors | The fit may be numerically good but poorly identifiable.                                                    |

---

### 10.5 Displaying fit metrics on plots

The fitted Bode and Nyquist plots can display both `chi_square` and `reduced_chi_square` directly on the figure when the following option is set in `global.json`:

```json
"validation": {
    "display_chi_square": true
}
```

This makes it easier to visually compare the fitted spectrum and the numerical fit quality at the same time.

---

### 10.6 How to interpret fit quality

A fit should not be judged from `chi_square` or `reduced_chi_square` alone (see *https://www.gamry.com/application-notes/EIS/fit-in-eis/*)

In practice, the fit should be considered reliable only when the following conditions are reasonably satisfied:

* the Nyquist and Bode plots show good visual agreement between experimental and fitted data
* `reduced_chi_square` is low compared with other candidate circuits
* residuals are randomly distributed around zero
* fitted parameters are physically reasonable
* fitted parameter uncertainties are acceptable
* no covariance warning is reported
* the circuit elements have a meaningful physical interpretation
* the same circuit gives consistent trends across related samples

A fit with a low `reduced_chi_square` but structured residuals should be interpreted cautiously. This means the circuit reproduces the overall shape of the impedance spectrum but does not fully capture all frequency-dependent processes.

Similarly, a fit with a low `reduced_chi_square` but very large parameter standard deviations is considered poorly identifiable. In the current workflow, fits are rejected when any fitted parameter has a percentage error greater than `validation.error_percent_param_threshold`.

---

## 11. Parameter CSV files

The fitter saves one CSV file per circuit:

```text
fits/<experiment>_<circuit_name>.csv
```

Each CSV should contain only the parameters relevant to that circuit.

For example, one circuit CSV may contain:

```text
sample
circuit_name
circuit_string
fit_success
error
chi_square
reduced_chi_square
n_points_before_outlier_removal
n_points_after_outlier_removal
n_invalid_points_removed
n_quantile_outliers_removed
n_outliers_removed
lin_kk_success
lin_kk_error
lin_kk_M
lin_kk_mu
R_i
SD_R_i
R_i_percent_error
R_bulk
SD_R_bulk
R_bulk_percent_error
Q_bulk
SD_Q_bulk
Q_bulk_percent_error
alpha_bulk
SD_alpha_bulk
alpha_bulk_percent_error
Q_DL
SD_Q_DL
Q_DL_percent_error
alpha_DL
SD_alpha_DL
alpha_DL_percent_error
conductivity_S_cm_R_i
SD_conductivity_S_cm_R_i
conductivity_S_cm_R_bulk
SD_conductivity_S_cm_R_bulk`
f_min_fit_Hz
f_max_fit_Hz
n_fit_points
```

If `run.compute_mean_parameters` is enabled, the CSV contains an additional summary row:

```text
sample = MEAN
fit_status = mean_all_successful_samples
is_summary = True
n_mean_samples = <number of successful fits>
```

If a fit is rejected, the CSV row still records the sample, circuit, `fit_success=False`, the error message, and `NaN` fit metrics. This makes failed fits visible without silently dropping samples.

When a restricted fitting frequency range is used, the CSV records the actual frequency window used for each fit. This makes it possible to trace whether parameters were obtained from the full spectrum or from a selected frequency interval.

When automatic outlier removal and Lin-KK validation are enabled, the CSV can also record outlier-removal counts and Lin-KK diagnostic values. This makes failed or accepted fits traceable from the numerical output alone.

---

## 12. Conductivity calculation

The analyser calculates conductivities from fitted resistances only when requested in the circuit mapping.

For a resistance entry in `circuits.py`:

```python
"R_1": {
    "name": "R_i",
    "colour": "#80D4F8",
    "order": 0,
    "compute_conductivity": True,
}
```

the analyser writes:

```text
conductivity_S_cm_R_i
SD_conductivity_S_cm_R_i
```

The conductivity is calculated as:

$$\sigma=\frac{l}{A\times R}$$

with $l$ and $A$ the geometric parameters given in `_methods/method.json`
 
The uncertainty is propagated from:

- thickness uncertainty
- area uncertainty
- resistance uncertainty

The uncertainty over the computed conductivity is given by the following propagation formula :

$$
u(\sigma) =
\sigma \times
\sqrt{
\left(\frac{u(l)}{l}\right)^2
+
\left(\frac{u(A)}{A}\right)^2
+
\left(\frac{u(R)}{R}\right)^2
}
$$

> Note that only resistances marked with `"compute_conductivity": True` are converted into conductivities, this is useful since not all fitted resistances represent a *through-sample* resistance geometrically

| Resistance type | Usually compute conductivity ? | Reason |
|---|---:|---|
| $R_i$ | Yes | Intended to represent ionic through-sample resistance |
| $R_{bulk}$ | Yes, if physically assigned to bulk transport. | Bulk pathway across the sample geometry |
| $R_e$ | Maybe | Highly depends on the nature of the system/networks |
| $R_c$ | No | Contact resistance is not a bulk material resistance |
| $R_{ct}$ | No | Charge-transfer resistance is an interfacial kinetic parameter |
| $R_s$ | Usually no | Often includes solution, lead, or series contributions |

The conductivity plots are generated directly from the conductivity columns present in the parameter CSV. Therefore, the analysis step decides which conductivities exist, and the plotting step simply plots them.

For each conductivity type, one separate experiment-level plot is created. For example:

```text
fits/plots/<experiment>_<circuit>_Ri_conductivity_all_samples.png
fits/plots/<experiment>_<circuit>_Rbulk_conductivity_all_samples.png
```

> This avoids combining conductivity estimates that correspond to physically distinct processes within the same figure, since these decoupled conductivity contributions cannot be straightforwardly related to macroscopic material properties.

---

## 13. Plotting

### 13.1 Preliminary plots

Created during preprocessing when `pre_plot` is enabled :

<p align="center">
  <img src="./resources/Portugal25_R1_2.png" height="300">
</p>

These are useful for identifying bad frequency ranges, outliers, or corrupted data before fitting.

### 13.2 Fitted Bode plots

For each sample and circuit:

```text
fits/plots/<sample>/<circuit>/<sample>_<circuit>_bode_fit.png
```

These show:

- experimental impedance magnitude and phase
- fitted circuit response
- chi-square and reduced chi-square, if enabled in `global.json` (here `true`)

<p align="center">
  <img src="./resources/DK_circuit_1_bode.png" height="300">
</p>

### 13.3 Fitted Nyquist plots

For each sample and circuit:

```text
fits/plots/<sample>/<circuit>/<sample>_<circuit>_nyquist_fit.png
```

<p align="center">
  <img src="./resources/Portugal25_R1_1_circuit_1_nyquist.png" height="300">
</p>

These show:

- experimental complex and real parts of the impedance
- corresponding fitted circuit response
- chi-square and reduced chi-square, if enabled in `global.json` (here `false`)

### 13.4 Residual plots

For each successful sample/circuit fit:

```text
fits/plots/<sample>/<circuit>/<sample>_<circuit>_residuals.png
```

<p align="center">
  <img src="./resources/DK_circuit_1_residuals.png" height="300">
</p>

> For more detailed information see `section 10. Fit-quality metrics and residuals`

### 13.5 Parameter bar charts

For each successful sample and circuit:

```text
fits/plots/<sample>/<circuit>/<sample>_<circuit>_R_and_Q.png
```

<p align="center">
  <img src="./resources/Portugal25_R1_1_circuit_1_R_and_Q.png" height="300">
</p>

These show the fitted resistances and capacitance/CPE parameters for one sample.

The left y-axis is used for resistance values, and the right y-axis is used for CPE or capacitance values.

If the circuit contains CPEs, the alpha values can be written above the relevant bars.

### 13.6 Conductivity summary plots

For each conductivity column present in the parameter CSV, the workflow creates one experiment-level conductivity plot.

Example:

```text
fits/plots/<experiment>_<circuit>_Ri_conductivity_all_samples.png
fits/plots/<experiment>_<circuit>_Rbulk_conductivity_all_samples.png
```

<p align="center">
  <img src="./resources/circuit_2_Ri_conductivity_all_samples.png" height="300">
</p>


Each plot contains:

- one bar per sample
- one conductivity type only
- optional error bars
- automatic or user-defined y-axis scaling
- optional normalization
- optional logarithmic scaling
- a colour gradient across samples for style
- configurable sample-label rotation, alignment, and font size through `plotting.conductivities.legend`

The conductivity plots are saved directly in `fits/plots/`, not inside the per-sample folders.

### 13.7 Replotting

If only the plots need to be regenerated, delete `fits/plots/` and rerun the workflow with:

```json
"plot": true
```

The fit CSVs are preserved, so the workflow can replot without refitting. This is useful after changing:

- plot limits
- save preferences
- residual display
- chi-square display
- figure formats
- labels or colours

--- 

## 14. Plot customisation

Most plotting options are controlled from the root-level `global.json`.

### 14.1 Figure saving

Controlled by:

```json
"save_preferences": {
    "transparent": false,
    "png": true,
    "svg": false,
    "dpi": 250
}
```

Possible options:

| Option | Meaning |
|---|---|
| `png` | Save PNG figures |
| `svg` | Save SVG figures |
| `transparent` | Save figures with transparent backgrounds |
| `dpi` | Resolution for raster exports such as PNG. Defaults to `300` |

The `dpi` option is especially useful when balancing speed, file size, and figure quality. For quick testing, values such as `100` or `150` can make plotting faster. For reports or publications, `300` or higher is recommended.

### 14.2 R/Q or R/C parameter plots

Manual y-axis limits for parameter bar charts are controlled by:

```json
"plotting": {
    "R_and_Q": {
        "left_ylim": [1e-1, 1e15],
        "right_ylim": [1e-10, 1e-1]
    }
}
```

| Option | Meaning |
|---|---|
| `plotting.R_and_Q.left_ylim` | Y-axis limits for the left axis, usually resistance |
| `plotting.R_and_Q.right_ylim` | Y-axis limits for the right axis, usually CPE magnitude or capacitance |

Use `[null, null]` to keep automatic Matplotlib scaling.

### 14.3 Conductivity summary plots

Conductivity plot style is controlled by:

```json
"plotting": {
    "conductivities": {
        "left_ylim": [
            null,
            null
        ],
        "normalize": true,
        "cmap_name": "viridis",
        "log_scale": false,
        "legend": {
            "rotation": 0,
            "ha": "center",
            "fontsize": 14
        }
    }
}
```

| Option | Meaning |
|---|---|
| `plotting.conductivities.left_ylim` | Y-axis limits for conductivity plots. Use `[null, null]` for automatic scaling |
| `plotting.conductivities.normalize` | If `true`, conductivity values are normalized, typically by the maximum value in the plotted series |
| `plotting.conductivities.cmap_name` | Matplotlib colormap used for the conductivity bars |
| `plotting.conductivities.log_scale` | If `true`, use a logarithmic y-axis |
| `plotting.conductivities.legend.rotation` | Rotation of the sample labels on the x-axis |
| `plotting.conductivities.legend.ha` | Horizontal alignment of the sample labels |
| `plotting.conductivities.legend.fontsize` | Font size of the sample labels |

Examples of useful Matplotlib colormaps:

```text
viridis
Blues
Greys
magma
cividis
```

To reverse a colormap, append `_r`, for example:

```text
viridis_r
Blues_r
```

For conductivity plots, automatic y-axis scaling is often recommended because conductivities can vary significantly between experiments.

### 14.4 Chi-square display

Chi-square display on fit plots is controlled by:

```json
"validation": {
    "display_chi_square": true
}
```

If disabled, chi-square and reduced chi-square are still calculated and stored in the CSV, but they are not written directly on the figures.

### 14.5 Residual plotting

Residual plot generation is controlled by:

```json
"validation": {
    "plot_residuals": true
}
```

If disabled, residual values can still be calculated internally, but residual figures are not generated.

## 15. Fit warnings and failure handling

The fitter tracks whether the fit was successful.

Useful output columns include:

```text
fit_success
error
chi_square
reduced_chi_square
<parameter>
SD_<parameter>
<parameter>_percent_error
```

A fit may be marked as failed if:

- the complex, non-linear least-squares method does not converge
- fitted parameters are not finite
- covariance or standard errors are unavailable
- one or more fitted parameter percentage errors exceed `validation.error_percent_param_threshold`
- Lin-KK validation fails while `validation.lin_kk.reject_on_fail` is enabled
- the circuit prediction fails
- the fit raises an exception

The fitted-parameter error threshold is designed to catch non-identifiable fits. These are cases where the circuit reproduces the impedance spectrum, but one or more individual parameters are poorly determined.

For example, with:

```json
"error_percent_param_threshold": 200
```

the following value causes the fit to be rejected:

```text
R_bulk_percent_error = 250%
```

The CSV row then contains:

```text
fit_success = False
error = Fit rejected: parameter uncertainty above 200% for R_bulk: 250.0%
```

## 16. Common issues

### Missing parameter CSV

If the terminal prints:

```text
Skipping missing file: <path>
```

then the expected fit CSV does not exist.

Check that:

- the circuit name exists in `metadata.json`
- the circuit was successfully fitted
- the output folder is correct
- workspace/local mode points to the expected location

### CPE values appear as `0.0`

CPE magnitudes can be very small, for example:

```text
1e-8
1e-10
```

Do not round fitted values in `_fit_one`.

Avoid:

```python
row[key] = round(value, 6)
```

because this can turn small CPE values into `0.0`.

Use raw floats and save with scientific notation:

```python
df.to_csv(
    save_path,
    index=False,
    float_format="%.10e",
)
```

### Bar chart has invalid log scale

Log-scale plots cannot contain zero or negative values.

The plotting code should skip values that are:

```text
NaN
inf
0
negative
```

before passing them to Matplotlib.

### Fit rejected because parameter error exceeds the configured threshold

A visually good fit can still be rejected if one or more fitted parameters have a standard error larger than the configured percentage-error threshold.

This means the equivalent circuit can reproduce the spectrum, but the individual parameter is not reliably identifiable.

Check the `error` column in the fit CSV for a message such as:

```text
Fit rejected: parameter uncertainty above 200% for R_bulk: 250.0%
```

Rejected fits are not plotted.

### Empty plot folder

An empty plot folder usually means that the folder was created before the fit was rejected or skipped.

Check the corresponding fit CSV. If `fit_success` is `False`, no Bode, Nyquist, residual, or parameter bar plot is expected.

### Replotting does not update figures

Delete the plot folder:

```bash
rm -rf <processed_experiment>/fits/plots
```

then rerun with:

```json
"plot": true
```

Do not delete the fit CSV files unless you want to refit the data.

## 17. Current development status

The current object-oriented version supports:

- local and workspace data modes
- root-level `global.json` configuration
- metadata-driven experiment configuration
- reusable method files in `data/_methods/`
- preprocessing and pre-plotting
- optional automatic quantile/IQR outlier removal
- optional Lin-KK validation before fitting
- multi-circuit fitting
- one fitted-parameter CSV per circuit
- circuit-specific parameter columns
- residual, chi-square, and reduced chi-square calculation through `fit_metrics.py`
- fitted Bode and Nyquist plotting
- optional residual plotting
- per-sample R/Q or R/C parameter plots
- mapping-controlled conductivity calculation through `compute_conductivity`
- one experiment-level conductivity summary plot per conductivity type
- configurable plot limits and figure saving through `global.json`
- re-plotting without refitting by deleting only `fits/plots/`
- configurable rejection of non-identifiable fits using `validation.error_percent_param_threshold`
- progress bars and terminal output options
- optional computation of experiment-level mean fitted parameters per circuit
- propagated uncertainties for mean parameter rows

This version is intended as a robust research workflow before packaging the project as a reusable Python library.
