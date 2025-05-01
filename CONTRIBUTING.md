# Contributing to TeachBooks
We welcome any kind of contributions to our software and documentation, from simple comment or question to a full fledged pull request. Please read and follow our Code of Conduct.

A contribution can be one of the following cases:

1. you have a question;
2. you think you may have found a bug (including unexpected behavior);
3. you want to make some kind of change to the code base (e.g. to fix a bug, to add a new feature, to update documentation).
4. you want to make a release for a sphinx extension
5. you want to publish your tool to PyPI

The sections below outline the steps in each case.

## You have a question
1. Use the search functionality [in the organization's discussions page](https://github.com/orgs/TeachBooks/discussions) or in the issues tab of a specific repository to see if someone already filed the same issue;
2. If your issue search did not yield any relevant results, make a new issue or discussion post;

## You think you may have found a bug
1. Use the search functionality [in the organization's discussions page](https://github.com/orgs/TeachBooks/discussions) or in the issues tab of a specific repository to see if someone already filed the same issue;
2. If your issue search did not yield any relevant results, make a new issue or discussion post, making sure to provide enough information to the rest of the community to understand the cause and context of the problem. Depending on the issue, you may want to include:
  - the SHA hashcode of the commit that is causing your problem;
  - some identifying information (name and version number) for dependencies you’re using;
  - information about the operating system;

## You want to make some kind of change to the code base of a certain tool or documentation
1. (important) Announce your plan to the rest of the community before you start working. This announcement should be in the form of a (new) issue;
2. (important) Wait until some kind of consensus is reached about your idea being a good idea; 
3. If needed, fork the repository to your own Github profile and create your own feature branch off of the latest main commit (typically the stable or develop branch). While working on your feature branch, make sure to stay up to date with the main branch by pulling in changes, possibly from the ‘upstream’ repository (follow the instructions here and here);
4. Update or expand the documentation
5. Push your feature branch to (your fork of) the TeachBooks repository on GitHub;
6. Create the pull request, e.g. following the instructions here.

In case you feel like you’ve made a valuable contribution, but you don’t know how to write or run tests for it, or how to generate the documentation: don’t let this discourage you from making the pull request; we can help you! Just go ahead and submit the pull request, but keep in mind that you might be asked to append additional commits to your pull request.

## You want to test your GitHub workflow
To be written

## You want to test your Sphinx extension
To be written

## You want to make a release for a sphinx extension
To be written

## You want to publish your tool to PyPi
You can publish your package to PyPI by making use of GitHub Actions. This involves multiple steps. 

### 1. Prepare Your Project File Structure
The project file hierarchy should (at least) contain the following:

- A src directory containing a subdirectory **with the same name as your package**. The subdirectory should contain the `__init__.py` and any other `.py` files.
- A `pyproject.toml` file, which is the configuration file telling Python tools how to build your packages and what dependencies are needed when doing so. For more information about the `pyproject.toml` file, you can consult this [link](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/). Typically, this file looks as follows:

```toml
[build-system]
requires = ["hatchling", "hatch-vcs"]
build-backend = "hatchling.build"

[project]
name = "<your-package-name>"
dynamic=["version"]
authors = [
    { name = "<Name>", email = "<Email>" },
]
description = "<Your description>"
readme = "README.md"
dependencies = [
    “sphinx”,
    "<other-package-dependencies>"
]
requires-python = ">=3.10"

[tool.hatch]
version.source = "vcs"
build.hooks.vcs.version-file = "src/<your-package-name>/_version.py"
```

- Optionally, you can include files such as the `README.md`, or `LICENSE`.

### 2. Configure trusted publishing in PyPI
- Go to https://pypi.org/manage/account/publishing/.
- Fill in the name you wish to publish your new PyPI project under. This needs to correspond with the name value in your `pyproject.toml`. Then, fill in the GitHub repository owner’s name (org or user), repository name, and the name of the release workflow file under the `.github/` folder - in this case, this will be `python-publish.yml`.  Finally, add the name of the GitHub Environment (pypi) you’re going to set up under your repository. Then register the trusted publisher. 

### 3. Create a new GitHub Actions Workflow 
Click the *Actions* tab in your repository, and then select *New workflow*. This will show several suggested workflow options. Navigate and find the *Publish Python Package*, and click on *Configure*. This will display a pre-filled workflow for publishing your package to PyPI. To extend the functionality to also creating GitHub releases whenever a new tag is pushed, you can replace the contents of the default `python-publish.yml` with the following:

```yaml
# This workflow will upload a Python Package using Twine when a release is created
# For more information see: https://docs.github.com/en/actions/automating-builds-and-tests/building-and-testing-python#publishing-to-package-registries

# This workflow uses actions that are not certified by GitHub.
# They are provided by a third-party and are governed by
# separate terms of service, privacy policy, and support
# documentation.

name: Publish <Your Package Name> Package to PyPI

on:
  push

jobs:
  build:
    name: Build distribution
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4
      with:
        persist-credentials: false
    - name: Set up Python
      uses: actions/setup-python@v5
      with:
        python-version: "3.x"
    - name: Install pypa/build
      run: >-
        python3 -m
        pip install
        build
        --user
    - name: Build a binary wheel and a source tarball
      run: python3 -m build
    - name: Store the distribution packages
      uses: actions/upload-artifact@v4
      with:
        name: python-package-distributions
        path: dist/

  publish-to-pypi:
    name: >-
      Publish Python distribution to PyPI
    if: startsWith(github.ref, 'refs/tags/') # only publish to PyPI on tag pushes
    needs:
    - build
    runs-on: ubuntu-latest
    environment:
      name: pypi
      url: https://pypi.org/p/<your-package-name>
    permissions:
      id-token: write  # IMPORTANT: mandatory for trusted publishing

    steps:
    - name: Download all the dists
      uses: actions/download-artifact@v4
      with:
        name: python-package-distributions
        path: dist/
    - name: Publish distribution to PyPI
      uses: pypa/gh-action-pypi-publish@release/v1

  github-release:
    name: >-
      Sign the Python distribution with Sigstore
      and upload them to GitHub Release
    needs:
    - publish-to-pypi
    runs-on: ubuntu-latest

    permissions:
      contents: write  # IMPORTANT: mandatory for making GitHub Releases
      id-token: write  # IMPORTANT: mandatory for sigstore

    steps:
    - name: Download all the dists
      uses: actions/download-artifact@v4
      with:
        name: python-package-distributions
        path: dist/
    - name: Sign the dists with Sigstore
      uses: sigstore/gh-action-sigstore-python@v3.0.0
      with:
        inputs: >-
          ./dist/*.tar.gz
          ./dist/*.whl

    - name: Determine Release Tag
      id: release_tag
      run: echo "GITHUB_REF_NAME=${GITHUB_REF#refs/tags/}" >> $GITHUB_ENV

    - name: Check if release exists
      id: check_release 
      env:
        GITHUB_TOKEN: ${{ github.token }}
      run: |
        if gh release view "$GITHUB_REF_NAME" --repo "$GITHUB_REPOSITORY"; then
          echo "RELEASE_EXISTS=true" >> $GITHUB_ENV
        else 
          echo "RELEASE_EXISTS=false" >> $GITHUB_ENV
        fi

    - name: Create GitHub Release (if not exists)
      if: env.RELEASE_EXISTS == 'false'
      env:
        GITHUB_TOKEN: ${{ github.token }}
      run: >-
        gh release create
        "$GITHUB_REF_NAME"
        --repo "$GITHUB_REPOSITORY"
        --notes "Release $GITHUB_REF_NAME"

    - name: Mark as Latest Release (if created)
      if: env.RELEASE_EXISTS == 'false'
      env:
        GITHUB_TOKEN: ${{ github.token }}
      run: >-
        gh release edit 
        "$GITHUB_REF_NAME" 
        --repo "$GITHUB_REPOSITORY" 
        --latest

    - name: Upload artifact signatures to GitHub Release
      env:
        GITHUB_TOKEN: ${{ github.token }}
      # Upload to GitHub Release using the `gh` CLI.
      # `dist/` contains the built packages, and the
      # sigstore-produced signatures and certificates.
      run: >-
        gh release upload
        "$GITHUB_REF_NAME" dist/**
        --repo "$GITHUB_REPOSITORY"
```

Make sure to modify the environment url within the `publish-to-pypi` job to take the name of your package (should coincide with the name field from the `pyproject.toml` file). Also fill in the workflow name with the name of your project. 

Once you’re done, press *Commit changes* to add the workflow to the repository. 

You're all set! This workflow will build the package on each push, and will publish a new PyPI version with a corresponding GitHub release whenever a tag is pushed. 

**IMPORTANT!** Make sure that all tags have different versions and that they are in ascending order.

For more detailed information regarding the publishing process, please consult this [link](https://packaging.python.org/en/latest/guides/publishing-package-distribution-releases-using-github-actions-ci-cd-workflows/).

## Releases and Versioning
Semantic numbering is used: vA.B.C, where patches advance C and minor releases advance B.
