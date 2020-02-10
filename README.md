# Sparch (Single Page Application Archive)

Many modern websites are now Single Page Applications, built with tooling such as Webpack.  Although this provides better dependency management for scripts and assets, serving up the application is still usually done manually and custom for each project.

Sparch is a standard for describing the contents/assets of a single page application, as well as how those assets should be served up to requesting clients.  This repository aims to provide a reference implementation in Go that can be used in containers or standalone applications to serve the Sparch package.  Alternative implementations in languages such as C# or Java can be written, but I only plan to support Go natively right now.

## Motivation

SPAs are often served up as static resources from the file system, however, this presents a couple of issues:

* Caching
  * Static file servers often serve up the file(s) with a long TTL, or none at all.  This can either cause assets to get stale on end user systems (which is more common in my experience), or cause a lot of overhead, downloding new copies of all the assets every time.
  * Although some bundlers such as webpack can include hashes, etc, in the file name, this is usually not possible for the entry points (e.g. main.js), as they need to be referenced in the HTML initially sent to the client.
  * SPAs do not change often (relative to the number of requests), and so proper caching can dramatically reduce the overhead of loading an application.
* Performance
  * The application must request a small HTML file, parse it, and then start requesting the assets it needs.  Some SPAs use Javascript to load subsequent assets, and so you may need multiple round trip requests before the browser knows everything it needs to load, and this adds to page load time.

## The Package

At its heart, a Sparch is a specially formatted zip file (like jar or NuGet files).  The assets can be laid out however the consumer wants, with the exception of the reserved `.meta/` directory, which is used for metadata specific to the sparch.  The zip file can be compressed or uncompressed.

The `.meta/` directory contains at a very minimum a file called `sparch-version.json`.  This is a file that contains a JSON object with one required key: **version**.

```json
{
    "version": "v0.0",
}
```

At the time of this writing, "v0.0" is the only supported version/value for this value.  This version string will determine how the rest of the sparch is handled, and allows for future changes in the specification without causing issues between older and newer servers.  As such, servers should always consult this version first before parsing and handling the rest of the archive.

In summation, a Sparch package is a zip file containing a `sparch-version.json` file with a key of `version`.  

## The Server

The server is the application serving up the Sparch package, and it should do so in a specific way.

The server should respond to any requests for `html` files with a core `html` file that is generated (i.e. it is not part of the package).  This is important because routing will be done by the application.  Any requests for types that aren't `html` will only be served if there is a file with the given name listed in the assets.  All files should include their hash in the ETAG, and respond with 304 if the user agent requests a version of the file with the same ETAG.  The ETAG of the html file will be the hash of the manifest file.

## The Implementation

The default implemenation will be written in Go for the following reasons:

* Excellent HTTP/2 support
* Performance
