# vexctl: A tool to make VEX work

[![Build Status](https://github.com/openvex/vexctl/actions/workflows/ci-build-test.yaml/badge.svg?branch=main)](https://github.com/openvex/vexctl/actions/workflows/ci-build-test.yaml?query=branch%3Amain)
[![Go Report Card](https://goreportcard.com/badge/github.com/openvex/vexctl)](https://goreportcard.com/report/github.com/openvex/vexctl)

`vexctl` is a tool to create, apply, and attest VEX (Vulnerability Exploitability
eXchange) data. Its purpose is to help with the creation and management of
VEX documents that allow "turning off" security scanner alerts of vulnerabilities
known not to affect a product.

VEX can be thought of as a "negative security advisory". Using VEX, software authors
can communicate to their users that an otherwise vulnerable component has no security
implications for their product.

## Installing

If you have Go 1.16 or later installed, you can run the following to install `vexctl`:
```console
go install github.com/openvex/vexctl@latest
```

## Operational Model

To achieve its mission, `vexctl` has three main modes of operation:

1. Creating VEX documents
2. Wrapping VEX documents in signed attestations
3. Applying the VEX data to scanner results

### 1. Creating VEX Documents

#### Creating New VEX Documents

VEX data can be created to a file on disk, or it can be captured in a
signed attestation that can be attached to a container image.

The easiest way to create a VEX document is using the `vexctl create` command:

```
vexctl create --product="pkg:apk/wolfi/git@2.38.1-r0?arch=x86_64" \
               --vuln="CVE-2014-123456" \
               --status="not_affected" \
               --justification="inline_mitigations_already_exist"
```


The previous invocations creates a VEX document with a single statement asserting
that the WolfiOS package `git-2.38.1-r0` is not affected by `CVE-2014-123456` because
it has already been mitigated in the distribution.

This is the resulting document:

```json
{
  "@context": "https://openvex.dev/ns",
  "@id": "https://openvex.dev/docs/public/vex-cfaef18d38537412a0307ec266bed56aa88fa58b7c1f2c6b8c9ef997028ba4bd",
  "author": "Unknown Author",
  "role": "Document Creator",
  "timestamp": "2023-01-10T20:24:50.498233798-06:00",
  "version": "1",
  "statements": [
    {
      "vulnerability": "CVE-2014-123456",
      "products": [
        "pkg:apk/wolfi/trivy@0.36.1-r0?arch=x86_64"
      ],
      "status": "not_affected",
      "justification": "component_not_present"
    }
  ]
}

```

vexctl can create VEX documents from three different sources:

1. From the command line, as shown
2. From a _golden file_ of predefined rules
3. From merging other VEX documents into a new one

The data is generated from a known rule set (the Golden Data) which is
reused and reapplied to new releases of the same project.

#### Merging Existing Documents

When more than one stakeholder is issuing VEX metadata about a piece of software,
vexctl can merge the documents to get the most up-to-date impact assessment of
a vulnerability. The following example can be run using the test documents found
in this repository:

```
vexctl merge --product=pkg:apk/wolfi/bash@1.0.0 \
             pkg/ctl/testdata/document1.vex.json \
             pkg/ctl/testdata/document2.vex.json
```
The resulting document combines the VEX statements that express data about
`bash@1.0.0` into a single document that tells the whole story of how `CVE-2014-123456`
was `under_investigation` and then `fixed` four hours later:

```json
{
  "@context": "https://openvex.dev/ns",
  "@id": "https://openvex.dev/docs/public/merged-vex-67124ea942ef30e1f42f3f2bf405fbbc4f5a56e6e87684fc5cd957212fa3e025",
  "author": "Unknown Author",
  "role": "Document Creator",
  "timestamp": "2023-01-10T20:36:55.524170935-06:00",
  "version": "1",
  "statements": [
    {
      "vulnerability": "CVE-2014-123456",
      "timestamp": "2022-12-22T16:36:43-05:00",
      "products": [
        "pkg:apk/wolfi/bash@1.0.0"
      ],
      "status": "under_investigation"
    },
    {
      "vulnerability": "CVE-2014-123456",
      "timestamp": "2022-12-22T20:56:05-05:00",
      "products": [
        "pkg:apk/wolfi/bash@1.0.0"
      ],
      "status": "fixed"
    }
  ]
}

```

### 2. Attesting Examples

```shell
# Attest and attach VEX statements in mydata.vex.json to a container image:
vexctl attest --attach --sign mydata.vex.json cgr.dev/image@sha256:e4cf37d568d195b4..
```

### 3. VEXing a Results Set

Using statements in a VEX document or from an attestation, `vexctl` will filter
security scanner results to remove _VEX'ed out_ entries.

#### Filtering Examples

```shell
# From a VEX file:
vexctl filter scan_results.sarif.json vex_data.csaf

# From a stored VEX attestation:
vexctl filter scan_results.sarif.json cgr.dev/image@sha256:e4cf37d568d195b4b5af4c36a...
```

The output from both examples will be the same: the SARIF result data, but
without the vulnerabilities that were stated as not exploitable:

```json
{
  "version": "2.1.0",
  "$schema": "https://json.schemastore.org/sarif-2.1.0-rtm.5.json",
  "runs": [
    {
      "tool": {
        "driver": {
          "fullName": "Trivy Vulnerability Scanner",
          "informationUri": "https://github.com/aquasecurity/trivy",
          "name": "Trivy",
          "rules": [

```

We support results files in SARIF for now. We plan to add support for the
proprietary formats of the most popular scanners.

### Multiple VEX Files

Assessing impact is process that takes time. VEX is designed to
communicate with users as time progresses. An example timeline may look like
this:

1. A project becomes aware of `CVE-2014-123456`, associated with one of its components.
2. Developers issue a VEX data file with a status of `under_investigation` to
inform their users they are aware of the CVE but are checking what impact it has.
3. After investigation, the developers determine the CVE has no impact
in their project because the vulnerable function in the component is never executed.
4. They issue a second VEX document with a status of `not_affected` and using
the `vulnerable_code_not_in_execute_path` justification.

`vexctl` will read all the documents in chronological order and "replay" the
known impacts statuses the order they were found, effectively computing the
`not_affected` status.

If a SARIF report is VEX'ed with `vexctl` any entries alerting of `CVE-2014-123456`
will be filtered out.

## Build vexctl

To build `vexctl`, clone this repository and run `make`.

```console
$ git clone https://github.com/openvex/vexctl.git
$ cd vex
$ make
$ ./vexctl version
 _   _  _____ __   __ _____  _____  _
| | | ||  ___|\ \ / //  __ \|_   _|| |
| | | || |__   \ V / | /  \/  | |  | |
| | | ||  __|  /   \ | |      | |  | |
\ \_/ /| |___ / /^\ \| \__/\  | |  | |____
 \___/ \____/ \/   \/ \____/  \_/  \_____/
vexctl: A tool for working with VEX data

GitVersion:    v0.1.0-21-g769ba3f-dirty
GitCommit:     769ba3f0c638003b6c5e3c41ae88f4cdc63555ab
GitTreeState:  dirty
BuildDate:     2023-01-18T00:19:24Z
GoVersion:     go1.19.4
Compiler:      gc
Platform:      darwin/arm64

```
