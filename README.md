# Backblaze B2 Integration for Elastic

[![CI](https://github.com/NateShoffner/backblaze-b2-elastic-integration/actions/workflows/ci.yml/badge.svg)](https://github.com/NateShoffner/backblaze-b2-elastic-integration/actions/workflows/ci.yml)

An Elastic Agent integration that collects bucket access logs (`access`), daily
usage reports (`usage`), and bucket configuration (`bucket`) from
[Backblaze B2 Cloud Storage](https://www.backblaze.com/cloud-storage).

Access logs and usage reports are read from B2 buckets with the AWS S3 input
pointed at the B2 S3-compatible endpoint. Bucket configuration is polled from the
B2 Native API with the CEL input. Requires **Kibana 9.4 or later**, because the
dashboards use ES|QL panels in the Kibana 9.4 dashboard format.

**[Read the integration documentation](packages/backblaze_b2/docs/README.md)**
for setup steps, configuration options, exported fields, and troubleshooting.

## Dashboards

Storage, transfer, and estimated cost, with a backup freshness table showing when
each bucket last received uploads:

![Storage usage and costs dashboard](docs/images/storage-usage-costs.png)

Uploads, downloads, and deletions, with the credentials and clients behind them:

![Bucket activity dashboard](docs/images/bucket-activity.png)

Bucket configuration, including public buckets and buckets without default
encryption or Object Lock:

![Bucket inventory dashboard](docs/images/bucket-inventory.png)

Screenshots use synthetic data.

## Installation

Download the `.zip` from the [latest release](https://github.com/NateShoffner/backblaze-b2-elastic-integration/releases/latest)
and upload it through **Kibana > Integrations > Upload integration**, or:

```sh
elastic-package install --zip backblaze_b2-<version>.zip
```

To build it yourself instead:

```sh
cd packages/backblaze_b2
elastic-package build     # writes build/packages/backblaze_b2-<version>.zip
elastic-package install
```

## Development

Requires [`elastic-package`](https://github.com/elastic/elastic-package) and
Docker. Run from `packages/backblaze_b2`:

```sh
elastic-package check            # lint and build
elastic-package stack up -d -v   # needed by the test commands below
elastic-package test pipeline    # ingest pipeline tests
elastic-package test policy      # agent policy rendering tests
elastic-package test system --data-streams bucket   # end-to-end against a mock B2 API
```

System tests cover the `bucket` data stream only. The `aws-s3` input has no
docker-based mock, so `access` and `usage` rely on pipeline tests.

Edit documentation in `_dev/build/docs/README.md`, never in `docs/README.md`,
which is regenerated on every build.

Dashboards live in `packages/backblaze_b2/kibana/dashboard/`. Edit them in Kibana
and export with `elastic-package export dashboards --id <id>`. The Kibana as-code
dashboards API rejects `links` panels, so updating a dashboard through that API
silently drops the navigation bar; re-add it to the exported file afterwards.

### Releasing

Bump `version` in `packages/backblaze_b2/manifest.yml`, add a matching
`changelog.yml` entry, then push a tag:

```sh
git tag v0.2.0 && git push origin v0.2.0
```

CI verifies the tag matches the manifest version, builds the package, and
publishes a GitHub release with the `.zip` attached and notes drawn from the
changelog entry.

## License

[Elastic License 2.0](LICENSE).

Backblaze and B2 are trademarks of Backblaze, Inc. The package icon is the
Backblaze logo from their published brand assets, used to identify the service.
This project is not affiliated with, endorsed by, or supported by Backblaze or
Elastic.
