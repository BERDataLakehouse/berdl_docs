# Jupyter AI CBorg Extension

**Repository:** [cdm-jupyter-ai-cborg](https://github.com/BERDataLakehouse/cdm-jupyter-ai-cborg)

The `cdm-jupyter-ai-cborg` package enables integration between JupyterLab's built-in AI assistant framework ([Jupyter AI](https://jupyter-ai.readthedocs.io/)) and the [CBorg API provider](https://cborg.lbl.gov/) services available at LBL. 

## Overview

This module is constructed as a standard Jupyter AI Module using their Cookiecutter. Its core responsibility is registering the Custom Model Provider (`cborg.py`) into the Jupyter AI system.

By integrating CBorg as a Custom Model Provider, BERDL users can leverage powerful institutional AI models seamlessly via standard Jupyter AI slash commands within their notebooks and chat interfaces, while adhering to locally managed LLM usage tracking or governance.

## Requirements

- Python 3.8 - 3.12
- JupyterLab 4
- An active `jupyter-ai` installation

## Development & Usage

The extension is installed directly as a Python package. 

```bash
# Installation
pip install -e "."

# Uninstallation
pip uninstall jupyter_ai_cborg
```

Once installed into an environment running `jupyter-ai`, the Custom Model Provider allows Jupyter AI interactions to route through CBorg's endpoints automatically.
