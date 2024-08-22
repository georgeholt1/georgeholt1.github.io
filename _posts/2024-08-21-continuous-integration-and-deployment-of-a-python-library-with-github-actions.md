---
layout: single
title: Continuous integration and deployment of a Python library with GitHub Actions
categories: [DevOps, CI/CD, Python, GitHub]
toc: true
toc_sticky: true
date: 2024-08-21
---

Continuous integration and continuous deployment ([CI/CD](https://en.wikipedia.org/wiki/CI/CD)) is an integral part of the [DevOps](https://www.redhat.com/en/topics/devops) process. Indeed, the first principle of the [Manifesto for Agile Software Development](https://agilemanifesto.org/) states (emphasis added):

> Our highest priority is to satisfy the customer through early and **continuous delivery** of valuable software.

Recently, while developing a static site using [Sphinx](https://www.sphinx-doc.org/en/master/), I found myself in need of a command line tool to crawl through the site and check for broken links. Having done a search, I couldn't find a tool that met my requirements (simple, lightweight, free). So, I quickly built a Python package and application, [PyLich](https://github.com/georgeholt1/pylich), to fill this gap and used it as an excuse to set up CI/CD with [GitHub Actions](https://docs.github.com/en/actions).

This post outlines the steps I took to set up CI/CD for PyLich.

## PyLich Overview

PyLich is a Python package that provides a command line interface to crawl through a site and check the status of all links. The package is hosted on the Python Package Index ([PyPI](https://pypi.org/)) and is installable with [pip](https://pip.pypa.io/en/stable/). Usage is as simple as providing a URL to a sitemap:
    
```bash
pylich https://example.com/sitemap.xml
```

The application exits with a status code of 0 if all links are valid and 1 if any links are broken, along with a report on the broken links. PyLich is open source and freely available on [GitHub](https://github.com/georgeholt1/pylich).

I used [Poetry](https://python-poetry.org/) for dependency management and to provide packaging and publishing capabilities. Unit tests are written using [pytest](https://docs.pytest.org/en/stable/), and formatting and linting are enforced with [black](https://github.com/psf/black), [isort](https://pycqa.github.io/isort/), and [flake8](https://flake8.pycqa.org/en/latest/).

## Project structure

Poetry handles the package structure, but there are a couple of additional files and directories that are required to configure formatting, CI/CD, the repository, etc. The project structure is as follows:

```plaintext
pylich
├── .flake8
├── .github
|   ├── actions
|   |   ci-setup
|   |   ├── action.yml
|   |   └── config.yml
|   ├── workflows
|   |   ├── build_test.yml
|   |   ├── formatting_and_linting.yml
|   |   ├── publish_to_pypi.yml
|   |   └── unit_tests.yml
├── .gitignore
├── .pre-commit-config.yaml
├── LICENSE
├── README.md
├── pylich
│   ├── __init__.py
│   ├── checker.py
│   └── cli.py
├── pyproject.toml
└── tests
    ├── __init__.py
    └── test_link_checker.py
```

The package itself is in the `pylich` directory, with the main entry point in `cli.py` and the core logic in `checker.py`. Dependencies, package metadata and build information are in the `pyproject.toml` file, which is largely managed by Poetry. The unit tests are in the `tests` directory, and the GitHub Actions workflows and custom actions are in the `.github` directory.

## GitHub Actions overview

GitHub describes [Actions](https://docs.github.com/en/actions) as a way to "automate, customize, and execute your software development workflows right in your repository", which means it can be used for CI/CD and other automation tasks.

In a nutshell<sup>*</sup>, a user defines a GitHub Actions **workflow** in a YAML file, which specifies the steps to run when an **event** occurs. The workflow contains one or more **jobs**, which can run in parallel or sequentially, and each job contains one or more **steps**. Steps can be a user-defined script or an **action**, which is a reusable unit of code. Each job runs on its own **runner**, which is a virtual machine or container that can be provided by GitHub or self-hosted.

<sup><sup>*</sup>Paraphrased from [Understanding GitHub Actions](https://docs.github.com/en/actions/about-github-actions/understanding-github-actions).</sup>

## Setting up GitHub Actions

The GitHub actions for PyLich are broadly split into two categories: workflows that run on every pull request (CI) and a workflow that runs on every release (CD).

The CI workflows are triggered on every pull request to the `main` branch and consist of the following:

- Formatting and linting checks with black, isort, and flake8.
- Unit tests with pytest.
- A package build-for-deployment test.

The unit tests are run with multiple Python versions to ensure compatibility. 

The CD workflow is triggered on every [release](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases). Specifically, the workflow is triggered when a tag that matches the pattern `v*.*.*` (e.g., `v1.0.0`) is created, which occurs when a release is published. The CD workflow builds and publishes the package to PyPI.

Now, into more detail.

### Common setup

Each workflow starts with a common setup step that consists of setting up Python, installing Poetry, and installing PyLich and its dependencies. To avoid code repetition, I created a custom action, `ci-setup`, that encapsulates this setup. The action is defined in the `.github/actions/ci-setup/action.yml` file:

```yaml
name: CI setup
description: 'Set up the CI environment for the repository'

inputs:
  python-version:
    description: 'The version of Python to use'
    required: true
    default: '3.x'

runs:
  using: composite
  steps:
    - name: Set up Python
      id: setup-python
      uses: actions/setup-python@v4
      with:
        {% raw %}python-version: ${{ inputs.python-version }}{% endraw %}

    - name: Read Poetry Version
      id: read-poetry-version
      run: |
        poetry_version=$(cat .github/actions/ci-setup/config.yml | grep poetry_version | awk '{print $2}')
        echo "poetry_version=$poetry_version" >> $GITHUB_ENV
      shell: bash

    - name: Cache poetry
      id: cache-poetry
      uses: actions/cache@v4
      with:
        path: ~/.local
        key: {% raw %}${{ runner.os }}-poetry-${{ env.poetry_version }}{% endraw %}

    - name: Install poetry
      id: install-poetry
      if: steps.cache-poetry.outputs.cache-hit != 'true'
      uses: snok/install-poetry@v1
      with:
        version: {% raw %}${{ env.poetry_version }}{% endraw %}
        virtualenvs-create: true
        virtualenvs-in-project: true

    - name: Configure poetry
      id: configure-poetry
      if: steps.cache-poetry.outputs.cache-hit
      run: poetry config virtualenvs.in-project true
      shell: bash

    - name: Set python version
      run: poetry env use python
      shell: bash

    - name: Cache poetry dependencies
      id: cache-poetry-dependencies
      uses: actions/cache@v4
      with:
        path: |
          .venv
        key: {% raw %}${{ runner.os }}-poetry-dependencies-${{ hashFiles('**/pyproject.toml') }}{% endraw %}

    - name: Install dependencies
      if: steps.cache-poetry-dependencies.outputs.cache-hit != 'true'
      run: poetry install
      shell: bash
```

The action takes as input the Python version to use and uses the standard [setup-python](https://github.com/actions/setup-python) action to set up the specified Python version.

The Poetry documentation [strongly recommends](https://python-poetry.org/docs/#ci-recommendations) pinning the Poetry version to avoid breaking changes. Since the poetry version is used in multiple places in the `ci-setup` action, and to maintain a [single source of truth](https://en.wikipedia.org/wiki/Single_source_of_truth), I created a `config.yml` file in the same directory:

```yaml
poetry_version: 1.8.2
```

The `read-poetry-version` step reads the Poetry version from the `config.yml` file and sets it as an environment variable.

The `cache-poetry` step checks for the existence of a cached version of Poetry and loads it if there's a cache hit. Per the [GitHub Actions caching documentation](https://docs.github.com/en/actions/writing-workflows/choosing-what-your-workflow-does/caching-dependencies-to-speed-up-workflows), the `cache` action creates a cache of the directory specified in the `path` variable on a cache miss at the end of the job. The key is a combination of the runner OS and Poetry version, so a new cache is created if the runner OS changes or the Poetry version is updated.

The `install-poetry` step installs the version of Poetry specified in the config file using the [snok/install-poetry](https://github.com/snok/install-poetry) action if there's no cache hit from the previous step.

The `configure-poetry` step sets the `virtualenvs.in-project` configuration option to `true`. This option tells Poetry to create virtual environments in the project directory, which is used later to cache the virtual environment. The step executes only if there's a cache hit from the `cache-poetry` step, which means that the `install-poetry` step was skipped and poetry was not configured to create virtual environments in the project directory by the `snok/install-poetry` action.

The `set-python-version` step sets the Python version to use with Poetry. Without this step, Poetry uses the system Python version, which has no guarantee of matching the version specified in the `config.yml` file.

Similar to the `cache-poetry` step, the `cache-poetry-dependencies` step loads a cached version of the virtual environment (which includes PyLich and its dependencies) if the cache exists. The path is set to `.venv`, which is the location of the virtual environment created by Poetry when the `snok/install-poetry` action is used with the `virtualenvs-create` and `virtualenvs-in-project` flags set to `true`. The key contains a hash of the `pyproject.toml` file, which is used to determine if the cache is up-to-date. If the `pyproject.toml` file changes (for example, if a new dependency is added or the project version number is updated), the cache is invalidated and recreated.

Finally, the `install-dependencies` steps installs PyLich and its dependencies into a virtual environment using Poetry if there's no cache hit from the previous step.

### CI workflows

The CI workflows are defined in the `build_test.yml`, `formatting_and_linting.yml`, and `unit_tests.yml` files in the `.github/workflows` directory.

#### Formatting and linting

The formatting and linting job is shown below:

```yaml
name: Formatting and Linting

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  formatting:
    name: Formatting
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Setup
      uses: './.github/actions/ci-setup'
      id: setup
      with:
        python-version: '3.x'

    - name: Run black
      run: poetry run black --check .

    - name: Run isort
      run: poetry run isort --check-only .

  linting:
    name: Linting
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup
      uses: './.github/actions/ci-setup'
      id: setup
      with:
        python-version: '3.x'

    - name: Run flake8
      run: poetry run flake8 .
```

The workflow is triggered on every push to the `main` branch and every pull request to the `main` branch. The workflow consists of two jobs: `formatting` and `linting`. Both jobs run on an Ubuntu runner and start by using the `checkout` action to get the repository, and then use the `ci-setup` action to set up the Python environment and install PyLich and its dependencies.

The `formatting` job runs the `black` and `isort` commands to check if the code is correctly formatted. The `linting` job runs the `flake8` command to check for linting errors. Each of these commands exits with a status code of 0 if there are no linting errors or formatting issues and 1 if there are.

The `formatting` and `linting` jobs run in parallel and are independent of each other. If either job fails, the entire workflow fails.

#### Unit tests

The `unit_tests.yml` file is similar to the `formatting_and_linting.yml` file but runs the `pytest` command to run the unit tests in the `tests` directory:

```yaml
name: Unit tests

on:
  push:
    branches:
      - main
  pull_request:
    branches:
      - main

jobs:
  unit-tests:
    name: Unit tests
    runs-on: ubuntu-latest

    strategy:
      matrix:
        python-version: ["3.9", "3.10", "3.11", "3.12"]

    steps:
    - name: Checkout code
      uses: actions/checkout@v3
    
    - name: Setup
      uses: './.github/actions/ci-setup'
      id: setup
      with:
        python-version: {% raw %}${{ matrix.python-version }}{% endraw %}

    - name: Run unit tests
      run: poetry run pytest
```

The key difference is the `strategy` section, which runs the `unit-tests` job with multiple Python versions. The `matrix` section specifies the Python versions to use, and the `python-version` variable is used in the `ci-setup` action to set up the correct Python version. The job runs four times, once for each Python version specified in the `matrix` section, which occurs in parallel. If any of the jobs fail, the entire workflow fails. This allows for specifying Python version compatability in the `pyproject.toml` file:

```toml
[tool.poetry.dependencies]
python = "^3.9"
```

#### Build test

The `build_test.yml` file is similar again to `formatting_and_linting.yml` but contains only a single job that runs the `poetry build` command to check that the package builds correctly:

```yaml
...
    steps:
    ...
    - name: Build
      run: poetry build
```

### CD workflow

The CD workflow consists of a single job that is triggered on every [release](https://docs.github.com/en/repositories/releasing-projects-on-github/about-releases). PyLich follows the [Semantic Versioning](https://semver.org/) convention, so the workflow is triggered when a tag that matches the pattern `v*.*.*` is created. The workflow is defined in the `publish_to_pypi.yml` file:

```yaml
name: Publish to PyPI

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  build-and-publish:
    runs-on: ubuntu-latest

    steps:
    - name: Checkout code
      uses: actions/checkout@v3

    - name: Setup
      uses: './.github/actions/ci-setup'
      id: setup
      with:
        python-version: '3.x'

    - name: Build
      run: poetry build

    - name: Publish package to PyPI
      run: poetry publish --username __token__ --password {% raw %}${{ secrets.PYPI_API_TOKEN }}{% endraw %}
```

Like in the CI workflows, the CD workflow job starts by checking out the code and setting up the Python environment. The `build` step runs the `poetry build` command to build the package, and the `Publish package to PyPI` step runs the `poetry publish` command to publish the package to PyPI. The `PYPI_API_TOKEN` [secret](https://docs.github.com/en/actions/security-for-github-actions/security-guides/using-secrets-in-github-actions) is used to authenticate with PyPI.

One small caveat of the CD workflow is that, in order to generate a [PyPI API token](https://pypi.org/help/#apitoken) that is limited in scope to a single PyPI repository, the repository must already exist. This means that the first release must be done manually, but subsequent releases can be automated.

## Conclusion

The PyLich CI/CD is hopefully a sensible reference for setting up CI/CD for a Python package with GitHub Actions.

Also, check out PyLich if you need a simple tool to spot broken links in a site that you're buliding. I integrated it as part of the CI for my static documentation sites and found it to be useful.

> **_NOTE:_**  Updated on 2024-08-22 to include a fix to the dependency caching step.