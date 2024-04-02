# Get and Set New Version Action

This GitHub Action fetches the latest version of a package from GitHub Packages, compares it with the current version
specified, and determines the new version to be used. It's designed to automatically increment the patch version unless
the current version is ahead in terms of major or minor versions.

## Inputs

### `github_token`

**Required** The GitHub token used for authentication when fetching the latest package version from GitHub Packages.
This token should have the necessary permissions to access the repository and packages.

### `package_url`

**Required** The URL to the package versions on GitHub Packages. This should include the API endpoint that lists
versions for a specific package.

Example: `https://api.github.com/orgs/blkndn/packages/maven/es.bulkynaden.spring.exceptions`

### `current_version`

**Required** The current version of the package. This version will be compared with the latest version fetched from
GitHub Packages to determine the new version.

Example: `1.0.0`

## How to Use

To use this action in your workflow, you will need to specify the `github_token`, `package_url`, and `current_version`.
Here is an example of how to include it in a workflow step:

```yaml
- name: Get Latest Version from GitHub Packages and Decide New Version
  uses: blkndn/composite/get-and-set-new-version@main
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    package_url: 'https://api.github.com/orgs/blkndn/packages/maven/es.bulkynaden.spring.exceptions'
    current_version: ${{ env.CURRENT_VERSION }}