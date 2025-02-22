<p align="center">
    <img src="https://user-images.githubusercontent.com/5199289/136844524-1527b09f-c5cb-4aa9-be54-5aa92a6086c1.png" width="271" alt="Cute pink owl syft logo">
</p>

[![Validations](https://github.com/anchore/syft/actions/workflows/validations.yaml/badge.svg)](https://github.com/anchore/syft/actions/workflows/validations.yaml)
[![Go Report Card](https://goreportcard.com/badge/github.com/anchore/syft)](https://goreportcard.com/report/github.com/anchore/syft)
[![GitHub release](https://img.shields.io/github/release/anchore/syft.svg)](https://github.com/anchore/syft/releases/latest)
[![GitHub go.mod Go version](https://img.shields.io/github/go-mod/go-version/anchore/syft.svg)](https://github.com/anchore/syft)
[![License: Apache-2.0](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](https://github.com/anchore/syft/blob/main/LICENSE)
[![Slack Invite](https://img.shields.io/badge/Slack-Join-blue?logo=slack)](https://anchore.com/slack)

A CLI tool and Go library for generating a Software Bill of Materials (SBOM) from container images and filesystems. Exceptional for vulnerability detection when used with a scanner tool like [Grype](https://github.com/anchore/grype).

### Join our community meetings!

- Calendar: https://calendar.google.com/calendar/u/0/r?cid=Y182OTM4dGt0MjRtajI0NnNzOThiaGtnM29qNEBncm91cC5jYWxlbmRhci5nb29nbGUuY29t
- Agenda: https://docs.google.com/document/d/1ZtSAa6fj2a6KRWviTn3WoJm09edvrNUp4Iz_dOjjyY8/edit?usp=sharing (join [this group](https://groups.google.com/g/anchore-oss-community) for write access)
- All are welcome!

![syft-demo](https://user-images.githubusercontent.com/590471/90277200-2a253000-de33-11ea-893f-32c219eea11a.gif)

## Features
- Catalog container images and filesystems to discover packages and libraries.
- Supports packages and libraries from various ecosystems (APK, DEB, RPM, Ruby Bundles, Python Wheel/Egg/requirements.txt, JavaScript NPM/Yarn, Java JAR/EAR/WAR, Jenkins plugins JPI/HPI, Go modules)
- Linux distribution identification (supports Alpine, BusyBox, CentOS/RedHat, Debian/Ubuntu flavored distributions)
- Supports Docker and OCI image formats
- Direct support for [Grype](https://github.com/anchore/grype), a fast and powerful vulnerability matcher.


If you encounter an issue, please [let us know using the issue tracker](https://github.com/anchore/syft/issues).

## Installation

### Recommended
```bash
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b /usr/local/bin
```

...or, you can specify a release version and destination directory for the installation:

```
curl -sSfL https://raw.githubusercontent.com/anchore/syft/main/install.sh | sh -s -- -b <DESTINATION_DIR> <RELEASE_VERSION>
```

### Homebrew
```bash
brew tap anchore/syft
brew install syft
```

**Note**: Currently, Syft is built only for macOS and Linux.

## Getting started

To generate an SBOM for a Docker or OCI image:
```
syft <image>
```

**Note**: This is equivalent to specifying the `packages` subcommand:
```
syft packages <image>
```

The above output includes only software that is visible in the container (i.e., the squashed representation of the image).
To include software from all image layers in the SBOM, regardless of its presence in the final image, provide `--scope all-layers`:

```
syft packages <image> --scope all-layers
```

### Supported sources

Syft can generate a SBOM from a variety of sources:

```
# catalog a container image archive (from the result of `docker image save ...`, `podman save ...`, or `skopeo copy` commands)
syft packages path/to/image.tar

# catalog a directory
syft packages path/to/dir
```

Sources can be explicitly provided with a scheme:

```
docker:yourrepo/yourimage:tag          use images from the Docker daemon
docker-archive:path/to/yourimage.tar   use a tarball from disk for archives created from "docker save"
oci-archive:path/to/yourimage.tar      use a tarball from disk for OCI archives (from Skopeo or otherwise)
oci-dir:path/to/yourimage              read directly from a path on disk for OCI layout directories (from Skopeo or otherwise)
dir:path/to/yourproject                read directly from a path on disk (any directory)
file:path/to/yourproject/file          read directly from a path on disk (any single file)
registry:yourrepo/yourimage:tag        pull image directly from a registry (no container runtime required)
```

### Output formats

The output format for Syft is configurable as well:

```
syft packages <image> -o <format>
```

Where the `formats` available are:
- `json`: Use this to get as much information out of Syft as possible!
- `text`: A row-oriented, human-and-machine-friendly output.
- `cyclonedx`: A XML report conforming to the [CycloneDX 1.3 specification](https://cyclonedx.org/specification/overview/).
- `cyclonedx-json`: A JSON report conforming to the [CycloneDX 1.3 specification](https://cyclonedx.org/specification/overview/).
- `spdx`: A tag-value formatted report conforming to the [SPDX 2.2 specification](https://spdx.github.io/spdx-spec/).
- `spdx-json`: A JSON report conforming to the [SPDX 2.2 JSON Schema](https://github.com/spdx/spdx-spec/blob/v2.2/schemas/spdx-schema.json).
- `table`: A columnar summary (default).

## Private Registry Authentication

### Local Docker Credentials
When a container runtime is not present, Syft can still utilize credentials configured in common credential sources (such as `~/.docker/config.json`). 
It will pull images from private registries using these credentials. The config file is where your credentials are stored when authenticating with private registries via some command like `docker login`. 
For more information see the `go-containerregistry` [documentation](https://github.com/google/go-containerregistry/tree/main/pkg/authn).


An example `config.json` looks something like this:
```
// config.json
{
	"auths": {
		"registry.example.com": {
			"username": "AzureDiamond",
			"password": "hunter2"
		}
	}
}
```

You can run the following command as an example. It details the mount/environment configuration a container needs to access a private registry:

`docker run -v ./config.json:/config/config.json -e "DOCKER_CONFIG=/config" anchore/syft:latest  <private_image>`


### Docker Credentials in Kubernetes
The below section shows a simple workflow on how to mount this config file as a secret into a container on kubernetes.
1. Create a secret. The value of `config.json` is important. It refers to the specification detailed [here](https://github.com/google/go-containerregistry/tree/main/pkg/authn#the-config-file). 
Below this section is the `secret.yaml` file that the pod configuration will consume as a volume. 
The key `config.json` is important. It will end up being the name of the file when mounted into the pod.
    ```
    # secret.yaml
    
    apiVersion: v1
    kind: Secret
    metadata:
      name: registry-config
      namespace: syft
    data:
      config.json: <base64 encoded config.json>
    ```

    `kubectl apply -f secret.yaml`


2. Create your pod running syft. The env `DOCKER_CONFIG` is important because it advertises where to look for the credential file. 
In the below example, setting `DOCKER_CONFIG=/config` informs syft that credentials can be found at `/config/config.json`. 
This is why we used `config.json` as the key for our secret. When mounted into containers the secrets' key is used as the filename. 
The `volumeMounts` section mounts our secret to `/config`. The `volumes` section names our volume and leverages the secret we created in step one.
    ```
    # pod.yaml
    
    apiVersion: v1
    kind: Pod
    spec:
      containers:
        - image: anchore/syft:latest
          name: syft-private-registry-demo
          env:
            - name: DOCKER_CONFIG
              value: /config
          volumeMounts:
          - mountPath: /config
            name: registry-config
            readOnly: true
          args:
            - <private_image>
      volumes:
      - name: registry-config
        secret:
          secretName: registry-config
    ```

    `kubectl apply -f pod.yaml`


3. The user can now run `kubectl logs syft-private-registry-demo`. The logs should show the syft analysis for the `<private_image>` provided in the pod configuration.

Using the above information, users should be able to configure private registry access without having to do so in the `grype` or `syft` configuration files.
They will also not be dependent on a docker daemon, (or some other runtime software) for registry configuration and access.

## Configuration

Configuration search paths:

- `.syft.yaml`
- `.syft/config.yaml`
- `~/.syft.yaml`
- `<XDG_CONFIG_HOME>/syft/config.yaml`

Configuration options (example values are the default):

```yaml
# the output format of the SBOM report (options: table, text, json)
# same as -o ; SYFT_OUTPUT env var
output: "table"

# suppress all output (except for the SBOM report)
# same as -q ; SYFT_QUIET env var
quiet: false

# same as --file; write output report to a file (default is to write to stdout)
file: ""

# enable/disable checking for application updates on startup
# same as SYFT_CHECK_FOR_APP_UPDATE env var
check-for-app-update: true

# cataloging packages is exposed through the packages and power-user subcommands
package:
  cataloger:
    # enable/disable cataloging of packages
    # SYFT_PACKAGE_CATALOGER_ENABLED env var
    enabled: true

    # the search space to look for packages (options: all-layers, squashed)
    # same as -s ; SYFT_PACKAGE_CATALOGER_SCOPE env var
    scope: "squashed"

# cataloging file classifications is exposed through the power-user subcommand
file-classification:
  cataloger:
    # enable/disable cataloging of file classifications
    # SYFT_FILE_CLASSIFICATION_CATALOGER_ENABLED env var
    enabled: true

    # the search space to look for file classifications (options: all-layers, squashed)
    # SYFT_FILE_CLASSIFICATION_CATALOGER_SCOPE env var
    scope: "squashed"

# cataloging file contents is exposed through the power-user subcommand
file-contents:
  cataloger:
    # enable/disable cataloging of secrets
    # SYFT_FILE_CONTENTS_CATALOGER_ENABLED env var
    enabled: true

    # the search space to look for secrets (options: all-layers, squashed)
    # SYFT_FILE_CONTENTS_CATALOGER_SCOPE env var
    scope: "squashed"

  # skip searching a file entirely if it is above the given size (default = 1MB; unit = bytes)
  # SYFT_FILE_CONTENTS_SKIP_FILES_ABOVE_SIZE env var
  skip-files-above-size: 1048576

  # file globs for the cataloger to match on
  # SYFT_FILE_CONTENTS_GLOBS env var
  globs: []

# cataloging file metadata is exposed through the power-user subcommand
file-metadata:
  cataloger:
    # enable/disable cataloging of file metadata
    # SYFT_FILE_METADATA_CATALOGER_ENABLED env var
    enabled: true

    # the search space to look for file metadata (options: all-layers, squashed)
    # SYFT_FILE_METADATA_CATALOGER_SCOPE env var
    scope: "squashed"

  # the file digest algorithms to use when cataloging files (options: "sha256", "md5", "sha1")
  # SYFT_FILE_METADATA_DIGESTS env var
  digests: ["sha256"]

# cataloging secrets is exposed through the power-user subcommand
secrets:
  cataloger:
    # enable/disable cataloging of secrets
    # SYFT_SECRETS_CATALOGER_ENABLED env var
    enabled: true

    # the search space to look for secrets (options: all-layers, squashed)
    # SYFT_SECRETS_CATALOGER_SCOPE env var
    scope: "all-layers"

  # show extracted secret values in the final JSON report
  # SYFT_SECRETS_REVEAL_VALUES env var
  reveal-values: false

  # skip searching a file entirely if it is above the given size (default = 1MB; unit = bytes)
  # SYFT_SECRETS_SKIP_FILES_ABOVE_SIZE env var
  skip-files-above-size: 1048576

  # name-regex pairs to consider when searching files for secrets. Note: the regex must match single line patterns
  # but may also have OPTIONAL multiline capture groups. Regexes with a named capture group of "value" will
  # use the entire regex to match, but the secret value will be assumed to be entirely contained within the
  # "value" named capture group.
  additional-patterns: {}

  # names to exclude from the secrets search, valid values are: "aws-access-key", "aws-secret-key", "pem-private-key",
  # "docker-config-auth", and "generic-api-key". Note: this does not consider any names introduced in the
  # "secrets.additional-patterns" config option.
  # SYFT_SECRETS_EXCLUDE_PATTERN_NAMES env var
  exclude-pattern-names: []

# options when pulling directly from a registry via the "registry:" scheme
registry:
  # skip TLS verification when communicating with the registry
  # SYFT_REGISTRY_INSECURE_SKIP_TLS_VERIFY env var
  insecure-skip-tls-verify: false
  # use http instead of https when connecting to the registry
  # SYFT_REGISTRY_INSECURE_USE_HTTP env var
  insecure-use-http: false

  # credentials for specific registries
  auth:
    - # the URL to the registry (e.g. "docker.io", "localhost:5000", etc.)
      # SYFT_REGISTRY_AUTH_AUTHORITY env var
      authority: ""
      # SYFT_REGISTRY_AUTH_USERNAME env var
      username: ""
      # SYFT_REGISTRY_AUTH_PASSWORD env var
      password: ""
      # note: token and username/password are mutually exclusive
      # SYFT_REGISTRY_AUTH_TOKEN env var
      token: ""
    - ... # note, more credentials can be provided via config file only

log:
  # use structured logging
  # same as SYFT_LOG_STRUCTURED env var
  structured: false

  # the log level; note: detailed logging suppress the ETUI
  # same as SYFT_LOG_LEVEL env var
  level: "error"

  # location to write the log file (default is not to have a log file)
  # same as SYFT_LOG_FILE env var
  file: ""

# uploading package SBOM is exposed through the packages subcommand
anchore:
  # (feature-preview) the Anchore Enterprise Host or URL to upload results to (supported on Enterprise 3.0+)
  # same as -H ; SYFT_ANCHORE_HOST env var
  host: ""

  # (feature-preview) the path after the host to the Anchore External API (supported on Enterprise 3.0+)
  # same as SYFT_ANCHORE_PATH env var
  path: ""

  # (feature-preview) the username to authenticate against Anchore Enterprise (supported on Enterprise 3.0+)
  # same as -u ; SYFT_ANCHORE_USERNAME env var
  username: ""

  # (feature-preview) the password to authenticate against Anchore Enterprise (supported on Enterprise 3.0+)
  # same as -p ; SYFT_ANCHORE_PASSWORD env var
  password: ""

  # (feature-preview) path to dockerfile to be uploaded with the syft results to Anchore Enterprise (supported on Enterprise 3.0+)
  # same as -d ; SYFT_ANCHORE_DOCKERFILE env var
  dockerfile: ""

```
