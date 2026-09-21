# GCP provider

This contains the Pulumi code for provisioning GCP preemptible instances, configuring a ClickHouse cluster, running a configured query against S3, and immediately shutting them down.

The code aims to provision and destroy resources as quickly as possible (with the aim of minimizing costs) - improvements here are welcome.

Users may wish to experiment with different datasets and instance types to minimize query runtime AND/OR cost. We recommend consulting [Google Cloud Spot VM Pricing](https://cloud.google.com/spot-vms/pricing) for exploring instance costs.

See [ClickHouse and The One Trillion Row Challenge](https://clickhouse.com/blog/clickhouse-1-trillion-row-challenge) for an original blog with further details. Note: the blog post uses the AWS equivalent code found elsewhere in this repository.

## Dependencies

- [Pulumi](https://www.pulumi.com/docs/install/) >= v3.107.0
- [gcloud CLI](https://cloud.google.com/sdk/docs/install) configured with appropriate credentials

## Configuration

Currently single configuration (but we may create more stacks in the future).

`Pulumi.dev.yaml`
```yaml
config:
  gcp:region: us-east4
  gcp:project: <your-gcp-project-id>
  1trc:zone: us-east4-a
  1trc:instance_type: "c4a-highcpu-72"
  1trc:number_instances: 5
  # change as required
  1trc:cluster_password: "clickhouse_admin"
  # Ubuntu Minimal 24.04 LTS ARM64 image
  1trc:image: "ubuntu-minimal-2404-lts-arm64"
  # modify for your query
  1trc:query: "SELECT station, min(measure), max(measure), round(avg(measure), 2) FROM s3Cluster('default','https://coiled-datasets-rp.s3.us-east-1.amazonaws.com/1trc/measurements-*.parquet', '<AWS_ACCESS_KEY_ID>', '<AWS_SECRET_ACCESS_KEY>', headers('x-amz-request-payer' = 'requester')) GROUP BY station ORDER BY station ASC SETTINGS max_download_buffer_size = 52428800, max_threads=128"
```

By default, this queries a trillion row dataset `https://coiled-datasets-rp.s3.us-east-1.amazonaws.com/1trc/measurements-*.parquet` of weather measurements, computing a min, max and avg per station. This data is located in `us-east-1` and requires the requester pay.

To achieve this, it:

- Deploys infrastructure to `us-east4` (Northern Virginia) in Google Cloud to minimize latency to AWS `us-east-1`.
- Uses 5 * `c4a-highcpu-72` (ARM) preemptible instances in `us-east4-a` as these were the fastest per vCPU during initial testing.
- Requires AWS credentials to be configured in the query for S3 access. This will be used to pay, as the requestor, for the data transfer out of S3.
- Uses the Ubuntu Minimal 24.04 LTS ARM64 image.

Users can either add stacks or change the above configuration.

**Important: Ensure you replace `<your-gcp-project-id>` with your actual GCP project ID in `Pulumi.dev.yaml`**

**Important: Ensure you modify the `1trc:query` to include the `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY` with which you want to query**

The default configuration requires a `C4A` quota of at least 360 vCPUs in the region. Compute is short-lived and inexpensive; the main cost is the requester-pays S3 data transfer, which is billed to the AWS account whose credentials you supply.

## Deploying

```bash
pulumi stack select dev
./run.sh
```

This utility script performs a `pulumi up` followed by a `pulumi down`. If `pulumi up` fails it still runs `pulumi down`, so that instances are not left running.

## Implementation

Pulumi code deploys the following to the configured region and zone:

1. A VPC network with CIDR block `10.0.0.0/16`
2. A subnet with the above VPC for all instances
3. Firewall rules that allow:
   - Port 22 for SSH from the requester's IP address.
   - Port 8123 is also opened (HTTP) to allow queries to be run from the requester's IP address. Note: ClickHouse password is configurable.
   - All traffic between instances on all ports
   - All external outbound traffic is allowed.
   These loose security rules are permitted as the instances should be available for minutes, even on datasets with 1 trillion rows.
4. The requested number of preemptible (spot) instances with a 20GB `hyperdisk-balanced` disk.
5. A [custom resource provider](./query.py) handles the querying of ClickHouse once instances are deployed and the ClickHouse cluster has formed.

The code will generate configurations for the number of specified nodes under `./tmp`.
