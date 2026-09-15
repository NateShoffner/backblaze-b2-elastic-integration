{{- generatedHeader }}
# Backblaze B2 Integration for Elastic

## Overview

The Backblaze B2 integration for Elastic collects bucket access logs, daily usage reports, and bucket inventory from [Backblaze B2 Cloud Storage](https://www.backblaze.com/cloud-storage). Use it to see which application keys are reading, writing, and deleting objects in your buckets, and to track storage, downloads, API transactions, and estimated costs over time.

### Compatibility

This integration uses the Backblaze B2 S3-compatible API and the B2 Native API.

### How it works

- `access`: Backblaze B2 writes bucket access logs as objects to a log destination bucket. Elastic Agent polls that bucket with the AWS S3 input through the B2 S3-compatible endpoint.
- `usage`: Backblaze B2 writes daily usage report CSV files to the `b2-reports-<accountId>` bucket. Elastic Agent polls that bucket the same way.
- `bucket`: Elastic Agent polls the B2 Native API with the CEL input.

## What data does this integration collect?

The Backblaze B2 integration collects log messages of the following types:
* `access`: One event per request made against a monitored bucket, including uploads, downloads, deletes, listings, and lifecycle actions, with the application key that made the request.
* `usage`: One event per bucket per day, plus one account-level event, with storage, upload, download, and API transaction totals from Backblaze usage reports.
* `bucket`: Bucket configuration snapshots, including bucket type, encryption, Object Lock, lifecycle rules, and replication settings.

### Supported use cases

- Audit which application keys access or delete backup data.
- Track storage growth and download volume per bucket.
- Estimate B2 costs from daily usage.
- Detect bucket configuration changes, such as Object Lock or lifecycle rule edits.

## What do I need to use this integration?

- A Backblaze B2 account with Bucket Access Logs enabled on the buckets you want to monitor.
- Usage reports enabled on the account. Standalone accounts must ask Backblaze Support to enable them.
- Two B2 application keys, so that no single key can both list every bucket and read backup data:
  - A key restricted to the log destination and usage reports buckets that can list and read files, for access logs and usage reports. The master application key can't be used with the S3-compatible API.
  - A key with no bucket restriction and only the `listBuckets`, `readBucketEncryption`, `readBucketRetentions`, and `readBucketReplications` capabilities, for bucket inventory. Without the read capabilities, encryption, Object Lock, and replication settings are reported as not readable. None of these capabilities can read file contents.

## How do I deploy this integration?

### Agent-based deployment

Elastic Agent must be installed. For more details, check the Elastic Agent [installation instructions](https://www.elastic.co/guide/en/fleet/current/elastic-agent-installation.html). You can install only one Elastic Agent per host.

Elastic Agent is required to read data from Backblaze B2 and ship the data to Elastic, where the events will then be processed via the integration's ingest pipelines.

### Set up steps in Backblaze B2

1. Create a bucket to receive access logs in the same region as the buckets you want to monitor. The log destination bucket can't have Object Lock enabled. Consider adding a lifecycle rule to delete old log files.
2. Enable Bucket Access Logs on each bucket you want to monitor, and set the destination to the bucket from step 1.
3. Ask Backblaze Support to enable usage reports for your account. Reports are written to the `b2-reports-<accountId>` bucket.
4. Create an application key restricted to the log destination and usage reports buckets, with read access to their files. If you plan to collect bucket inventory, create a second application key with no bucket restriction and the `listBuckets`, `readBucketEncryption`, `readBucketRetentions`, and `readBucketReplications` capabilities.
5. Note the region from the S3 endpoint shown on the bucket details, for example `us-west-004` from `s3.us-west-004.backblazeb2.com`.

#### Vendor resources

- [Bucket Access Logs](https://www.backblaze.com/docs/cloud-storage-bucket-access-logs)
- [Usage reports](https://www.backblaze.com/docs/cloud-storage-use-partner-api-reports)
- [S3-compatible API](https://www.backblaze.com/apidocs/introduction-to-the-s3-compatible-api)

### Set up steps in Kibana

1. In Kibana, go to **Management > Integrations** and search for "Backblaze B2".
2. Click **Add Backblaze B2**.
3. Under the access logs and usage reports input, enter the **Application Key ID**, **Application Key**, and **Region** for the bucket-restricted key.
4. For bucket access logs, enter the **Log Destination Bucket Name** and the prefix configured for access logs, if any. Review **Operations to Drop**: by default, `B2_API.B2_LIST_FILE_NAMES` calls are dropped because backup tools generate large numbers of them. Remove the entry to keep them.
5. For usage reports, enter the **Usage Reports Bucket Name**.
6. Under the bucket inventory input, enter the **Application Key ID** and **Application Key** for the bucket listing key.
7. Save the integration and assign it to an agent policy.

### Validation

1. In Kibana, open **Discover**.
2. Filter on `data_stream.dataset` with one of `backblaze_b2.access`, `backblaze_b2.usage`, or `backblaze_b2.bucket`.
3. Confirm that events are arriving. Access logs can take several hours to appear after requests are made.

## Troubleshooting

- No access log events: Backblaze delivers access logs on a best-effort basis, usually within a few hours. Confirm that new log objects exist in the log destination bucket.
- Authentication errors from the S3 endpoint: The master application key isn't supported by the S3-compatible API. Use a standard application key.
- No usage events: Backblaze creates the `b2-reports-<accountId>` bucket only after the first reports are generated, which can take many hours after usage reports are enabled. It's a `restricted` bucket, so some tools that list buckets, such as rclone, don't show it. Confirm that the bucket exists and that the application key can read it.
- Usage events with `event.kind: pipeline_error`: The usage report format changed, for example Backblaze added a column. The raw row is kept in `event.original`. Update the integration to a version that supports the new format.
- Duplicate access log events: S3 polling mode doesn't coordinate between agents. Collect each bucket with only one Elastic Agent.

## Performance and scaling

Access logs and usage reports are collected in S3 bucket polling mode, which doesn't support distributing a bucket across multiple agents. Use the **Number of Workers** setting to control how many log objects are processed in parallel.

Bucket listing calls usually make up almost all access log lines when backup tools sync with the B2 Native API. The **Operations to Drop** setting filters them in Elastic Agent, before they reach Elasticsearch. Add other operations to the list to reduce volume further.

For more information on architectures that can be used for scaling this integration, check the [Ingest Architectures](https://www.elastic.co/docs/manage-data/ingest/ingest-reference-architectures) documentation.

## Reference

### Inputs used

- `aws-s3`, in S3 bucket polling mode with a non-AWS bucket name and the B2 S3-compatible endpoint: `access` and `usage` data streams. SQS mode, bucket ARNs, and session tokens aren't used.
- `cel`: `bucket` data stream, which calls `b2_authorize_account` and `b2_list_buckets`.

### API usage

These APIs are used with this integration:
* [`b2_authorize_account`](https://www.backblaze.com/apidocs/b2-authorize-account)
* [`b2_list_buckets`](https://www.backblaze.com/apidocs/b2-list-buckets)
* S3-compatible `ListObjectsV2` and `GetObject`

### Vendor documentation links

- [B2 Native API](https://www.backblaze.com/apidocs/introduction-to-the-b2-native-api)
- [B2 S3-compatible API](https://www.backblaze.com/apidocs/introduction-to-the-s3-compatible-api)
- [B2 rate limits](https://www.backblaze.com/docs/cloud-storage-rate-limits)

### Data streams

#### access

The `access` data stream provides bucket access log events from Backblaze B2.

##### access fields

{{ fields "access" }}

#### usage

The `usage` data stream provides daily per-bucket usage events from Backblaze B2 usage reports. Each report row becomes one event with `event.kind: metric`, timestamped at the start of the report day in UTC. `backblaze_b2.usage.scope` is `bucket` for bucket rows and `account` for the account-level row, which holds API transactions that aren't tied to a bucket. The account email in the report isn't indexed. Rows from the audit report files are dropped. If Backblaze regenerates a report file, identical rows aren't indexed twice within the same backing index.

##### usage fields

{{ fields "usage" }}

#### bucket

The `bucket` data stream provides bucket configuration snapshots from the B2 Native API.

##### bucket fields

{{ fields "bucket" }}

##### bucket sample event

{{ event "bucket" }}
