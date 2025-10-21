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

