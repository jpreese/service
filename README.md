# service
The service side of clearlydefined.io

## Getting Started
1. Set the following environment variables:
   * CLEARLY_DEFINED_CURATION_GITHUB_OWNER=clearlydefined
   * CLEARLY_DEFINED_CURATION_GITHUB_REPO=curated-data
   * CLEARLY_DEFINED_CURATION_GITHUB_BRANCH=master
   * CLEARLY_DEFINED_CURATION_GITHUB_TOKEN=[personal access token with public_repo scope]
1. `npm install && npm start`

## System Flow
1. Scan plugin checks if it has already harvested data for a package by calling GET /harvest/...
1. If it's already harvested then it stops processing
1. If not it performs the scan and uploads the resulting files by calling PUT /harvest/...
1. User visits the site and looks up information about a package which calls GET /package/...
   1. This gets the harvested data for a package
   1. It then normalized the harvested data to the normalized schema
   1. It then runs the summarizer to condense the normalized schemas into a single normalized schema for the package
   1. It then loads any curation patch that applies to the package and patches the normalized schema and returns the end result
1. They notice an error and edit a patch, a preview of the result of applying the patch is displayed by calling POST /package/.../preview with the proposed patch
1. They submit the patch which calls PATCH /curations/...
1. A pull request is initiated and a build process runs against the patch
1. The build gets the normalized schema for each of the patches in the pull request by calling GET /package/... and also a preview of the result by calling POST /package/.../preview and puts a diff in the PR for a curator to review
1. A curator reviews the diff and if they're happy with the end result merges the PR
1. As an optimization post merge we could normalize, summarize, and patch the affected package and store the result, if we did this then GET /package/... would simply read that cache rather than doing the work on the fly

## Endpoints
### Resolved
TODO

### Curation
#### PATCH /curations/:packageFormat/:provider/:packageName/:packageRevision

##### Request Body
```
{
  "source_location": {
    "provider": "",
    "url": "",
    "revision": "",
    "path": ""
  },
  "copyright": {
    "statements": [],
    "holders": [],
    "authors": []
  },
  "license": {
    "expression": ""
  }
}
```

##### Description
As a PATCH you only need to provide the attributes you want to add or update, any attributes not included will be ignored. To explicitly remove an attribute set its value to `null`.

TODO: Make sure the attribute names are consistent with AboutCode/ScanCode
TODO: Include a section where the author's identity and reasoning is provided

### Harvested
TODO

## Storage
### Curation
Curation patches will be stored in:
https://github.com/clearlydefined/curated-data

#### Structure
```
packageFormat (npm)
  provider (npmjs.org)
    packageName.yaml (redie)
```

Note that the package name may contain a namespace portion, if it does, then the namespace will become a directory under provider and the packageName.yaml will be stored in the namespace directory. For example, a scoped NPM package would have a directory for the scope under provider, and then the packageName.yaml would be in the scope directory. Similarly, for Maven, the groupId would be the namespace, and the artifactId would be the packageName.

```
packageFormat (git)
  provider (github.com)
    namespace (Microsoft)
      packageName.yaml (redie)
```

#### Format
TODO

### Harvested
Harvested data will be stored in:
https://github.com/clearlydefined/harvested-data

This location is temporary, as harvested data grows will likely need to move it out of GitHub to scale.

#### Structure
```
packageFormat
  provider
    packageName
      packageRevision
        configurationName - configurationNames always include tool name "ScanCode-2.1"
          [tool output files]
```

#### Raw Notes
What term should we use instead of package?
* AboutCode says package
* Concerns that "native" source consumers don't consider what they consume as a package
* Defer decision :)

What to name output files?
1. If a single file, then output.ext (e.g. output.json)
2. If multiple files, then keep their native names.

How to handle different versions of scanners?
* Results are immutable
* Curations are tied to particular "tool configurations"
* Tool configurations are immutable
* Tool configuration and revision should be captured in the output directory

Do we merge results from different versions of ScanCode? How does this impact curation?
* New scan results flag previous curations as needing review (optimization: only if they change the result)
* The summarization process will be configured as to what tool configurations to consider for summarization (this would need to be a priority list)
* The summarization process should take into account the link from the package back to the source

Scanning a package where it's actually the source you need to scan, what to store where
Maven supports scanning sources JAR or scanning GitHub repository
* If we can determine the source location from the package, then we'll queue up the source to be scanned
* If we can't determine the source location, and the package contains "source" then we'll scan the package
* Some scanners will run on packages (such as things that inspect package manifest)
* We should track the link from the package to the source

How to handle tags?
* When we scan GitHub we need to track what tags/branches point at which commits
* We will use the long commit (40 character) reference

Need to define "origin" and/or pick another term
* Propose to use "provider"

How do we handle case sensitivity?
* If no package managers allow different packages with same name different case, then we can be case preserving, case insensitive
* We need to be case preserving because some registry endpoints are case sensitive (e.g. NPM)

Define how to do the linking
* We will store one end (package to source), we will cache/materialize the reverse link as part of a build job (or the like)

#### Format
The format of harvested data is tool-specific. Tool output is stored in the tool's native output format. If there is a choice between multiple output formats then the priorities are:
1. Machine-readable
1. Lossless
1. JSON

## Package Format Registry
* git
* maven
* npm
* nuget
* rubygem

## Origin Registry
* central.maven.org
* github.com
* npmjs.org
* nuget.org

## Tool Name Registry
* ScanCode
* Fossology

## Terminology
* provider - the provider of metadata (e.g. npmjs.org, github.com)
* revision - used instead of version because it's a superset and valid for source

## TODO
* Swagger to replace most of this doc
* Complete registries
* Complete terminology

## Running ORT for scanning
Build and run the container.

```
docker build -t ort .
docker run --mount type=bind,source="<path to repo>",target=/app ort scanner -d /app/output/package-json-dependencies.yml -o /app/output-scanner
```
