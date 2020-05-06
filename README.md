# Recipes

[![license][license-badge]][license]

This repository contains some use cases of workflows with [actions-ecosystem](https://github.com/actions-ecosystem) GitHub Actions.

If you're not so familiar with GitHub Actions, you may want to read [GitHub Actions Documentation](https://help.github.com/en/actions).

## Automation of updating a Git tag with semver and creating a GitHub release

If you're interested in the latest one, see .github/workflows/release.yml in [actions-ecosystem](https://github.com/actions-ecosystem) repositories.

![screenshot](./docs/assets/screenshot-release-pull-request.png)
![screenshot](./docs/assets/screenshot-release-release.png)

With this workflow, you can automatically update a Git tag and create a GitHub release with only adding a *release label* and optionally a *release note* after a pull request has been merged.

1. [actions-ecosystem/action-get-merged-pull-request](https://github.com/actions-ecosystem/action-get-merged-pull-request) gets a pull request merged with the base branch.
2. [actions-ecosystem/action-release-label](https://github.com/actions-ecosystem/action-release-label) gets a semver update level from a *release label*.
3. [actions-ecosystem/action-get-latest-tag](https://github.com/actions-ecosystem/action-get-latest-tag) fetches the latest Git tag in the repository.
4. [actions-ecosystem/action-bump-semver](https://github.com/actions-ecosystem/action-bump-semver) bumps up the Git tag previously fetched based on the semver update level at the step *1*.
5. *[Optional]* [actions-ecosystem/action-regex-match](https://github.com/actions-ecosystem/action-regex-match) extracts a *release note* from the pull request body.
6. [actions-ecosystem/action-push-tag](https://github.com/actions-ecosystem/action-push-tag) pushes the bumped Git tag with the pull request reference as a message.
7. [actions/create-release](https://github.com/actions/create-release) creates a GitHub release with the Git tag and the *release note* when the semver update level is *major* or *minor*.
8. *[Optional]* [actions-ecosystem/action-create-comment](https://github.com/actions-ecosystem/action-create-comment) creates a comment that reports the new GitHub release.

For further details, see each action document.

```yaml
name: Create Release

on:
  push:
    branches:
      - master

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - uses: actions-ecosystem/action-get-merged-pull-request@v1
        id: get-merged-pull-request
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}

      - uses: actions-ecosystem/action-release-label@v1
        id: release-label
        if: ${{ steps.get-merged-pull-request.outputs.title != null }}
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          labels: ${{ steps.get-merged-pull-request.outputs.labels }}

      - uses: actions-ecosystem/action-get-latest-tag@v1
        id: get-latest-tag
        if: ${{ steps.release-label.outputs.level != null }}
        with:
          semver_only: true

      - uses: actions-ecosystem/action-bump-semver@v1
        id: bump-semver
        if: ${{ steps.release-label.outputs.level != null }}
        with:
          current_version: ${{ steps.get-latest-tag.outputs.tag }}
          level: ${{ steps.release-label.outputs.level }}

      - uses: actions-ecosystem/action-regex-match@v2
        id: regex-match
        if: ${{ steps.bump-semver.outputs.new_version != null }}
        with:
          text: ${{ steps.get-merged-pull-request.outputs.body }}
          regex: '```release_note([\s\S]*)```'

      - uses: actions-ecosystem/action-push-tag@v1
        if: ${{ steps.bump-semver.outputs.new_version != null }}
        with:
          tag: ${{ steps.bump-semver.outputs.new_version }}
          message: "${{ steps.bump-semver.outputs.new_version }}: PR #${{ steps.get-merged-pull-request.outputs.number }} ${{ steps.get-merged-pull-request.outputs.title }}"

      - uses: actions/create-release@v1
        if: ${{ steps.release-label.outputs.level == 'major' || steps.release-label.outputs.level == 'minor' }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: ${{ steps.bump-semver.outputs.new_version }}
          release_name: ${{ steps.bump-semver.outputs.new_version }}
          body: ${{ steps.regex-match.outputs.group1 }}

      - uses: actions-ecosystem/action-create-comment@v1
        if: ${{ steps.bump-semver.outputs.new_version != null }}
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          number: ${{ steps.get-merged-pull-request.outputs.number }}
          body: |
            The new version [${{ steps.bump-semver.outputs.new_version }}](https://github.com/${{ github.repository }}/releases/tag/${{ steps.bump-semver.outputs.new_version }}) has been released :tada:
```

### Check release status

In addition, the following workflow works well with the release workflow above.
This workflow tells you what version will be released with the pull request.

![screenshot](./docs/assets/screenshot-check-release-comment.png)

```yaml
name: Check Release

on:
  pull_request:
    types:
      - labeled

jobs:
  release:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2

      - uses: actions-ecosystem/action-release-label@v1
        id: release-label
        if: ${{ startsWith(github.event.label.name, 'release/') }}

      - uses: actions-ecosystem/action-get-latest-tag@v1
        id: get-latest-tag
        if: ${{ steps.release-label.outputs.level != null }}
        with:
          semver_only: true

      - uses: actions-ecosystem/action-bump-semver@v1
        id: bump-semver
        if: ${{ steps.release-label.outputs.level != null }}
        with:
          current_version: ${{ steps.get-latest-tag.outputs.tag }}
          level: ${{ steps.release-label.outputs.level }}

      - uses: actions-ecosystem/action-create-comment@v1
        if: ${{ steps.bump-semver.outputs.new_version != null }}
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          body: |
            This PR will update [${{ github.repository }}](https://github.com/${{ github.repository }}) from [${{ steps.get-latest-tag.outputs.tag }}](https://github.com/${{ github.repository }}/releases/tag/${{ steps.get-latest-tag.outputs.tag }}) to ${{ steps.bump-semver.outputs.new_version }} :rocket:

            If this update isn't as you expected, you may want to change or remove the *release label*.

```

## License

Copyright 2020 The Actions Ecosystem Authors.

<!-- badge links -->

[license]: LICENSE
[license-badge]: https://img.shields.io/github/license/actions-ecosystem/action-add-labels?style=for-the-badge
