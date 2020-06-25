# Releasing Foundry VTT modules on Github

So you're developing an amazing new Foundry VTT module, but every time you want to release your next version you forget to update all the files in the zip and upload it. Or maybe you feel it's wrong to put the zip file inside the git repository.

Well, you've come to the right address. Let's create a Github workflow to automatically release your latest versions!

## Step 1: Establishing a file structure
Every foundry module consists of at least a manifest (`module.json`) and a javascript file. In this example, I'll also assume you might want to include translations and templates. However, if you don't have them, just ignore them.

Let's assume our module is called `my-module`. Our file structure will look something like this:

```
/my-module/
    - lang/
        - en.json
        - fr.json
    - module.json
    - my-module.js
    - templates/
        - my-template.html
```
*The file structure of our module 'my-module'.*

To get started with Github actions, we need to add another directory first. In the root of your repository add the following folders and file: `.github/workflows/main.yml`. *The `.` (period) in front of "github" is important!*

```
/my-module/
    - .github/
        - workflows/
            - main.yml
    - lang/
    - module.json
    - my-module.js
    - templates/
```

## Step 2: Setting up your first workflow

Inside `.github/workflows/main.yml` add the following text:
```yml
name: my-module CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - name: Create Release
      id: create_latest_release
      uses: ncipollo/release-action@v1
      with:
        allowUpdates: true
        name: Latest
        draft: false
        prerelease: false
        token: ${{ secrets.GITHUB_TOKEN }}
        artifacts: './module.json'
        tag: latest
```

If you push these changes to Github, you should see the first release being created after a few minutes. You can check the progress at `https://github.com/<user>/<repo>/actions`.

When you take a look at the release, you will see that we have successfully uploaded `module.json` and you can use the link to this *asset* as the manifest url in your future `module.json` and in the Foundry administration panel.

## Step 3: Creating and adding the zip file
However, one file is still missing: the `my-module.zip`! To add this file to our release, we need to create it in our workflow. Just above our release we will add the following line:
`    - run: zip -r ./my-module.zip module.json my-module.js lang/ templates/`.

```yml
name: my-module CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - run: zip -r ./my-module.zip module.json my-module.js lang/ templates/
    - name: Create Release
      id: create_latest_release
      uses: ncipollo/release-action@v1
      with:
        allowUpdates: true
        name: Latest
        draft: false
        prerelease: false
        token: ${{ secrets.GITHUB_TOKEN }}
        artifacts: './module.json'
        tag: latest
```

Pushing those changes should update the existing release with `my-module.zip` in the list of assets.

> NB: The attached source code on the release will **not** be updated and the tag will **not** be moved in your repository.
> However, when you download attached `module.json`, you will see it is indeed updated. 😀

## Bonus: Creating a separate release for every version
It is often useful to provide older versions too, in case users want to revert or if you need it for testing purposes.

### Extract the version
Inside `.github/workflows/` create a new file named `get-version.js` with the following content:

```js
var fs = require('fs');
console.log(JSON.parse(fs.readFileSync('module.json', 'utf8')).version);
```

Inside your `.github/workflows/main.yml` add the following change:

```yml
name: my-module CI

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - run: zip -r ./my-module.zip module.json my-module.js lang/ templates/
    - name: Get Version                                   # Run the script that returns the version from `module.json`
      shell: bash
      id: get-version
      run: echo "::set-output name=version::$(node ./.github/workflows/get-version.js)"
    - name: Create Release                                # Create an additional release for this version
      id: create_versioned_release
      uses: ncipollo/release-action@v1
      with:
        allowUpdates: true # set to false if you do not want to allow updates on existing releases
        name: Release ${{ steps.get-version.outputs.version }} # Use the version in the name
        draft: false
        prerelease: false
        token: ${{ secrets.GITHUB_TOKEN }}
        artifacts: './module.json,./my-module.zip'
        tag: ${{ steps.get-version.outputs.version }} # Use the version as the tag
    - name: Create Release
      id: create_latest_release
      uses: ncipollo/release-action@v1
      if: endsWith(github.ref, 'master') # Only update the latest release when pushing to the master branch
      with:
        allowUpdates: true
        name: Latest
        draft: false
        prerelease: false
        token: ${{ secrets.GITHUB_TOKEN }}
        artifacts: './module.json,./my-module.zip'
        tag: latest
```

This will create (or update) two releases: the *Latest* release and a separate release for the version specified in `module.json`.
As an addition, we added `if: endsWith(github.ref, 'master')` to the *Latest* release, so that it only updates when you push new code to the master branch.
So if you do all your experimental development on a separate branch, the *Latest* release stays stable.

## Alternative versioning
Perhaps you prefer to keep control over the tags in your repository and want to release only when you tag a commit with the appropriate version. To do so we need to adjust our workflow a little bit.

### Step 1: Changing the trigger
We start with specifying that the workflow should only run when a commit is pushed with a version tag.

```yml
name: my-module CI

on: [push]
  tags:
    - v*

# rest of the yml...
```

As you can see, we added `tags: - v*` to the trigger. This means it will only run when a tag that starts with `v` is pushed.

Next up, we want to update our manifest with the right version.

### Step 2: Updating the manifest
To update the version in the manifest, we use the same trick we used to read the version from it.
Let's first create the file `.github/workflows/set-version.js` with the following contents:

```js
var fs = require('fs');
const manifest = JSON.parse(fs.readFileSync('module.json', 'utf8'));
const tagVersion = process.argv[2]; // first argument is node, second is the filename of the script, third is the version we pass in our workflow.
if (!tagVersion || !tagVersion.startsWith('v')) {
  console.error(`Invalid version specified: ${tagVersion}`);
  process.exitCode = 1;
} else {
  manifest.version = tagVersion.substring(1); // strip the 'v'-prefix
  fs.writeFileSync('module.json', JSON.stringify(manifest, null, 2)); // pretty print JSON back to module.json
}
```

And of course we need to add this step to our yaml:

```yml
name: my-module CI

on: [push]
  tags:
    - v*

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - run: zip -r ./my-module.zip module.json my-module.js lang/ templates/
    - name: Update Version
      shell: bash
      id: set-version
      run: node ./.github/workflows/set-version.js ${{ github.ref }} # github.ref contains the tag name

# ...
```

### Step 3: Rreating the release
The final step is to use the version in the tag in our release.
Our final yaml will look like this:

```yml
name: my-module CI

on: [push]
  tags:
    - v*

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v2
    - run: zip -r ./my-module.zip module.json my-module.js lang/ templates/
    - name: Update Version
      shell: bash
      id: set-version
      run: node ./.github/workflows/set-version.js ${{ github.ref }}
    - name: Create Release
      id: create_versioned_release
      uses: ncipollo/release-action@v1
      with:
        allowUpdates: true
        name: Release ${{ github.ref }} # Also don't forget to use the tag for the release itself!
        draft: false
        prerelease: false
        token: ${{ secrets.GITHUB_TOKEN }}
        artifacts: './module.json,./my-module.zip'
        tag: ${{ github.ref }} # and one last reference to the tag
```

## Bonus: download count!
Check out the github download shields over at shields.io to get a cool image with the download count:
https://shields.io/category/downloads

Example: https://img.shields.io/github/downloads-pre/janssen-io/foundry-github-workflow/latest/my-module.zip
![my-module-downloads](https://img.shields.io/github/downloads-pre/janssen-io/foundry-github-workflow/latest/my-module.zip)

## Existing FVTT module workflows
- Token Hotbar: https://github.com/janssen-io/foundry-token-hotbar/blob/master/.github/workflows/main.yml
- Forien's Quest Log: https://github.com/Forien/foundryvtt-forien-quest-log/blob/master/.github/workflows/release.yml
