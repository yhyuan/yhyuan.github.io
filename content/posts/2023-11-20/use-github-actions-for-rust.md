---
title: "Use Github Actions for Rust"
date: 2023-11-20T09:06:01-05:00
tags: ['rust', 'Github']
draft: false
---

## Use Github Actions for Rust

Find a [blog post](https://dzfrias.dev/blog/deploy-rust-cross-platform-github-actions)
and [another post](https://eftakhairul.com/githubs-repository-not-found-error-and-how-to-fix-it/)

## How to Deploy Cross-Platform Rust Binaries with GitHub Actions
rust
2023-06-23

If you’re ready to release the first version of your Rust project, it’s likely that you want to upload some assets containing pre-built binaries (for macOS, Linux, and Windows) to your GitHub releases page. This is one of the earliest steps in making your project widely available, and will help greatly when setting up other installation methods like with Homebrew. However, figuring out cross-compilation is not easy, especially when doing it through a GitHub Action.

By the end of this post, you’ll have a fully working deployment pipeline that automatically creates releases and uploads binaries to that release for Windows, Linux, and macOS. You’ll have a releases page that looks something like this:

assets

You might even learn a bit about GitHub Actions along the way!

### Workflow Trigger
First, we need the condition in which we should run our action. Since releases are coupled with git tags, we want our automatic-releaser to run when a new tag is pushed to GitHub. Not only that, but we want it to be a version tag (like 1.0.0 or 0.2.4), so GitHub doesn’t run our action for a tag that’s not related to releases.

This can be done with the following code:

.github/workflows/deploy.yml
```
# Just setting the name of our action
name: Deploy

on:
  push:
    tags:
      # Regex for a version number such as 0.2.1
      - "[0-9]+.[0-9]+.[0-9]+"
```
Alright, with that out of the way, let’s get into what our action will actually do!

### Building the Action
To compile our binaries cross-platform, we’ll need to create a new job. Let’s call it build-and-upload, because this job will also be in charge of building the binaries and uploading them to a new release. A job is a bundle of steps, and a step is just anything our action does!

#### Defining the Metadata
Let’s begin writing our job. First, we need to define the metadata before we get into the steps.

.github/workflows/deploy.yml
```
# ...

jobs:
  # ...

  build-and-upload:
    name: Build and upload
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        # You can add more, for any target you'd like!
        include:
          - build: linux
            os: ubuntu-latest
            target: x86_64-unknown-linux-musl

          - build: macos
            os: macos-latest
            target: x86_64-apple-darwin

          - build: windows-gnu
            os: windows-latest
            target: x86_64-pc-windows-gnu
```
You might notice the matrix! The matrix is an important part, as it means our job will run for each item in included in the matrix. You can see the matrix being used in the runs-on key, meaning our job will run on a different operating system each time, where the operating system is defined by the os key of each matrix item.

#### Installing Dependencies
Okay, with that finished, we need to start adding steps. The first two are easy: just clone our repository and install Rust!

.github/workflows/deploy.yml
```
# ...

build-and-upload:
  # ...
  steps:
    - name: Clone repository
      uses: actions/checkout@v3

    - name: Install Rust
      # Or @nightly if you want
      uses: dtolnay/rust-toolchain@stable
      # Arguments to pass in
      with:
        # Make Rust compile to our target (defined in the matrix)
        targets: ${{ matrix.target }}
```
These step use the uses key. uses tells our runner that we’re calling an external action, that’s premade. You can even see the actions on GitHub that we’re calling! actions/checkout is a common one - it clones our repository. Arguments to an action are passed in using the with key. We utilize this in dtolnay/rust-toolchain to tell it what target our Rust should compile to.

#### Getting the Version
Another one of the “preparation steps” we need to take is getting the version. We’ll use this information later!

.github/workflows/deploy.yml
```
- name: Get the release version from the tag
  shell: bash
  run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_ENV
```
Instead of a uses key, you might notice the shell: bash. This basically tells GitHub Actions that we’re going to run a shell script. After that, we use the run key to specify what we’re actually going to be executing.

The script itself puts the version from our tag into an environment variable, $VERSION. Using environment variables is a very common idiom for getting steps to communicate with each other, as we’ll now be able to access our version tag name (like “0.1.0”) with ${{ env.VERSION }} for the rest of this job.

#### Building the Binaries
Okay, now we can get to building the binaries. For this, we’ll use cross, a Rust tool that uses Docker to build your project cross-platform. There’s already a super simple action that allows us to use cross!

Let’s put that in our step list:

.github/workflows/deploy.yml
```
- name: Build
  uses: actions-rs/cargo@v1
  with:
    use-cross: true
    command: build
    args: --verbose --release --target ${{ matrix.target }}
```
This step tells GitHub Actions that we’d like to build our project using cross. Similar to cargo build, our binary will end up in target/<TARGET>/release/<BINARY_NAME>, where <TARGET> is our target key of the matrix. The name of the binary obviously depends on your project.

#### Compressing the Binaries
We’re almost done! Now that we have our binaries built, we need to compress them into a .tar.gz file (or .zip) so they’re easier to download from our assets page. Let’s do that with this step:

.github/workflows/deploy.yml
```
- name: Build archive
  shell: bash
  run: |
    # Replace with the name of your binary
    binary_name="<BINARY_NAME>"

    dirname="$binary_name-${{ env.VERSION }}-${{ matrix.target }}"
    mkdir "$dirname"
    if [ "${{ matrix.os }}" = "windows-latest" ]; then
      mv "target/${{ matrix.target }}/release/$binary_name.exe" "$dirname"
    else
      mv "target/${{ matrix.target }}/release/$binary_name" "$dirname"
    fi

    if [ "${{ matrix.os }}" = "windows-latest" ]; then
      7z a "$dirname.zip" "$dirname"
      echo "ASSET=$dirname.zip" >> $GITHUB_ENV
    else
      tar -czf "$dirname.tar.gz" "$dirname"
      echo "ASSET=$dirname.tar.gz" >> $GITHUB_ENV
    fi
```
This step is pretty long, but not that complicated. Let’s go over how it works:

Create a directory that we’ll eventually compress (like prj-0.2.1-x86_64-apple-darwin)
Move our built binary into that directory. The name changes depending on the OS
Compress. Windows uses .zip, and Unix uses .tar.gz
Put the path to that compressed file into an environment under the name of $ASSET
You can see our $VERSION environment variable coming into play here!

#### Uploading the Binaries
With our binaries compressed, we’re ready to upload upload them! This is the final step of our job!

.github/workflows/deploy.yml
```
- name: Upload the binaries
  uses: softprops/action-gh-release@v1
  with:
    files: |
      ${{ env.ASSET }}
```
Compared to some of the other steps, this one’s pretty easy to understand. All we do is use the pre-made softprops/action-gh-release to upload our compressed files (using the $ASSET environment variable we created in the previous step).

And we’re done! Any time we push to GitHub with a tag like 2.1.0, we’ll create a release, build our cross-platform binaries, and upload them!

#### Wrapping Up + Final Code
I hope this post was informative and interesting, and good luck with deploying your future projects! If there was anything wrong with this post, please submit an issue on GitHub.

Here’s the final action, just for good measure:

.github/workflows/deploy.yml
```
name: Deploy

on:
  push:
    tags:
      - "[0-9]+.[0-9]+.[0-9]+"

permissions:
  contents: write

jobs:
  build-and-upload:
    name: Build and upload
    runs-on: ${{ matrix.os }}

    strategy:
      matrix:
        # You can add more, for any target you'd like!
        include:
          - build: linux
            os: ubuntu-latest
            target: x86_64-unknown-linux-musl

          - build: macos
            os: macos-latest
            target: x86_64-apple-darwin

    steps:
      - name: Checkout
        uses: actions/checkout@v3

      - name: Get the release version from the tag
        shell: bash
        run: echo "VERSION=${GITHUB_REF#refs/tags/}" >> $GITHUB_ENV

      - name: Install Rust
        # Or @nightly if you want
        uses: dtolnay/rust-toolchain@stable
        # Arguments to pass in
        with:
          # Make Rust compile to our target (defined in the matrix)
          targets: ${{ matrix.target }}

      - name: Build
        uses: actions-rs/cargo@v1
        with:
          use-cross: true
          command: build
          args: --verbose --release --target ${{ matrix.target }}

      - name: Build archive
        shell: bash
        run: |
          # Replace with the name of your binary
          binary_name="<BINARY_NAME>"

          dirname="$binary_name-${{ env.VERSION }}-${{ matrix.target }}"
          mkdir "$dirname"
          if [ "${{ matrix.os }}" = "windows-latest" ]; then
            mv "target/${{ matrix.target }}/release/$binary_name.exe" "$dirname"
          else
            mv "target/${{ matrix.target }}/release/$binary_name" "$dirname"
          fi

          if [ "${{ matrix.os }}" = "windows-latest" ]; then
            7z a "$dirname.zip" "$dirname"
            echo "ASSET=$dirname.zip" >> $GITHUB_ENV
          else
            tar -czf "$dirname.tar.gz" "$dirname"
            echo "ASSET=$dirname.tar.gz" >> $GITHUB_ENV
          fi

      - name: Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            ${{ env.ASSET }}
```