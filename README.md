# Open Service Mesh Docs

:book: This section contains the [OSM Docs](https://github.com/openservicemesh/osm-docs)

:ship: Also the website config to generate [docs.openservicemesh.io](https://docs.openservicemesh.io)

:link: Looking for the main OSM website? Visit [osm-www](https://github.com/openservicemesh/osm-www)


## Editing Content

docs.openservicemesh.io is a static site. The documentation content needs to be located at `content/docs/`.

The content served on [https://docs.openservicemesh.io](https://docs.openservicemesh.io) is served from the main branch of this repo. Most updates should be made only in main.

If it's necessary to change published release-specific docs, those changes should be made in the release-specific branch serving those docs. Once configured as described in [Adding release-specific docs](#adding-release-specific-docs), PRs to that branch will auto-build just like PRs to main.

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
* the `linkTitle` attribute allows you to simplify the name as it appears in the left-side nav bar - ideally it should be short and clear - whereas the title can handle longform names for pages/documents.

## Adding release-specific docs

Steps to add a release-specific version of the docs site:

1. Create a release branch from main, named after the released major and minor version, like [release-v0.8](https://github.com/openservicemesh/osm-docs/tree/release-v0.8).
1. List the new branch in the [config.toml](https://github.com/openservicemesh/osm-docs/blob/main/config.toml#L84-L89) for all currently-displayed versions.
1. Open an issue in this repo asking for the Netlify config to be created and the site update to be completed.
1. When published, the newly-added branch will function like [https://release-v0-8.docs.openservicemesh.io/](https://release-v0-8.docs.openservicemesh.io/)


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

The site auto deploys the main branch via [Netlify](https://app.netlify.com/sites/osm-docs). Once pull requests are merged the changes will appear at docs.openservicemesh.io after a couple of minutes. Check the [logs](https://app.netlify.com/sites/osm-docs/deploys) for details.

[![Netlify Status](https://api.netlify.com/api/v1/badges/8c8b7b52-b87f-42e0-949a-a784c3ca6d9a/deploy-status)](https://app.netlify.com/sites/osm-docs/deploys)

`hugo serve` will run the site locally at [localhost:1313](http://localhost:1313/)
