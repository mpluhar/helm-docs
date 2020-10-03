helm-docs
=========
[![Go Report Card](https://goreportcard.com/badge/github.com/norwoodj/helm-docs)](https://goreportcard.com/report/github.com/norwoodj/helm-docs)

The helm-docs tool generates automatic documentation from helm charts into a markdown file. The resulting
file contains metadata about the chart and a table with all of your charts' values, their defaults, and an
optional description parsed from comments.

The markdown generation is entirely [gotemplate](https://golang.org/pkg/text/template) driven. The tool parses metadata
from charts and generates a number of sub-templates that can be referenced in a template file (by default `README.md.gotmpl`).
If no template file is provided, the tool has a default internal template that will generate a reasonably formatted README.

In particular, this tool will auto-detect descriptions of fields from comments:
```yaml
controller:
  # -- Configure the healthcheck for the ingress controller
  livenessProbe:
    httpGet:
      # -- This is the liveness check endpoint
      path: /healthz
```

Resulting in a resulting README section like so:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| controller.livenessProbe | object | `{"httpGet":{"path":"/healthz","port":8080}}` | Configure the healthcheck for the ingress controller |
| controller.livenessProbe.httpGet.path | string | `"/healthz"` | This is the liveness check endpoint |


## Installation
helm-docs can be installed using [homebrew](https://brew.sh/):

```bash
brew install norwoodj/tap/helm-docs
```

This will download and install the [latest release](https://github.com/norwoodj/helm-docs/releases/latest)
of the tool.

To build from source in this repository:

```bash
cd cmd/helm-docs
go build
```

Or install from source:

```bash
GO111MODULE=on go get github.com/norwoodj/helm-docs/cmd/helm-docs
```

## Usage

### Pre-commit hook

If you want to automatically generate `README.md` files with a pre-commit hook, make sure you
[install the pre-commit binary](https://pre-commit.com/#install), and add a [.pre-commit-config.yaml file](./.pre-commit-config.yaml)
to your project. Then run:

```bash
pre-commit install
pre-commit install-hooks
```

Future changes to your chart's `requirements.yaml`, `values.yaml`, `Chart.yaml`, or `README.md.gotmpl` files will cause an update to documentation when you commit.

### Running the binary directly

To run and generate documentation into READMEs for all helm charts within or recursively contained by a directory:

```bash
helm-docs
# OR
helm-docs --dry-run # prints generated documentation to stdout rather than modifying READMEs
```

The tool searches recursively through subdirectories of the current directory for `Chart.yaml` files and generates documentation
for every chart that it finds.

### Using docker

You can mount a directory with charts under `/helm-docs` within the container.

Then run:

```bash
docker run --rm --volume "$(pwd):/helm-docs" -u $(id -u) jnorwood/helm-docs:latest
```

### Building from source
Notice that you need to have build chain toolkit for given platform and golang installed.
Next you may need to export some vars to build standalone, non-linked binary for given platform and architecture:

```bash
export CGO_ENABLED=0 GOOS=linux GOARCH=amd64
make clean
make fmt
make test
make
```

## Ignoring Chart Directories
helm-docs supports a `.helmdocsignore` file, exactly like a `.gitignore` file in which one can specify directories to ignore
when searching for charts. Directories specified need not be charts themselves, so parent directories containing potentially
many charts can be ignored and none of the charts underneath them will be processed. You may also directly reference the
Chart.yaml file for a chart to skip processing for it.


## Markdown Rendering
The helm-docs tool uses go-templates to render output documentation. There are a number of sub-templates that are
referenced in the internal default template and can be used by custom `README.md.gotmpl` templates your repository provides as well.

Templates can be invoked like so:
```
{{ template "template-name" . }}
```

And the complete listing of available templates is below:

| Name | Description |
|------|-------------|
| chart.header              | The main heading of the generated markdown file |
| chart.name                | The _name_ field from the chart's `Chart.yaml` file |
| chart.deprecationWarning  | A deprecation warning which is displayed when the _deprecated_ field from the chart's `Chart.yaml` file is `true` |
| chart.description         | A description line containing the _description_ field from the chart's `Chart.yaml` file, or "" if that field is not set |
| chart.version             | The _version_ field from the chart's `Chart.yaml` file |
| chart.versionBadge        | A badge stating the current version of the chart |
| chart.type                | The _type_ field from the chart's `Chart.yaml` file |
| chart.typeBadge           | A badge stating the current type of the chart |
| chart.appVersion          | The _appVersion_ field from the chart's `Chart.yaml` file |
| chart.appVersionBadge     | A badge stating the current appVersion of the chart |
| chart.homepage            | The _home_ link from the chart's `Chart.yaml` file, or "" if that field is not set |
| chart.homepageLine        | A text line stating the current homepage of the chart |
| chart.maintainersHeader   | The heading for the chart maintainers section |
| chart.maintainersTable    | A table of the chart's maintainers |
| chart.maintainersSection  | A section headed by the maintainersHeader from above containing the maintainersTable from above or "" if there are no maintainers |
| chart.sourcesHeader       | The heading for the chart sources section |
| chart.sourcesList         | A list of the chart's sources |
| chart.sourcesSection      | A section headed by the sourcesHeader from above containing the sourcesList from above or "" if there are no sources |
| chart.kubeVersion         | The _kubeVersion_ field from the chart's `Chart.yaml` file |
| chart.kubeVersionLine     | A text line stating the required Kubernetes version for the chart |~~~~
| chart.requirementsHeader  | The heading for the chart requirements section |
| chart.requirementsTable   | A table of the chart's required sub-charts |
| chart.requirementsSection | A section headed by the requirementsHeader from above containing the kubeVersionLine and/or the requirementsTable from above or "" if there are no requirements |
| chart.valuesHeader        | The heading for the chart values section |
| chart.valuesTable         | A table of the chart's values parsed from the `values.yaml` file (see below) |
| chart.valuesSection       | A section headed by the valuesHeader from above containing the valuesTable from above or "" if there are no values |

For an example of how these various templates can be used in a `README.md.gotmpl` file to generate a reasonable markdown file,
look at the charts in [example-charts](./example-charts).

If there is no `README.md.gotmpl` (or other specified gotmpl file) present, the default template is used to generate the README.
That template looks like so:
```
{{ template "chart.header" . }}
{{ template "chart.deprecationWarning" . }}

{{ template "chart.versionBadge" . }}{{ template "chart.typeBadge" . }}{{ template "chart.appVersionBadge" . }}

{{ template "chart.description" . }}

{{ template "chart.homepageLine" . }}

{{ template "chart.maintainersSection" . }}

{{ template "chart.sourcesSection" . }}

{{ template "chart.requirementsSection" . }}

{{ template "chart.valuesSection" . }}
```

The tool includes the [sprig templating library](https://github.com/Masterminds/sprig), so those functions can be used
in the templates you supply.


### values.yaml metadata
This tool can parse descriptions and defaults of values from `values.yaml` files. The defaults are pulled directly from
the yaml in the file. 

It was formerly the case that descriptions had to be specified with the full path of the yaml field. This is no longer
the case, although it is still supported. Where before you would document a values.yaml like so:

```yaml
controller:
  publishService:
    # controller.publishService.enabled -- Whether to expose the ingress controller to the public world
    enabled: false

  # controller.replicas -- Number of nginx-ingress pods to load balance between.
  # Do not set this below 2.
  replicas: 2
```

You may now equivelantly write:
```yaml
controller:
  publishService:
    # -- Whether to expose the ingress controller to the public world
    enabled: false

  # -- Number of nginx-ingress pods to load balance between.
  # Do not set this below 2.
  replicas: 2
```

New-style comments are much the same as the old-style comments, except that while old comments for a field could appear
anywhere in the file, new-style comments must appear **on the line(s) immediately preceding the field being documented.**

I invite you to check out the [example-charts](./example-charts) to see how this is done in practice. The `but-auto-comments`
examples in particular document the new comment format.

Note that comments can continue on the next line. In that case leave out the double dash, and the lines will simply be
appended with a space in-between, as in the `controller.replicas` field in the example above

The following rules are used to determine which values will be added to the values table in the README:

* By default, only _leaf nodes_, that is, fields of type `int`, `string`, `float`, `bool`, empty lists, and empty maps
  are added as rows in the values table. These fields will be added even if they do not have a description comment
* Lists and maps which contain elements will not be added as rows in the values table _unless_ they have a description
  comment which refers to them
* Adding a description comment for a non-empty list or map in this way makes it so that leaf nodes underneath the
  described field will _not_ be automatically added to the values table. In order to document both a non-empty list/map
  _and_ a leaf node within that field, description comments must be added for both

e.g. In this case, both `controller.livenessProbe` and `controller.livenessProbe.httpGet.path` will be added as rows in
the values table, but `controller.livenessProbe.httpGet.port` will not
```yaml
controller:
  # -- Configure the healthcheck for the ingress controller
  livenessProbe:
    httpGet:
      # -- This is the liveness check endpoint
      path: /healthz
      port: http
```

Results in:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| controller.livenessProbe | object | `{"httpGet":{"path":"/healthz","port":8080}}` | Configure the healthcheck for the ingress controller |
| controller.livenessProbe.httpGet.path | string | `"/healthz"` | This is the liveness check endpoint |

If we remove the comment for `controller.livenessProbe` however, both leaf nodes `controller.livenessProbe.httpGet.path`
and `controller.livenessProbe.httpGet.port` will be added to the table, with our without description comments:

```yaml
controller:
  livenessProbe:
    httpGet:
      # -- This is the liveness check endpoint
      path: /healthz
      port: http
```

Results in:

| Key | Type | Default | Description |
|-----|------|---------|-------------|
| controller.livenessProbe.httpGet.path | string | `"/healthz"` | This is the liveness check endpoint |
| controller.livenessProbe.httpGet.port | string | `"http"` | |


### nil values
If you would like to define a key for a value, but leave the default empty, you can still specify a description for it
as well as a type. This is possible with both the old and the new comment format:
```yaml
controller:
  # -- (int) Number of nginx-ingress pods to load balance between
  replicas:
  
  # controller.image -- (string) Number of nginx-ingress pods to load balance between
  image:
```
This could be useful when wanting to enforce user-defined values for the chart, where there are no sensible defaults.

### Default values/column
In cases where you do not want to include the default value from `values.yaml`, or where the real default is calculated
inside the chart, you can change the contents of the column like so:

```yaml
service:
  # -- Add annotations to the service, this is going to be a long comment across multiple lines
  # but that's fine, these will be concatenated and the @default will be rendered as the default for this field
  # @default -- the chart will add some internal annotations automatically
  annotations: []
```

The order is important. The first comment line(s) must be the one specifying the key or using the auto-detection feature and
the description for the field. The `@default` comment must follow.

See [here](./example-charts/custom-template/values.yaml) for an example.

### Spaces and Dots in keys
In the old-style comment, if a key name contains any "." or " " characters, that section of the path must be quoted in
description comments e.g.

```yaml
service:
  annotations:
    # service.annotations."external-dns.alpha.kubernetes.io/hostname" -- Hostname to be assigned to the ELB for the service
    external-dns.alpha.kubernetes.io/hostname: stupidchess.jmn23.com

configMap:
  # configMap."not real config param" -- A completely fake config parameter for a useful example
  not real config param: value
```
