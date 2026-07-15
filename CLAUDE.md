# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Methodology

Use Superpowers skills for all development work.

Before writing code:
1. Use brainstorming to clarify the feature.
2. Present the design in small sections for approval.
3. Use writing-plans to create an implementation plan.
4. Use test-driven-development where applicable.
5. Use requesting-code-review before considering the work complete.
6. Use finishing-a-development-branch when the feature is done.

Do not jump directly into implementation unless explicitly instructed.

## Project Overview

**Future1 Pro** is a Streamlit-based forecasting platform with 17+ time series forecasting models. It provides automatic model selection, multi-level data analysis (by product/location/channel), interactive visualizations, and Excel export capabilities. The application includes a multi-layer authentication system and supports both CSV and Excel file uploads.

## Commands

### Setup and Installation
```bash
# Create virtual environment
python -m venv venv
source venv/bin/activate  # On Windows: venv\Scripts\activate

# Install dependencies
pip install -r requirements.txt
```

### Running the Application
```bash
# Run the Streamlit app (default: localhost:8501)
streamlit run app.py
```

### Development Workflow
```bash
# View app logs during development
streamlit run app.py --logger.level=debug

# Run with reload enabled (already default)
streamlit run app.py
```

## Architecture

### Entry Point: `app.py`
- Initializes Streamlit session state (`authenticated`, `username`, `data`)
- Handles authentication via `Authenticator` class
- Manages page navigation (currently only "Best-Fit Forecast" tab is visible)
- Handles file upload in sidebar and loads data via `DataProcessor`
- Renders the active tab page (primarily `multi_model.render()`)

### Core Modules

#### Authentication (`auth/authenticator.py`)
- Implements login/logout flow
- **Security requirement**: Credentials loaded from Streamlit secrets or `FUTURE1_USERS` environment variable (JSON format)
- No hardcoded default credentials allowed; app fails if credentials not configured
- SHA-256 password hashing for stored passwords
- Session management with timeout support

#### Configuration (`config/settings.py`)
- Centralized app settings (name, version, timeouts)
- Forecasting parameters: min data points (3), max forecast periods (120), default periods (24)
- File upload limits (100 MB), allowed extensions (csv, xlsx, xls)
- Model count (17) and visualization settings
- OAuth client IDs for enterprise features (Google, GitHub)

#### Data Processing (`utils/data_processor.py`)
- `load_data()`: Loads CSV/Excel files with automatic format detection
- `load_default_data()`: Fallback to default data if no file uploaded
- `remove_outliers()`: Clips outliers beyond 3 standard deviations (instead of removing)
- `smooth_series()`: Applies moving average smoothing with window parameter
- Handles multiple date formats with fallback parsing

#### Forecasting Models (`models/all_models.py`)
- `ModelFactory` class provides 17 models via `train_and_predict(model_name, train_data, periods)`
- Models grouped by category:
  - **Classical**: Simple/Weighted Average, Moving Averages, Linear Regression
  - **Exponential Smoothing**: Single/Double/Triple, Automated, Adaptive Response Rate, Brown's Linear
  - **Advanced ML**: Auto-ARIMA, SARIMAX, Gradient Boosting, XGBoost variant, Prophet
- Each model returns predictions array; handles errors gracefully

#### Metrics (`utils/metrics.py`)
- Static methods for forecast evaluation: `mape()`, `mae()`, `rmse()`, `mse()`
- MAPE: Mean Absolute Percentage Error (percentage scale, good for mixed magnitudes)
- Used to rank model performance in multi-model comparison

#### Visualizations (`utils/visualizer.py`)
- `plot_forecast()`: Plotly chart showing actual (blue line+markers), fitted line, and forecast
- Properly connects fitted line to forecast with visual continuity
- Used for single and multi-model comparison visualizations

#### Pages (`pages/`)
- **`multi_model.py` (1171 lines, primary page)**: 
  - Core feature: compares all 17 models on uploaded data
  - Creates period-based time series from Year + Week/Month columns (handles both numeric and text month names)
  - Supports group-wise model training (separate forecast per group: e.g., by product/region)
  - Filters negative values; configurable train/test split via Streamlit sliders
  - Generates Excel export with forecasts, actuals, and metrics
  - **Key function**: `create_period_column()` - converts Year + time columns to sequential periods
  
- **`auto_future.py`**: Prophet-specific forecasting interface (currently hidden in tabs)
- **`holt_winters.py`**: Manual Holt-Winters parameter tuning (currently hidden in tabs)
- **`home.py`**: Landing page (currently hidden in tabs)

### Data Flow for Multi-Model Tab

1. User uploads CSV/Excel file via sidebar file uploader
2. `DataProcessor.load_data()` reads file; defaults to sample data if none provided
3. User selects columns via Streamlit dropdowns:
   - Date/Year column (identifies time periods)
   - Value column (forecast target)
   - Optional: Group columns (e.g., Product, Region)
4. Data preprocessing:
   - `create_period_column()` converts Year + Month into sequential periods
   - Filters out rows with negative/zero values
5. Train/test split (configurable slider, default ~80/20)
6. For each group (or global if no groups):
   - Run all 17 models via `ModelFactory`
   - Calculate accuracy metrics (MAPE, MAE, RMSE) on test set
   - Train best model on full data and generate forecast
7. Visualizations:
   - Best model forecast plot
   - Model comparison bar chart (by MAPE score)
8. Excel export:
   - Summary sheet with metrics
   - Forecast sheet with dates, actuals, fitted, forecast values

### Key Design Patterns

- **Session State**: Streamlit `st.session_state` holds authentication status, username, and uploaded data across reruns
- **Streamlit Caching**: Use `@st.cache_data` for expensive operations (model training) to avoid recomputation on every UI interaction
- **Period-Based Indexing**: Multi-model page converts calendar dates (Year/Month) to sequential periods to handle irregular sampling and multi-year data
- **Group-Wise Isolation**: Each product/region/channel is forecasted independently to avoid cross-group averaging effects
- **Graceful Error Handling**: Models catch exceptions and fallback; MAPE handles zero-division
- **Tab Visibility**: Currently only "Best-Fit Forecast" (multi_model) is visible; other tabs commented out in app.py

### Environment and Deployment

- **Local Development**: `streamlit run app.py` (auto-reload on code changes)
- **Credentials**: Set `FUTURE1_USERS` env var (JSON: `{"username": {"password": "sha256_hash"}}`) or use Streamlit secrets
- **Python Version**: 3.8+ (requirements.txt pins major versions)
- **Dependencies**: pandas, numpy, statsmodels, prophet, scikit-learn, plotly, openpyxl, streamlit
- **Dev Container**: `.devcontainer` folder included for containerized development

## Common Development Tasks

### Adding a New Forecasting Model
1. Open `models/all_models.py`
2. Add method `_your_model_name(self, train, periods)` to `ModelFactory`
3. Return predictions array with length = `periods`
4. Register in `__init__` dict: `'N. Model Name': self._your_model_name`
5. Update `AVAILABLE_MODELS` count in `config/settings.py` if needed

### Debugging Multi-Model Forecast Issues
- Check `create_period_column()` logic if forecast is misaligned with data
- Verify group columns are properly selected and data has no null group values
- Look for negative/zero values being filtered out prematurely
- Check train/test split slider to ensure test set has sufficient data

### Modifying Export Format
- Excel generation logic is in `multi_model.py` (search for `xlsxwriter`)
- Columns: typically Date, Actual, Fitted, Forecast, Metric values
- Summary sheet lists all models and their MAPE scores

### Credential Management (Security)
- **Never commit credentials**; always use `FUTURE1_USERS` env var or Streamlit secrets
- For local testing, set env var: `export FUTURE1_USERS='{"admin": {"password": "8c6976e5b5410415bde908bd4dee15dfb167a9c873fc4bb8a81f6f2ab448a918"}}'` (admin/admin)
- For Streamlit Cloud: store credentials in app settings → Secrets

## Key Files by Purpose

| Purpose | File | Lines |
|---------|------|-------|
| Main app entry | app.py | 112 |
| Multi-model comparison (primary feature) | pages/multi_model.py | 1171 |
| All 17 forecasting models | models/all_models.py | 316 |
| Data loading and preprocessing | utils/data_processor.py | 219 |
| Accuracy metrics (MAPE, MAE, RMSE) | utils/metrics.py | 50 |
| Plotly visualizations | utils/visualizer.py | 168 |
| Login/logout and session management | auth/authenticator.py | 100+ |
| App configuration and constants | config/settings.py | 47 |

## Recent Work and Context

- **Latest commit (5047ad8)**: Added Dev Container Folder
- **Recent features**: Hide requested tabs (only multi-model visible), modify forecast periods, enhance download with Excel summaries, support text-based months
- **Current focus**: Multi-model page and data preprocessing robustness

