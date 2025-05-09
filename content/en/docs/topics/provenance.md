---
title: "Helm Provenance and Integrity"
description: "Describes how to verify the integrity and origin of a Chart."
aliases: ["/docs/provenance/"]
weight: 5
---

Helm has provenance tools which help chart users verify the integrity and origin
of a package. Using industry-standard tools based on PKI, GnuPG, and
well-respected package managers, Helm can generate and verify signature files.

## Overview

Integrity is established by comparing a chart to a provenance record. Provenance
records are stored in _provenance files_, which are stored alongside a packaged
chart. For example, if a chart is named `myapp-1.2.3.tgz`, its provenance file
will be `myapp-1.2.3.tgz.prov`.

Provenance files are generated at packaging time (`helm package --sign ...`),
and can be checked by multiple commands, notably `helm install --verify`.

## The Workflow

This section describes a potential workflow for using provenance data
effectively.

Prerequisites:

- A valid PGP keypair in a binary (not ASCII-armored) format
- The `helm` command line tool
- GnuPG command line tools (optional)
- Keybase command line tools (optional)

**NOTE:** If your PGP private key has a passphrase, you will be prompted to
enter that passphrase for any commands that support the `--sign` option.

Creating a new chart is the same as before:

```console
$ helm create mychart
Creating mychart
```

Once ready to package, add the `--sign` flag to `helm package`. Also, specify
the name under which the signing key is known and the keyring containing the
corresponding private key:

```console
$ helm package --sign --key 'John Smith' --keyring path/to/keyring.secret mychart
```

**Note:** The value of the `--key` argument must be a substring of the desired
key's `uid` (in the output of `gpg --list-keys`), for example the name or email.
**The fingerprint _cannot_ be used.**

**TIP:** for GnuPG users, your secret keyring is in `~/.gnupg/secring.gpg`. You
can use `gpg --list-secret-keys` to list the keys you have.

**Warning:**  the GnuPG v2 store your secret keyring using a new format `kbx` on
the default location  `~/.gnupg/pubring.kbx`. Please use the following command
to convert your keyring to the legacy gpg format:

```console
$ gpg --export >~/.gnupg/pubring.gpg
$ gpg --export-secret-keys >~/.gnupg/secring.gpg
```

At this point, you should see both `mychart-0.1.0.tgz` and
`mychart-0.1.0.tgz.prov`. Both files should eventually be uploaded to your
desired chart repository.

You can verify a chart using `helm verify`:

```console
$ helm verify mychart-0.1.0.tgz
```

A failed verification looks like this:

```console
$ helm verify topchart-0.1.0.tgz
Error: sha256 sum does not match for topchart-0.1.0.tgz: "sha256:1939fbf7c1023d2f6b865d137bbb600e0c42061c3235528b1e8c82f4450c12a7" != "sha256:5a391a90de56778dd3274e47d789a2c84e0e106e1a37ef8cfa51fd60ac9e623a"
```

To verify during an install, use the `--verify` flag.

```console
$ helm install --generate-name --verify mychart-0.1.0.tgz
```

If the keyring containing the public key associated with the signed chart is not
in the default location, you may need to point to the keyring with `--keyring
PATH` as in the `helm package` example.

If verification fails, the install will be aborted before the chart is even
rendered.

### Using Keybase.io credentials

The [Keybase.io](https://keybase.io) service makes it easy to establish a chain
of trust for a cryptographic identity. Keybase credentials can be used to sign
charts.

Prerequisites:

- A configured Keybase.io account
- GnuPG installed locally
- The `keybase` CLI installed locally

#### Signing packages

The first step is to import your keybase keys into your local GnuPG keyring:

```console
$ keybase pgp export -s | gpg --import
```

This will convert your Keybase key into the OpenPGP format, and then import it
locally into your `~/.gnupg/secring.gpg` file.

You can double check by running `gpg --list-secret-keys`.

```console
$ gpg --list-secret-keys
/Users/mattbutcher/.gnupg/secring.gpg
-------------------------------------
sec   2048R/1FC18762 2016-07-25
uid                  technosophos (keybase.io/technosophos) <technosophos@keybase.io>
ssb   2048R/D125E546 2016-07-25
```

Note that your secret key will have an identifier string:

```
technosophos (keybase.io/technosophos) <technosophos@keybase.io>
```

That is the full name of your key.

Next, you can package and sign a chart with `helm package`. Make sure you use at
least part of that name string in `--key`.

```console
$ helm package --sign --key technosophos --keyring ~/.gnupg/secring.gpg mychart
```

As a result, the `package` command should produce both a `.tgz` file and a
`.tgz.prov` file.

#### Verifying packages

You can also use a similar technique to verify a chart signed by someone else's
Keybase key. Say you want to verify a package signed by
`keybase.io/technosophos`. To do this, use the `keybase` tool:

```console
$ keybase follow technosophos
$ keybase pgp pull
```

The first command above tracks the user `technosophos`. Next `keybase pgp pull`
downloads the OpenPGP keys of all of the accounts you follow, placing them in
your GnuPG keyring (`~/.gnupg/pubring.gpg`).

At this point, you can now use `helm verify` or any of the commands with a
`--verify` flag:

```console
$ helm verify somechart-1.2.3.tgz
```

### Reasons a chart may not verify

These are common reasons for failure.

- The `.prov` file is missing or corrupt. This indicates that something is
  misconfigured or that the original maintainer did not create a provenance
  file.
- The key used to sign the file is not in your keyring. This indicate that the
  entity who signed the chart is not someone you've already signaled that you
  trust.
- The verification of the `.prov` file failed. This indicates that something is
  wrong with either the chart or the provenance data.
- The file hashes in the provenance file do not match the hash of the archive
  file. This indicates that the archive has been tampered with.

If a verification fails, there is reason to distrust the package.

## The Provenance File

The provenance file contains a chart’s YAML file plus several pieces of
verification information. Provenance files are designed to be automatically
generated.

The following pieces of provenance data are added:

* The chart file (`Chart.yaml`) is included to give both humans and tools an
  easy view into the contents of the chart.
* The signature (SHA256, just like Docker) of the chart package (the `.tgz`
  file) is included, and may be used to verify the integrity of the chart
  package.
* The entire body is signed using the algorithm used by OpenPGP (see
  [Keybase.io](https://keybase.io) for an emerging way of making crypto
  signing and verification easy).

The combination of this gives users the following assurances:

* The package itself has not been tampered with (checksum package `.tgz`).
* The entity who released this package is known (via the GnuPG/PGP signature).

The format of the file looks something like this:

```
Hash: SHA512

apiVersion: v2
appVersion: "1.16.0"
description: Sample chart
name: mychart
type: application
version: 0.1.0

...
files:
  mychart-0.1.0.tgz: sha256:d31d2f08b885ec696c37c7f7ef106709aaf5e8575b6d3dc5d52112ed29a9cb92
-----BEGIN PGP SIGNATURE-----

wsBcBAEBCgAQBQJdy0ReCRCEO7+YH8GHYgAAfhUIADx3pHHLLINv0MFkiEYpX/Kd
nvHFBNps7hXqSocsg0a9Fi1LRAc3OpVh3knjPfHNGOy8+xOdhbqpdnB+5ty8YopI
mYMWp6cP/Mwpkt7/gP1ecWFMevicbaFH5AmJCBihBaKJE4R1IX49/wTIaLKiWkv2
cR64bmZruQPSW83UTNULtdD7kuTZXeAdTMjAK0NECsCz9/eK5AFggP4CDf7r2zNi
hZsNrzloIlBZlGGns6mUOTO42J/+JojnOLIhI3Psd0HBD2bTlsm/rSfty4yZUs7D
qtgooNdohoyGSzR5oapd7fEvauRQswJxOA0m0V+u9/eyLR0+JcYB8Udi1prnWf8=
=aHfz
-----END PGP SIGNATURE-----
```

Note that the YAML section contains two documents (separated by `...\n`). The
first file is the content of `Chart.yaml`. The second is the checksums, a map of
filenames to SHA-256 digests of that file's content at packaging time.

The signature block is a standard PGP signature, which provides [tamper
resistance](https://www.rossde.com/PGP/pgp_signatures.html).

## Chart Repositories

Chart repositories serve as a centralized collection of Helm charts.

Chart repositories must make it possible to serve provenance files over HTTP via
a specific request, and must make them available at the same URI path as the
chart.

For example, if the base URL for a package is
`https://example.com/charts/mychart-1.2.3.tgz`, the provenance file, if it
exists, MUST be accessible at
`https://example.com/charts/mychart-1.2.3.tgz.prov`.

From the end user's perspective, `helm install --verify myrepo/mychart-1.2.3`
should result in the download of both the chart and the provenance file with no
additional user configuration or action.

### Signatures in OCI-based registries

When publishing charts to an [OCI-based registry]({{< ref "registries.md" >}}), the
[`helm-sigstore` plugin](https://github.com/sigstore/helm-sigstore/) can be used 
to publish provenance to [sigstore](https://sigstore.dev/).  [As described in the
documentation](https://github.com/sigstore/helm-sigstore/blob/main/USAGE.md), the
process of creating provenance and signing with a GPG key are common, but the
`helm sigstore upload` command can be used to publish the provenance to an
immutable transparency log.

## Establishing Authority and Authenticity

When dealing with chain-of-trust systems, it is important to be able to
establish the authority of a signer. Or, to put this plainly, the system above
hinges on the fact that you trust the person who signed the chart. That, in
turn, means you need to trust the public key of the signer.

One of the design decisions with Helm has been that the Helm project would not
insert itself into the chain of trust as a necessary party. We don't want to be
"the certificate authority" for all chart signers. Instead, we strongly favor a
decentralized model, which is part of the reason we chose OpenPGP as our
foundational technology. So when it comes to establishing authority, we have
left this step more-or-less undefined in Helm 2 (a decision carried forward in
Helm 3).

However, we have some pointers and recommendations for those interested in using
the provenance system:

- The [Keybase](https://keybase.io) platform provides a public centralized
  repository for trust information.
  - You can use Keybase to store your keys or to get the public keys of others.
  - Keybase also has fabulous documentation available
  - While we haven't tested it, Keybase's "secure website" feature could be used
    to serve Helm charts.
  - The basic idea is that an official "chart reviewer" signs charts with her or
    his key, and the resulting provenance file is then uploaded to the chart
    repository.
  - There has been some work on the idea that a list of valid signing keys may
    be included in the `index.yaml` file of a repository.

