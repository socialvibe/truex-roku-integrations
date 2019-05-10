# Getting started with true[X] on Roku

Documentation and reference apps for true[X]'s Roku-based integration library.

## Overview

In order to support interactive ads on Roku, true[X] has created a renderer component library that can renderer true[X] ads natively, which interfaces with a hosting channel, as well as its existing ad server and content delivery mechanism (e.g. SSAI).

With this library, the host player channel can defer to the TruexAdRenderer when it is required to display a true[X] ad.

## Setup

The true[X] interactive ad component and its rendering logic are distributed as part of a component library. It is required for the hosting channel to add a reference to the component library in order for it to be used, for instance via the following reference added to your channel’s main scene:

    <ComponentLibrary id="TruexAdLibrary" uri="http://static.truex.com.s3.amazonaws.com/roku/TruexAdRenderer-Roku-v0.9.0.pkg"/>

## Next Steps

### Integration Documentation

The [TruexAdRenderer-Roku Documentation](DOCS.md) outlines how to use the TruexAdRenderer, including supported input and output events. 

### Google Ad Manager (GAM) IMA SDK Integration Example

[This repo contains a fully working example](https://github.com/socialvibe/truex-roku-google-ad-manager-reference-app) of a true[X] integration using the Google Ad Manager (GAM) IMA SDK. 