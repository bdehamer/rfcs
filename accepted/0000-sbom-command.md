# SBOM Generation for npm Projects

## Summary

Update the npm CLI with a new command which will generate a Software Bill of Materials (SBOM) for the current project. Users will have the option to generate an SBOM conforming to either the [Software Package Data Exchange (SPDX](https://spdx.github.io/spdx-spec/v2.3/)) or [CycloneDX](https://cyclonedx.org/specification/overview/) specifications.


## Motivation

Finding and remediating vulnerabilities in open source projects is a critical component of securing the software supply chain. However, this requires that enterprises understand what OSS components are used across their infrastructure and applications. When new vulnerabilities are discovered, they need a complete inventory of their software assets in order to properly assess their exposure. Knowing about the critical [Log4j](https://www.cisa.gov/news-events/cybersecurity-advisories/aa21-356a#main) vulnerability doesn’t do you any good unless you can also pinpoint where in your organization you’re running the vulnerable code.

SBOMs help to solve this problem by providing a standardized way to document the components that comprise a software application. A proper SBOM should tell you exactly which packages you have deployed and which versions of those packages you are using.

Beyond the security benefit there may be a regulatory requirement to provide SBOMs in some sectors. In response to some recent, high-visibility attacks, The White House has issued an [Executive Order](https://www.whitehouse.gov/briefing-room/presidential-actions/2021/05/12/executive-order-on-improving-the-nations-cybersecurity/) which specifically includes directives which would make SBOMs a requirement for any vendor selling to the federal government.

Adding SBOM generation to the tool which many developers are already using to manage their project dependencies eliminates any friction which may come from having to adopt/learn a separate tool. 


## Detailed Explanation

A new `sbom` command will be added to the npm CLI which will generate an SBOM for the current project. The SBOM will use the current project as the root and enumerate all of its dependencies (similar to the way `npm-ls` does) in one of the two supported SBOM formats. See the 
[Example SBOMs](#example-sboms) section for sample CycloneDX and SPDX SBOM documents.

Supported command options: 

`--sbom-format` - SBOM format to use for output. Valid values are “spdx” or “cyclonedx”.

`--omit` - Dependency types to omit from generated SBOM. Valid values are “dev”, “optional”, and “peer” (can be set multiple times). 

`--package-lock-only` - Constructs the SBOM based on the tree described by the _package-lock.json_, rather than the contents of _node_modules_. Defaults to _false_. If the _node_modules_ folder is not present, this flag will be required in order to generate an SBOM.
 
`--workspace` - Runs the command only in the context of the specified workspace (can be set multiple times).

`--workspaces` - Runs the command in the context of all configured workspaces. 

If the user runs the `sbom` command without first installing the dependencies for the project (i.e. there is no _node_modules_ folder present) an error will be displayed. An SBOM can be generated solely based on the contents of the _package-lock.json_ but requires the user to explicitly specify the `--package-lock-only` flag.


## Rationale and Alternatives

There are a few existing tools which can be used to generate an SBOM from an npm project:

* <code>[@cyclonedex/cyclonedx-npm](https://www.npmjs.com/package/@cyclonedx/cyclonedx-npm)</code> - A CLI for generating a CycloneDX-style SBOM from an npm project. This project is written in TypeScript and actually invokes <code>npm-ls</code> in order to get dependency information for the project.
* <code>[spdx-sbom-generator](https://github.com/opensbom-generator/spdx-sbom-generator)</code> - A CLI for generating SPDX-style SBOMs for a number of different package managers (including npm). Currently, this tool only works with npm projects using <em>lockfileVersion</em> 1 so it’s not viable for a large number of projects (current <em>lockfileVersion</em> is 3)

While you can effectively generate the same output we’re proposing with this combination of tools, there is value in having this capability supported directly in npm. Beyond the obvious developer-experience benefit of having SBOM generation baked-in to the CLI, it gives us a future path to do things like automatic-signing of SBOMs or integration of SBOMs into the package publishing process.


## Implementation

The `npm-sbom` command is similar in function to `npm-ls` command and will likely utilize a similar implementation. We’ll use <code>[arborist](https://github.com/npm/cli/tree/latest/workspaces/arborist)</code> to construct the dependency tree and the <code>[treeverse](https://github.com/isaacs/treeverse)</code> library to traverse the tree and assemble the SBOM.


### Local (Linked) Dependencies

The generated SBOM will exclude any dependencies which are referenced via links on the local file system. A warning will be displayed for any linked dependency informing the user that it was omitted from the SBOM.


### Format Details 

Both of the SBOM formats present a flat list of dependencies (CycloneDX groups these under a key named `components` while SPDX groups them under a key named `packages`). The following sections describe how a dependency will be presented for the different SBOM formats.


#### CycloneDX

```json
{
  "type": "library",
  "name": "debug",
  "version": "4.3.4",
  "bom-ref": "debug@4.3.4",
  "purl": "pkg:npm/debug@4.3.4",
  "properties": [
    {
      "name": "cdx:npm:package:path",
      "value": "node_modules/debug"
    }
  ],
  "hashes": [
    {
      "alg": "SHA-512",
      "content": "3d15851ee494dde0ed4093ef9cd63b..."
    }
  ]
}
```

Scoped packages will have a <code>[group](https://cyclonedx.org/docs/1.5/json/#components_items_group)</code> field which identifies just the scope portion of the package name. For example:

```json
{
  "type": "library",
  "name": "node14",
  "group": "@tsconfig",
  "version": "1.0.3",
  "bom-ref": "@tsconfig/node14@1.0.3"
}
```

The <code>[properties](https://cyclonedx.org/docs/1.5/json/#components_items_properties)</code> collection also provides for a standard property under the [npm taxonomy](https://github.com/CycloneDX/cyclonedx-property-taxonomy/blob/main/cdx/npm.md) for annotating development dependencies. For any package which was determined to be a development dependency of the root project, we would add the following to the <code>properties</code> collection: 

```json
{
  "name": "cdx:npm:package:development",
  "value": true
}
```

#### SPDX

```json
{
  "name": "debug",
  "SPDXID": "SPDXRef-Package-debug-4.3.4",
  "versionInfo": "4.3.4",
  "downloadLocation": "https://registry.npmjs.org/debug/-/debug-4.3.4.tgz",
  "filesAnalyzed": false,
  "externalRefs": [
    {
      "referenceCategory": "PACKAGE-MANAGER",
      "referernceType": "npm",
      "referenceLocator": "debug@4.3.4"
    },
    {
      "referenceCategory": "PACKAGE-MANAGER",
      "referernceType": "purl",
      "referenceLocator": "pkg:npm/debug@4.3.4"
    }
  ],
  "checksums": [
    {
      "algorithm": "SHA512",
      "checksumValue": "3d15851ee494dde0ed4093ef9cd63b..."
    }
  ]
}
```

The example record shown above conforms to version 2.3 of the SPDX [Package](https://spdx.github.io/spdx-spec/v2.3/package-information/) specification.

The <code>[downloadLocation](https://spdx.github.io/spdx-spec/v2.3/package-information/#77-package-download-location-field)</code> field will report the resolved location calculated by Arborist. This may point to a package registry, or a git URL for dependencies referenced directly from git. The top-level project will specify a value of <code>NOASSERTION</code> for the download location as this information is not available.

All packages will specify a `false` value for the <code>[filesAnaylzed](https://spdx.github.io/spdx-spec/v2.3/package-information/#78-files-analyzed-field)</code> field to signal that no assertions are being made about the files contained within a package.

The <code>[externalRefs](https://spdx.github.io/spdx-spec/v2.3/package-information/#721-external-reference-field)</code> field will contain two <code>[PACKAGE-MANAGER](https://spdx.github.io/spdx-spec/v2.3/external-repository-identifiers/#f3-package-manager)</code> references, one using the <code>[npm](https://spdx.github.io/spdx-spec/v2.3/external-repository-identifiers/#f32-npm)</code> reference type and the other using the <code>[purl](https://spdx.github.io/spdx-spec/v2.3/external-repository-identifiers/#f35-purl)</code> reference type.


## Unresolved Questions and Bikeshedding

* There’s a case to be made that SBOM generation should just be a flag on the existing `npm-ls` command – an SBOM is really just a different format of the output already generated by `npm-ls` (in fact, the CycloneDX tool described above parses the output from `npm-ls` in order to generate its SBOM). However, many of the arguments supported by `npm-ls` don’t really make sense in the context of SBOM generation (package spec filtering, depth, etc…) so it feels like a bit of an awkward fit. \
 \
Making it a distinct command allows us to add SBOM-specific features in the future like a `--sign` flag which could be used to generate a signed SBOM. \
 \
_Recommendation: Add a distinct command for generating an SBOM._

* SPDX doesn’t provide a natural way to differentiate between the various npm dependency types like “dev”, “peer”, and “optional”. We might consider using something like the [package comment field](https://spdx.github.io/spdx-spec/v2.3/package-information/#720-package-comment-field) to capture this information. \
 \
 _Recommendation: Skip for the first version of this feature. Wait to see if there is demand for this information and/or if the specification evolves to account for this metadata._
 
* Does `npm-sbom` command have a notion of a “default” SBOM format? Do we give preference to one of CycloneDX/SPDX or do we remain totally neutral (possibly at the expense of DX)? \
 \
_Recommendation: Default to SPDX if no format is specified._

* Both CycloneDX and SPDX support multiple document formats (JSON, XML, Protocol Buffers, etc). Should we support output of multiple formats, or do we stick w/ JSON? \
 \
_Recommendation: Stick with JSON-only for the first version of this feature._


## Example SBOMs

The sections below show what the different SBOM formats would look like for a basic npm project with a handful of dependencies.

The _package.json_ file below describes a “hello-world” application with two direct dependencies: the `debug` package and the `@tsconfig/node14` package (which is listed as a development dependency). The `debug` package itself has a dependency on the `ms` package.

```json
{
  "name": "hello-world",
  "version": "1.0.0",
  "license": "Apache-2.0",
  "repository": {
    "type": "git",
    "url": "git+https://github.com/example/hello-world.git"
  },
  "dependencies": {
    "debug": "^4.3.0"
  },
  "devDependencies": {
    "@tsconfig/node14": "^14.1.0"
  }
}
```

The complete dependency tree for this project looks like this:

```
$ npm ls

hello-world@1.0.0 
├── @tsconfig/node14@14.1.0
└─┬ debug@4.3.4
  └── ms@2.1.2
```


### CycloneDX

The proposed CycloneDX SBOM generated for the project above would look like the following:

```json
{
  "$schema": "http://cyclonedx.org/schema/bom-1.5.schema.json",
  "bomFormat": "CycloneDX",
  "specVersion": "1.5",
  "serialNumber": "urn:uuid:1b4cd070-3f4c-4f63-965e-4ab302ad7b41",
  "version": 1,
  "metadata": {
    "timestamp": "2023-08-04T21:37:16.639Z",
    "tools": [
      {
        "vendor": "npm",
        "name": "cli",
        "version": "9.8.1"
      }
    ],
    "component": {
      "type": "application",
      "name": "hello-world",
      "version": "1.0.0",
      "bom-ref": "hello-world@1.0.0",
      "purl": "pkg:npm/hello-world@1.0.0",
      "properties": [
        {
          "name": "cdx:npm:package:path",
          "value": ""
        }
      ]
    }
  },
  "components": [
    {
      "type": "library",
      "name": "node14",
      "group": "@tsconfig",
      "version": "1.0.3",
      "bom-ref": "@tsconfig/node14@1.0.3",
      "purl": "pkg:npm/%40tsconfig/node14@1.0.3",
      "properties": [
        {
          "name": "cdx:npm:package:path",
          "value": "node_modules/@tsconfig/node14"
        },
        {
          "name": "cdx:npm:package:development",
          "value": true
        }
      ],
      "hashes": [
        {
          "alg": "SHA-512",
          "content": "cac4fc9a1762c562..."
        }
      ]
    },
    {
      "type": "library",
      "name": "debug",
      "version": "4.3.4",
      "bom-ref": "debug@4.3.4",
      "purl": "pkg:npm/debug@4.3.4",
      "properties": [
        {
          "name": "cdx:npm:package:path",
          "value": "node_modules/debug"
        }
      ],
      "hashes": [
        {
          "alg": "SHA-512",
          "content": "3d15851ee494dde0..."
        }
      ]
    },
    {
      "type": "library",
      "name": "ms",
      "version": "2.1.2",
      "bom-ref": "ms@2.1.2",
      "purl": "pkg:npm/ms@2.1.2",
      "properties": [
        {
          "name": "cdx:npm:package:path",
          "value": "node_modules/ms"
        }
      ],
      "hashes": [
        {
          "alg": "SHA-512",
          "content": "b0690fc7e56332d9..."
        }
      ]
    }
  ]
}
```

### SPDX

The proposed SPDX SBOM generated for the project above would look like the following:

```json
{
  "spdxVersion": "SPDX-2.3",
  "dataLicense": "CC0-1.0",
  "SPXID": "SPDXRef-DOCUMENT",
  "name": "hello-world@1.0.0",
  "documentNamespace": "http://spdx.org/spdxdocs/hello-world-1.0.0-<uuid>",
  "creationInfo": {
    "created": "2023-08-04T21:41:02.071Z",
    "creators": [
      "Tool: pkg:npm/cli@9.8.1"
    ]
  },
  "packages": [
    {
      "name": "hello-world",
      "SPDXID": "SPDXRef-Package-hello-world-1.0.0",
      "versionInfo": "1.0.0",
      "downloadLocation": "NOASSERTION",
      "filesAnalyzed": false,
      "externalRefs": [
        {
          "referenceCategory": "PACKAGE-MANAGER",
          "referernceType": "npm",
          "referenceLocator": "hello-world@1.0.0"
        },
        {
          "referenceCategory": "PACKAGE-MANAGER",
          "referernceType": "purl",
          "referenceLocator": "pkg:npm/hello-world@1.0.0"
        }
      ]
    },
    {
      "name": "@tsconfig/node14",
      "SPDXID": "SPDXRef-Package-tsconfig.node14-1.0.3",
      "versionInfo": "1.0.3",
      "downloadLocation": "https://registry.npmjs.org/@tsconfig/node14/...",
      "filesAnalyzed": false,
      "externalRefs": [
        {
          "referenceCategory": "PACKAGE-MANAGER",
          "referernceType": "npm",
          "referenceLocator": "@tsconfig/node14@1.0.3"
        },
        {
          "referenceCategory": "PACKAGE-MANAGER",
          "referernceType": "purl",
          "referenceLocator": "pkg:npm/%40tsconfig/node14@1.0.3"
        }
      ],
      "checksums": [
        {
          "algorithm": "SHA512",
          "checksumValue": "cac4fc9a1762c562..."
        }
      ]
    },
    {
      "name": "debug",
      "SPDXID": "SPDXRef-Package-debug-4.3.4",
      "versionInfo": "4.3.4",
      "downloadLocation": "https://registry.npmjs.org/debug/-/debug-4.3.4.tgz",
      "filesAnalyzed": false,
      "externalRefs": [
        {
          "referenceCategory": "PACKAGE-MANAGER",
          "referernceType": "npm",
          "referenceLocator": "debug@4.3.4"
        },
        {
          "referenceCategory": "PACKAGE-MANAGER",
          "referernceType": "purl",
          "referenceLocator": "pkg:npm/debug@4.3.4"
        }
      ],
      "checksums": [
        {
          "algorithm": "SHA512",
          "checksumValue": "3d15851ee494dde0..."
        }
      ]
    },
    {
      "name": "ms",
      "SPDXID": "SPDXRef-Package-ms-2.1.2",
      "versionInfo": "2.1.2",
      "downloadLocation": "https://registry.npmjs.org/ms/-/ms-2.1.2.tgz",
      "filesAnalyzed": false,
      "externalRefs": [
        {
          "referenceCategory": "PACKAGE-MANAGER",
          "referernceType": "npm",
          "referenceLocator": "ms@2.1.2"
        },
        {
          "referenceCategory": "PACKAGE-MANAGER",
          "referernceType": "purl",
          "referenceLocator": "pkg:npm/ms@2.1.2"
        }
      ],
      "checksums": [
        {
          "algorithm": "SHA512",
          "checksumValue": "b0690fc7e56332d9..."
        }
      ]
    }
  ]
}
```

## References

* [NTIA Software Bill of Materials](https://ntia.gov/page/software-bill-materials)
* [OSSF - SBOM Everywhere SIG](https://github.com/ossf/sbom-everywhere)
* [Authoritative Guide to SBOM](https://cyclonedx.org/guides/sbom/OWASP_CycloneDX-SBOM-Guide-en.pdf)
* [SPDX Spec v2.3](https://spdx.github.io/spdx-spec/v2.3/)