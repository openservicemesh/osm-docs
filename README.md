# Open Service Mesh Docs

:book: This section contains the [OSM Docs](https://github.com/openservicemesh/osm-docs)

:ship: Also the website config to generate [docs.openservicemesh.io](https://docs.openservicemesh.io)

:link: Looking for the main OSM website? Visit [osm-www](https://github.com/openservicemesh/osm-www)


## Editing Content

docs.openservicemesh.io is a static site. The documentation content needs to be located at `content/docs/`.

The content served on [https://docs.openservicemesh.io](https://docs.openservicemesh.io) is served from the latest release on this repo. Most updates should be made only in main and can be previewed at [https://main--osm-docs.netlify.app/](https://main--osm-docs.netlify.app/).

If it's necessary to change published release-specific docs, those changes should be made in the release-specific branch serving those docs. Once configured as described in [Adding release-specific docs](#adding-release-specific-docs), PRs to that branch will auto-build just like PRs to main.

References to the osm branch, osm version, and envoy version should be parameterized unless they are in a sample output. For example, a reference to OSM's `constants.go` should be parameterized as `https://github.com/openservicemesh/osm/blob/{{< param osm_branch >}}/pkg/constants/constants.go`.

To ensure the docs content renders correctly in the theme, each page will need to have [front matter](https://gohugo.io/content-management/front-matter/) metadata. Example front matter:

```
---
title: "Docs Home"
linkTitle: "Home"
description: "OSM Docs Home"
weight: 1
type: docs
---
```

## Front Matter Notes:

* inclusion of `type: docs` is important for the theme to properly index the site contents
* the `linkTitle` attribute allows you to simplify the name as it appears in the left-side nav bar - ideally it should be short and clear - whereas the title can handle long form names for pages/documents.

## Adding release-specific docs

### Create a release branch

Look for a branch in the upstream repo named `release-vX.Y`, where `X` and `Y` correspond to the major and minor version of the new release. For example, [release-v0.8](https://github.com/openservicemesh/osm-docs/tree/release-v0.8). If the branch already exists, move to the next step.

Identify the base commit in the main branch for the release and cut a release branch off main.

> Note: Care must be taken to ensure the release branch is created from a commit meant for the release. If unsure about the commit to use to create the release branch, please open an issue in the osm repo and a maintainer will assist you with this.

```console
$ git checkout -b release-<version> <commit-id> # ex: git checkout -b release-v0.4 0d05587
```

Push the release branch to the upstream repo (NOT forked), identified here by the upstream remote.

```console
$ git push upstream release-<version> # ex: git push upstream release-v0.4
```

Netlify will auto-deploy the branch to a url like [https://release-v0-8--osm-docs.netlify.app/](https://release-v0-8--osm-docs.netlify.app/). This can be used to preview and test that the branch builds correctly.

### Update the release references

Proceed with the following steps once the release branch has been created in the OSM repo.

1. Create a new branch off of the release branch to maintain updates specific to the new version. Let's call it the patch branch. The patch branch should not be created in the upstream repo.
2. On the patch branch, update the `osm_branch`, `osm_version`, and `envoy_version` in [config.toml](https://github.com/openservicemesh/osm-docs/blob/main/config.toml) to the new release versions.

    ```toml
    osm_branch = "release-v0.8"
    osm_version = "v0.8.0"
    envoy_version = "v1.17.2"
    ```
3. Create a pull request from the patch branch to the release branch. Proceed to the next step once the pull request is approved and merged.

### After cutting the OSM release

1. Open an issue in this repo asking for a new DNS record be added to the site (via Netlify), to assign a subdomain to the deployed branch.
2. When published, the newly-added branch will function like [https://release-v0-8.docs.openservicemesh.io/](https://release-v0-8.docs.openservicemesh.io/)

### After publishing the new release-specific docs

#### Update the release branch

1. Create another patch branch off of the release branch (or use the existing patch branch after fetching and rebasing on the release branch).
2. On the patch branch, add the new release as a version parameter in [config.toml](https://github.com/openservicemesh/osm-docs/blob/main/config.toml) with the `latest` tag. The version parameters represent all currently supported versions of OSM and are used to populate the Release drop down menu on the site. The url for the new release is formatted `https://release-vX-Y.docs.openservicemesh.io/`.

    ```toml
    [[params.versions]]
        version = "v0.8 (latest)"
        url = "https://release-v0-8.docs.openservicemesh.io/"
    ```

3. Create a pull request from the patch branch to the release branch. Proceed to the next step once the pull request is approved and merged.

#### Update main and previous release branches

1. Update the `redirects` in `netlify.toml` on the main branch to redirect `https://docs.openservicemesh.io/` to `https//release-vX-Y.docs.openservicemesh.io/` where `release-vX-Y` is the newest release.
2. Each previous release-specific site that is still supported needs to be able to access the latest release from the Release drop down. On the previous release branches, update the [config.toml](https://github.com/openservicemesh/osm-docs/blob/main/config.toml) to list the new release version as shown above.
3. The `latest` tag must be removed from all previous versions. For example, the `latest` tag must be removed from `v0.7 (latest)` on the `release-v0.7` branch.
4. Add the [banner](https://github.com/openservicemesh/osm-docs/blob/release-v0.9/themes/dosmy/layouts/partials/banner.html) to a previous release specific site if it has not been configured.
5. Add or update the version banner parameter in `config.toml` to enable the banner at the top of each previous release-specific site that will tell visitors which version they are looking at. For example, the version banner for `release-v0-7` would be configured as follows:

    ```toml
    [params.versionbanner]
	    show = true
	    archive = "v0.7"
	    latest = "https://release-v0-8.docs.openservicemesh.io/"
    ```

# Site Development

## Notes

* built with the [Hugo](https://gohugo.io/) static site generator
* custom theme uses [Docsy](https://www.docsy.dev/) as a base, with [Bootstrap](https://getbootstrap.com/) as the underlying css framework and some [OSM custom sass](https://github.com/openservicemesh/osm/pull/1840/files#diff-374e78d353e95d66afe7e6c3e13de4aab370ffb117f32aeac519b15c2cbd057aR1)
* deployed to [Netlify](https://app.netlify.com/sites/osm-docs/deploys) via merges to main. (@flynnduism can grant additional access to account)
* metrics tracked via Google Analytics

## Install dependencies:

* Hugo [installation guide](https://gohugo.io/getting-started/installing/)
* NPM packages are installed by running `yarn`. [Install Yarn](https://yarnpkg.com/getting-started/install) if you need to.

## Run the site:

```
// install npm packages
yarn

// rebuild the site (to compile latest css/js)
hugo

// or serve the site for local dev
hugo serve
```

## Deploying the site:

The site auto deploys the main branch via [Netlify](https://app.netlify.com/sites/osm-docs/deploys). Once pull requests are merged the changes will appear at docs.openservicemesh.io after a couple of minutes. Check the [logs](https://app.netlify.com/sites/osm-docs/deploys) for details.

[![Netlify Status](https://api.netlify.com/api/v1/badges/8c8b7b52-b87f-42e0-949a-a784c3ca6d9a/deploy-status)](https://app.netlify.com/sites/osm-docs/deploys)

`hugo serve` will run the site locally at [localhost:1313](http://localhost:1313/)
