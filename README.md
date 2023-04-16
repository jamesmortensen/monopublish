# monopublish-action

This GitHub Action helps publish independent npm packages in a monorepo where the packages are completely unrelated to each other. The motivation for creating this package is to help when working in environments where there is a lot of overhead in creating and setting up a new repository. In many corporate environments, it's easier to reuse a repository than to create a new one, especially for small tools. 

While there exists tools for managing monorepo packages, tools such as Lerna are designed to work with packages that are more strongly related to one another. This tool helps work with ones that are only loosely related and which should be versioned independently.

## Quick Start

### CI Server Setup and Usage

To test and publish a single package called `mypackage`, commit and push the following commit message:

```
$ git commit -am "build(deploy): mypackage"
```

Below is a convenient workflow file example, monopublish.yml, for publishing packages in a monorepo which are identified in the above conventional commit message:

monopublish.yml
```yaml
# This workflow will run tests using node and then publish a package to GitHub NPM Packages when a release is created
# For more information see: https://help.github.com/actions/language-and-framework-guides/publishing-nodejs-packages

name: Publish Node.js GitHub Packages From a Monorepo using monopublish GitHub Action

on:
  pull_request:
    types: [opened, synchronize, closed]
    
jobs:
  build:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    strategy:
      matrix:
        node-version: [16.x, 18.x, 19.x]
    env:
      NODE_AUTH_TOKEN: ${{ secrets.GITHUB_NPM_TOKEN }}
    outputs:
      should_deploy: ${{ steps.mono.outputs.should_deploy }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: ${{ matrix.node-version }}
          registry-url: https://npm.pkg.github.com
          scope: "@<YOUR_SCOPE_HERE>"
      - name:
        id: mono
        uses: ./.github/actions/monopublish/
        with:
          check-if-deploy: 'true'
      - name: Install npm modules
        uses: ./.github/actions/monopublish/
        with:
          npm-exec: install
      - name: Execute unit tests
        uses: ./.github/actions/monopublish/
        with:
          npm-exec: test
      - run: echo "Should deploy in action:" ${{ steps.mono.outputs.should_deploy }}

  publish-npm:
    needs: build
    if: ${{ needs.build.outputs.should_deploy && github.event.pull_request.merged == true && github.base_ref == 'main' }}
    runs-on: ubuntu-latest
    timeout-minutes: 10
    env:
      NODE_AUTH_TOKEN: ${{ secrets.GITHUB_NPM_TOKEN }}
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
          registry-url: https://npm.pkg.github.com
      - name: NPM Mono Publish - Publish package identified in commit message
        uses: ./.github/actions/monopublish/
        with:
          dry-run: false
          npm-exec: publish
```
        
If you want to trigger just testing the build without publishing, with the above monopublish.yml file, just add an empty commit with `ci` as the type and the name of the package. For example, the following empty commit triggers the workflow, but only for building and testing the package, not publishing:

```
$ git commit --allow-empty -m "build(ci): mypackage"
```

### Configuring NPM Publish to GitHub Actions

It may help to configure the NPM repository you're publishing to, although you can also just do this in the monopublish.yml workflow file:

package.json:
```json
"publishConfig": {
    "registry": "https://npm.pkg.github.com"
  },
```

## Advanced Usage

The workflow and action are the recommended ways to publish from GitHub Actions; however, there are scripts that can be used locally as well.

### monorepo script usage

The scripts are used by the GitHub Action. They can be used outside of the action to execute npm run scripts on individual packages, including publishing them.  Below are some examples:

Install node modules in a specific package, using PACKAGE_NAME as the folder name:

```
$ monopublish -s @jamesmortensen -p myscopedpackage -n -e install
```

Publish a scoped package without bumping version in package.json:

```
$ monopublish -s @scope -p package -n -e publish
```

Publish a non-scoped package without bumping version:

```
$ monopublish -p package -n -e publish
```

Install npm modules only in the package:

```
$ monopublish -p package -n -e install
```

Execute test cases only in the package:

```
$ monopublish -p package -n -e test
```

NOTE: With the -e argument, you can execute any NPM run script in the package.


### Troubleshooting and Notes for Maintainers

This is a proof of concept written in Bash and GitHub Actions. While the monopublish script is a bit more refined, the push-to-deploy script is not so user friendly.

To examine a conventional commit and deploy (or run an npm run script) from a pushed commit message, there's a script to call, included in the action, that will examine the commit message and determine if a package should be published. Here's an example that runs npm test on the package identified in the most recent commit message, assuming that commit message is `build(deploy): mypackage`

```bash
$ NPM_EXEC=test GITHUB_ACTION_PATH=../monopublish DRY_RUN_ARG="-d" ./.github/actions/monopublish/push-to-deploy "`git log | head -5 | tail -1`"
```

This will test the package named "mypackage", using the run script `npm test`.   Change `NPM_EXEC` to `publish` and it publishes the package instead. Use `install` to install npm modules for that package.

To make this cleaner, we can convert the environment variable inputs to command-line arguments. I am also considering rewriting this in either Node.js or Go. The scripts could also make nice CLI tools for working with independent monorepo packages.


## License

Copyright (c) James Mortensen, 2023 MIT License
