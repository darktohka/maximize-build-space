# Maximize available disk space for build tasks

By default, public shared Github runners come with around 25-29 GB of free disk space to be consumed by your build job.
If this is too little for you, and your build job runs out of disk space, this action might be for you.

If not: please go on, there's nothing to see here.

This action maximizes the available disk space on public shared GitHub runners by
optionally removing unnecessary preinstalled software. Depending on your use case, you can gain
tens of gigabytes of additional space.

The idea came up when I tried to build a Fedora Linux Kernel RPM on Github, and the Job aborted due to disk space issues.

## Caveats

- **This action is a hack, really**, and on a "works for me" basis. It has been built by reverse-enginnering undocumented parts of the Github runner setup, which might change in the future -- up to the point where this action ceases to work. For sure you're voiding the warranty on Github runners when using this. **If you are not running into disk space issues with the default runner setup, don't use it.**
- Removal of unnecessary software is implemented by `rm -rf` on specific folders, not by using a package manager or anything sophisticated. While this is quick and easy, it might delete dependencies that are required by your job and so break your build (e.g. because your build job uses a .NET based tool and you removed the required runtime).
- The folders to delete are hardcoded for the GitHub-hosted Ubuntu images. On other images, an enabled option simply frees nothing (or less than advertised), and the paths are subject to change by GitHub without notice.
- Only the root filesystem is affected. Nothing is repartitioned, mounted or moved, and your workspace is left untouched, so the action can run at any point before you need the space.

Btw: the choice of removable packages is not an expression of dislike against these -- they were just the largest folders for a single toolkit I could find quickly.

## How it works

At the time of writing, public [Github-hosted runners](https://docs.github.com/en/actions/using-github-hosted-runners/about-github-hosted-runners) run on [Azure DS2_v2 virtual machines](https://docs.microsoft.com/en-us/azure/virtual-machines/dv2-dsv2-series#dsv2-series), featuring an 84GB OS disk on `/`.

The root file system has ~29GB (of 84GB) available, the rest being consumed by the preinstalled build environment. Github runners come with a rich choice of software, see the image descriptions for [Ubuntu 22.04](https://github.com/actions/runner-images/blob/main/images/linux/Ubuntu2204-Readme.md) or [Ubuntu 24.04](https://github.com/actions/runner-images/blob/main/images/linux/Ubuntu2404-Readme.md). This is great to support a wide variety of workflows out of the box, but also consumes a lot of space by providing programs that might be unnecessary for an individual build job.

This action does the following:

1. (Optionally) removes unwanted preinstalled software
1. (Optionally) prunes cached Docker images

This frees the space previously taken by the toolkits you removed, which is then available to your build job on the same filesystem.

## Usage

You can use this action as the first build step, so that all subsequent steps benefit from the freed space. It does not touch `$GITHUB_WORKSPACE`, so it is safe to run before `actions/checkout` as well.

With all options disabled (the default) the action does not free any space. Enable the options for the toolkits your build job does not need. See [this table](https://github.com/darktohka/maximize-build-space/blob/test-report/README.md) for examples and the expectable amount of space.

When removing software, consider that the removal of large amounts of files (which this is) can take minutes to complete. On the upside, you'll get more than 60 GB of disk space available if you actually need it.

```yaml
name: My build action requiring more space
on: push

jobs:
  build:
    name: Build my artifact
    runs-on: ubuntu-latest
    steps:
      - name: Maximize build space
        uses: darktohka/maximize-build-space@master
        with:
          remove-dotnet: 'true'
          remove-android: 'true'
          remove-haskell: 'true'
          remove-codeql: 'true'
          remove-docker-images: 'true'
      - name: Checkout
        uses: actions/checkout@v3

      - name: Build
        run: |
          echo "Free space:"
          df -h
```

## Inputs

All inputs are optional and default to `'false'`, i.e. nothing is removed. Enable only what your build job does not need.

```yaml
  remove-dotnet:
    description: 'Removes .NET runtime and libraries. (frees ~17 GB)'
    required: false
    default: 'false'
  remove-android:
    description: 'Removes Android SDKs and Tools. (frees ~11 GB)'
    required: false
    default: 'false'
  remove-haskell:
    description: 'Removes GHC (Haskell) artifacts. (frees ~2.7 GB)'
    required: false
    default: 'false'
  remove-codeql:
    description: 'Removes CodeQL Action Bundles. (frees ~5.4 GB)'
    required: false
    default: 'false'
  remove-docker-images:
    description: 'Removes cached Docker images. (frees ~3 GB)'
    required: false
    default: 'false'
```
