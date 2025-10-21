# Python Package Publishing Notes

Goal: publish the `lekinpy` library to PyPI (optionally TestPyPI first) using modern packaging (pyproject) and Twine with API tokens.

## Prerequisites
- Python 3.9+ with `pip`
- Install tooling:
  - `python3 -m pip install --upgrade build twine packaging`
- Recommended: enable 2FA on PyPI and use an API token (username/password uploads are disabled).

## Prepare The Package
- `pyproject.toml` key settings (example):
  - `project.name = "lekinpy"`
  - `project.version = "0.1.0"`
  - `project.license = "MIT"` (use SPDX string)
  - Keep classifiers but omit the license classifier (deprecated)
  - `tool.setuptools.packages.find.include = ["lekinpy*"]`
  - `tool.setuptools.packages.find.exclude = ["examples*", "Data*", "Data2*", "tests*"]`
  - No package data section if you don’t ship data files
- `MANIFEST.in` (for sdist):
  - `include README.md`
  - `include LICENSE`
  - `prune examples`
  - `prune Data`
  - `prune Data2`

## Build Artifacts
- Clean build output (optional):
  - `rm -rf build dist *.egg-info`
- Build sdist and wheel:
  - `python3 -m build`  (or `python3 -m build --no-isolation` if needed)
- Verify metadata:
  - `python3 -m twine check dist/*`

## Check Name Availability (optional)
- Expect `404` for available name:
  - `curl -s -o /dev/null -w "%{http_code}" https://pypi.org/pypi/lekinpy/json`
  - `curl -s -o /dev/null -w "%{http_code}" https://test.pypi.org/pypi/lekinpy/json`

## TestPyPI (Recommended Dry Run)
1) Create a TestPyPI account (separate from PyPI) and an API token.
2) Upload:
   - `export TWINE_USERNAME=__token__`
   - `export TWINE_PASSWORD='pypi-TEST-XXXXXXXX'`
   - `python3 -m twine upload --repository testpypi dist/*`
3) Verify install:
   - `python3 -m pip install -i https://test.pypi.org/simple/ lekinpy==0.1.0`

## PyPI Upload
1) Create a PyPI API token (Account settings → API tokens).
2) Use env vars (avoids secrets in command history):
   - `export TWINE_USERNAME=__token__`
   - `export TWINE_PASSWORD='pypi-XXXXXXXXXXXXXXXX'`
   - `python3 -m twine upload dist/*`
   - Note: Username/password authentication is no longer supported by PyPI.
3) Alternative: configure `~/.pypirc` and run `python3 -m twine upload dist/*` without env vars:
```
[pypi]
  repository = https://upload.pypi.org/legacy/
  username = __token__
  password = pypi-XXXXXXXXXXXXXXXX

[testpypi]
  repository = https://test.pypi.org/legacy/
  username = __token__
  password = pypi-TEST-XXXXXXXX
```

## Versioning & Re-uploads
- PyPI/TestPyPI do not allow re-uploading the same file/version.
- If you need to republish, bump `project.version` (PEP 440), rebuild, and upload again.

## Troubleshooting
- 403 on PyPI: username/password uploads are blocked → use API token or Trusted Publishers.
- 403 on TestPyPI: ensure you have a TestPyPI account/token (separate from PyPI).
- License warnings/errors: use `project.license = "MIT"` (SPDX) and remove the license classifier.
- Missing `packaging.licenses`: upgrade `packaging` (e.g., `python3 -m pip install -U packaging`).
- Extra files in sdist: use `prune` in `MANIFEST.in` and restrict `packages.find` includes/excludes.

## Quick Command Summary
- Install tools: `python3 -m pip install -U build twine packaging`
- Build: `python3 -m build`
- Check: `python3 -m twine check dist/*`
- Upload (TestPyPI):
  - `export TWINE_USERNAME=__token__`
  - `export TWINE_PASSWORD='pypi-TEST-XXXXXXXX'`
  - `python3 -m twine upload --repository testpypi dist/*`
- Upload (PyPI):
  - `export TWINE_USERNAME=__token__`
  - `export TWINE_PASSWORD='pypi-XXXXXXXXXXXXXXXX'`
  - `python3 -m twine upload dist/*`


# Python Library Publishing — Personal General Notes

These are my **general, reusable notes** for creating and publishing any Python library to PyPI (or TestPyPI). The goal is to keep this practical, in **plain English**, with a **short checklist** of files, what they do, and a **step‑by‑step** command flow I can reuse for any project.

---

## 1) Folder Layout (recommended)

```
my-lib/
│
├── src/
│   └── my_lib/                # the importable package (use underscores)
│       ├── __init__.py        # expose version, main API
│       └── ...                # modules
│
├── tests/                     # unit tests (pytest)
├── README.md                  # shown on PyPI page
├── LICENSE                    # your license (e.g., MIT)
├── pyproject.toml             # build + project metadata (PEP 621)
├── MANIFEST.in                # optional sdist file rules
├── .gitignore                 # ignore build artifacts, venvs, etc.
├── CHANGELOG.md               # optional, version history
└── CITATION.cff               # optional, how to cite the work
```

> **Why `src/` layout?** It prevents accidental imports from the project root and mirrors what happens after install. It’s the most robust layout for packages.

---

## 2) What each file is for (easy language)

- **`pyproject.toml`** → The **single source** of build and metadata (name, version, description, dependencies). Replaces old `setup.py`.
- **`README.md`** → Shown on your PyPI page; explain what the library does and how to install/use it.
- **`LICENSE`** → Tells others what they’re allowed to do with your code (e.g., MIT, Apache-2.0). Use an SPDX name.
- **`src/my_lib/__init__.py`** → Makes the package importable and is a good home for `__version__` and top‑level API exports.
- **`MANIFEST.in`** (optional) → Extra rules for what to include/exclude in **source** distributions (sdist). Wheel content is governed by package discovery, not this file.
- **`tests/`** → Keeps automated checks; helps prevent regressions.
- **`.gitignore`** → Prevents build junk and virtualenvs from being committed.
- **`CHANGELOG.md`** (optional) → Human‑readable release notes.
- **`CITATION.cff`** (optional) → Tells researchers how to cite your library.

---

## 3) Minimal `pyproject.toml` (copy/paste and edit)

```toml
[build-system]
requires = ["setuptools>=61.0", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "my-lib"                     # final PyPI name (use hyphens allowed; import package uses underscores)
version = "0.1.0"                   # follow SemVer: MAJOR.MINOR.PATCH
description = "Short one-line description of the library"
readme = "README.md"
requires-python = ">=3.9"
license = { text = "MIT" }          # SPDX identifier or license file
keywords = ["utilities", "example"]
authors = [{ name = "Your Name", email = "you@example.com" }]
dependencies = [
  # "pandas>=2.0",
]

[project.urls]
Homepage = "https://github.com/yourname/my-lib"
Documentation = "https://yourname.github.io/my-lib"
Source = "https://github.com/yourname/my-lib"
Issues = "https://github.com/yourname/my-lib/issues"

[tool.setuptools.packages.find]
where = ["src"]
include = ["my_lib*"]
exclude = ["tests*"]
```

> If you ship non‑Python files (data, templates), you may also need `package_data` or `include-package-data = true` in setuptools config and ensure those files are inside your package directory.

---

## 4) `__init__.py` basics

```python
# src/my_lib/__init__.py
__all__ = [
    # expose your public API here, e.g., "main", "MyClass"
]

__version__ = "0.1.0"
```

---

## 5) Optional `MANIFEST.in` (for sdist)

Use this if you need to include extra top‑level files in the **sdist** (source tarball). Wheels don’t use this file.

```ini
include README.md
include LICENSE
prune tests
```

Examples:
- `include` keeps files.
- `exclude` removes specific paths.
- `prune` removes whole directories.

---

## 6) Common commands — End‑to‑End (Step by Step)

### Step 0 — Prerequisites
```
python3 -m pip install -U pip build twine packaging
```

### Step 1 — Pick a package name & version
- Check name availability (404 means available):
```
curl -s -o /dev/null -w "%{http_code}" https://pypi.org/pypi/my-lib/json
```
- Choose a version like `0.1.0` (SemVer). **You cannot re‑upload the same version** to PyPI.

### Step 2 — Project skeleton
```
mkdir -p my-lib/src/my_lib my-lib/tests
cd my-lib
printf "# My Lib\n\nShort description.\n" > README.md
printf "MIT License..." > LICENSE   # or copy a proper license text
```
Create `pyproject.toml` and `src/my_lib/__init__.py` as above.

### Step 3 — Build (clean + build)
```
rm -rf build dist *.egg-info
python3 -m build
```
This creates `dist/*.tar.gz` (sdist) and `dist/*.whl` (wheel).

### Step 4 — Check artifacts locally
```
python3 -m twine check dist/*
```
Fix any warnings/errors before upload.

### Step 5 — TestPyPI dry‑run (recommended)
1) Create a **TestPyPI** account and an **API token**.
2) Upload with environment variables (safer than typing tokens inline):
```
export TWINE_USERNAME=__token__
export TWINE_PASSWORD='pypi-TEST-XXXXXXXXXXXXXXXXXXXXXXXX'
python3 -m twine upload --repository testpypi dist/*
```
3) Try installing from TestPyPI (note the index URL):
```
pip install -i https://test.pypi.org/simple/ my-lib==0.1.0
```

### Step 6 — Upload to real PyPI
1) Create a **PyPI** account and an **API token**.
2) Upload:
```
export TWINE_USERNAME=__token__
export TWINE_PASSWORD='pypi-XXXXXXXXXXXXXXXXXXXXXXXX'
python3 -m twine upload dist/*
```
3) Users can then install:
```
pip install my-lib
```

(Alternative: configure `~/.pypirc` with your tokens and omit env vars.)

---

## 7) Versioning rules
- Use **Semantic Versioning**: `MAJOR.MINOR.PATCH`.
- **No re‑uploads**: bump version if you need to fix a release.
- Keep a `CHANGELOG.md` so users can see what changed.

---

## 8) Testing, CI, and quality (recommended)
- `pytest` for tests; run locally and/or in GitHub Actions.
- Format with `black`, type‑check with `mypy` if useful.
- Optional: **Trusted Publishers** via GitHub Actions to publish without local tokens.

---

## 9) Troubleshooting quick hits
- **403 on upload** → Use API token (`__token__`) not username/password.
- **Name taken** → Pick a different `project.name`.
- **Missing files in sdist** → Adjust `MANIFEST.in`.
- **Import works locally but fails after install** → Ensure `src/` layout and correct `[tool.setuptools.packages.find]` settings.
- **License warning** → Use a valid SPDX identifier in `project.license`.
- **Cannot overwrite a release** → Bump `version` and rebuild.

---

## 10) Quick command cheat‑sheet
```
# install tooling
python3 -m pip install -U pip build twine packaging

# clean + build
rm -rf build dist *.egg-info
python3 -m build

# validate files
python3 -m twine check dist/*

# upload to TestPyPI
export TWINE_USERNAME=__token__
export TWINE_PASSWORD='pypi-TEST-XXX'
python3 -m twine upload --repository testpypi dist/*

# upload to PyPI
export TWINE_USERNAME=__token__
export TWINE_PASSWORD='pypi-XXX'
python3 -m twine upload dist/*
```

> Keep this file generic and copy it into any new project to speed up publishing.