# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Setup Guide

**For new users deploying to Google Cloud:**
The `SETUP_GUIDE.md` file contains complete step-by-step instructions for configuring Google Cloud Platform to deploy this Streamlit application. The guide is designed for Google Cloud beginners and includes:

- Google Cloud project creation and configuration
- Required API enablement (Cloud Run, Artifact Registry, Cloud Build, IAM)
- Workload Identity setup for secure GitHub Actions authentication
- Artifact Registry repository creation for Docker image storage
- GitHub Secrets configuration
- Detailed explanations of what each step does and why it's needed

Users should follow `SETUP_GUIDE.md` first to configure their Google Cloud environment before attempting deployment. The guide uses environment variables to make commands easy to copy-paste and includes troubleshooting information.

## Development Commands

**Run the application locally:**
```bash
uv run streamlit run app.py
```

**Install dependencies:**
```bash
uv sync
```

**Code quality and linting:**
```bash
uv run ruff check
uv run ruff format
uv run mypy .
```

**Run tests:**
```bash
uv run pytest
```

## Project Architecture

This is a Streamlit web application designed for deployment to Google Cloud Run. The application structure follows a modular approach:

- **`app.py`**: Main entry point containing the Streamlit UI logic
- **`src/core/config.py`**: Application configuration using Pydantic Settings with support for environment variables
- **`src/utils/logger.py`**: Centralized logging setup with Google Cloud Logging integration
- **`Dockerfile`**: Multi-stage Docker build using Python 3.12 and uv for dependency management

## Key Components

**Configuration Management:**
- Uses Pydantic Settings for type-safe configuration
- Environment variables: `PROJECT_ID`, `CLOUD_LOGGING`
- Settings cached using `@lru_cache` decorator

**Logging:**
- Supports both local (StreamHandler) and Google Cloud Logging
- Conditional cloud logging based on `CLOUD_LOGGING` setting
- Structured JSON logging for application events

**Deployment:**
- GitHub Actions workflow for automated Cloud Run deployment
- Uses Workload Identity for secure GCP authentication (setup instructions in `SETUP_GUIDE.md`)
- Multi-stage Docker build optimized for production
- Complete deployment setup instructions available in `SETUP_GUIDE.md` for new users

## Dependency Management

This project uses `uv` for dependency management instead of pip. The `pyproject.toml` defines:
- Runtime dependencies (streamlit, google-cloud-logging, pydantic-settings)
- Development dependencies (ruff, mypy, pytest, jupyter)
- Code formatting with line length 120 and Python 3.12 target