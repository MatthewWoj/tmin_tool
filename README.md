# Minimum Temperature Forecasting Tool

A Python command-line tool implementing the Forecaster's Reference Book method for overnight minimum temperature calculation.

## Overview

This tool calculates overnight minimum temperature using:

<img width="314" height="37" alt="image" src="https://github.com/user-attachments/assets/7de85b87-7605-4db7-9125-22790d3a84a4" />

Where:

<img width="333" height="80" alt="image" src="https://github.com/user-attachments/assets/cf0177c4-572b-472c-82f0-1f3687ee7b8d" />

And K is looked up from a table based on wind speed (knots) and cloud cover (oktas).

The tool is designed for comparing the historic Reference Book method against modern models and observed temperatures.

## Requirements

- Python 3.8 or later
- No external dependencies for running the tool
- pytest (optional, for running tests)

## Quick Start

Run on the provided trial data:
```bash
python tmin_tool.py --input_csv example_input.csv --output_csv outputs.csv
```

View results in terminal without saving:
```bash
python tmin_tool.py --input_csv example_input.csv
```

## Input Format

Required CSV columns:
- `date`
- `location`
- `midday_temperature_c`
- `midday_dew_point_c`
- `wind_speed_knots`
- `cloud_cover_oktas`

Optional columns for comparison:
- `latest_model_tmin_c`
- `observed_tmin_c`

Example:
```csv
date,location,midday_temperature_c,midday_dew_point_c,wind_speed_knots,cloud_cover_oktas
1,A,22.4,10.9,14.56,3.9
1,B,18.6,12.65,3.4,6.0
```

## Output Format

The output CSV includes all input columns plus:
- `wind_band_label` - which wind band was used
- `cloud_band_label` - which cloud band was used
- `k_value` - K adjustment value from lookup table
- `tmin_c_raw` - calculated Tmin as float
- `tmin_c_reported` - rounded Tmin as integer
- `delta_vs_latest_model_c` - difference from model (if provided)
- `error_vs_observed_c` - error vs observation (if provided)
- `flags` - any validation warnings or issues
- `method_version` - config version used

## Usage Examples

### Basic forecast calculation
```bash
python tmin_tool.py --input_csv example_input.csv --output_csv outputs.csv
```

### With comparison data

When your CSV includes `latest_model_tmin_c` and/or `observed_tmin_c` columns:
```bash
python tmin_tool.py --input_csv example_input_with_comparison.csv --log-level INFO
```

This will log summary statistics:
```
INFO tmin_tool Legacy vs latest model: n=4 bias=-0.500 MAE=0.800 RMSE=1.000
INFO tmin_tool Legacy vs observed: n=4 bias=0.200 MAE=1.100 RMSE=1.300
```

### Using a custom configuration

To extend wind bands or modify K values without changing code:
```bash
python tmin_tool.py --input_csv data.csv --config_json custom_config.json
```

### Debug mode

For detailed logging:
```bash
python tmin_tool.py --input_csv data.csv --log-level DEBUG
```

## Custom Configuration

You can supply a JSON config file to modify wind/cloud bands and K lookup values.

Example config structure:
```json
{
  "version": "custom_v1",
  "wind_speed_bands_knots": [
    {
      "lower": 0,
      "upper": 12,
      "include_lower": true,
      "include_upper": true,
      "label": "0-12 kn"
    },
    {
      "lower": 12,
      "upper": 25,
      "include_lower": false,
      "include_upper": true,
      "label": "13-25 kn"
    }
  ],
  "cloud_cover_bands_oktas": [
    {
      "lower": 0,
      "upper": 2,
      "include_lower": true,
      "include_upper": false,
      "label": "0-2 oktas"
    }
  ],
  "k_values": [
    [-2.2, -1.7],
    [-1.1, 0.0]
  ]
}
```

The tool validates:
- Band dimensions match K table dimensions
- Bands are sorted and non-overlapping
- Boundary inclusivity rules prevent overlaps

## Running Tests

Install pytest:
```bash
pip install pytest
```

Run tests:
```bash
pytest -v test_tmin_tool.py
```

Expected output:
```
test_reference_book_example_rounds_to_10 PASSED
test_wind_band_boundary_no_overlap PASSED
test_unknown_k_cell_is_flagged PASSED
test_okta_9_is_treated_as_sky_obscured PASSED
test_invalid_config_shape_raises PASSED
test_invalid_config_overlap_raises PASSED
test_config_json_load PASSED
test_comparison_fields_compute_deltas PASSED
```

## Design Decisions

### Rounding Convention

The tool uses "round half away from zero" rather than Python's default banker's rounding. This matches the reference example (9.928 rounds to 10).

### Boundary Handling

Band boundaries are explicit with `include_lower` and `include_upper` flags. For example, 12.0 knots falls in the first band [0, 12], while 12.0001 falls in the second band (12, 25].

This prevents ambiguity and overlap bugs.

### Unknown K Values

When K is listed as "Unknown" in the lookup table (high wind + high cloud), the tool:
- Does not interpolate or guess
- Flags the case with `K_UNKNOWN_TABLE_CELL`
- Returns no forecast for that sample

### Okta 9 (Sky Obscured)

Cloud cover of 9 oktas represents "sky obscured" (fog, heavy precipitation) rather than cloud amount. The tool flags this specially and does not compute a forecast, as the K adjustment method assumes visible cloud cover.

### Error Handling

The tool uses flags rather than exceptions for validation issues. This allows processing the entire dataset and provides an audit trail of problems.

Common flags:
- `CLOUD_OUT_OF_RANGE_0_TO_8`
- `WIND_NEGATIVE`
- `DEW_POINT_GT_TEMPERATURE_CHECK_UNITS`
- `K_UNKNOWN_TABLE_CELL`
- `SKY_OBSCURED_OKTA9`
- `BINNING_ERROR:...`

## Files

- `tmin_tool.py` - Main tool (library + CLI)
- `test_tmin_tool.py` - Test suite
- `example_input.csv` - Trial data from exercise
- `example_input_with_comparison.csv` - With comparison columns
- `example_config_extended.json` - Extended config example
- `README.md` - This file

## Known Limitations

### Data Preparation

The tool does not join model and observation data. In production, you would:
- Standardize station IDs and timestamps
- Join forecast and observation datasets upstream
- Handle time zone conversions

### Overnight Definition

"Overnight minimum" depends on the operational definition (time window, local vs UTC). This should be agreed and standardized in the data pipeline.

### Scalability

For large datasets, consider:
- Batch processing with progress indicators
- Vectorized operations (numpy/pandas)
- Stratified verification (by season, location type)
- Additional diagnostics beyond MAE/RMSE

### Comparison Metrics

Current metrics are global averages. Production systems typically need:
- Stratification by regime or season
- Confidence intervals
- Skill scores relative to climatology
- Case studies of large errors

## Contributing

When modifying the K lookup table or bands:
1. Create a new config JSON file
2. Test with `--config_json your_config.json`
3. Validate dimensions match
4. Check boundary overlaps are prevented
5. Document the scientific basis for changes

## License

This tool was developed as an interview exercise demonstrating software engineering practices for meteorological applications.
