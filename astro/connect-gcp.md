---
sidebar_label: 'GCP'
title: 'Create a network connection between Astro and GCP'
id: connect-gcp
description: Create a network connection to Google Cloud Platform.
sidebar_custom_props: { icon: 'img/gcp.png' }
---

import Tabs from '@theme/Tabs';
import TabItem from '@theme/TabItem';


Use this document to learn how you can grant an Astro cluster and its Deployments access to your external Google Cloud Platform (GCP) resources.

Publicly accessible endpoints allow you to quickly connect your Astro clusters or Deployments to GCP through an Airflow connection. If your cloud restricts IP addresses, you can add the external IPs of your Deployment or cluster to an GCP resource's allowlist.

If you have stricter security requirements, you can [create a private connection](#create-a-private-connection-between-astro-and-gcp) to GCP in a few different ways.

After you create a connection from your cluster to GCP, you might also need to individually authorize Deployments to access specific resources. See [Authorize your Deployment using workload identity](authorize-deployments-to-your-cloud.md).

## Standard and dedicated cluster support for GCP networking

Standard clusters have different connection options than dedicated clusters.

Standard clusters can connect to GCP in the following ways:

- Using [static external IP addresses](#allowlist-external-ip-addresses-for-a-cluster).
- Using Private Service Connect to all managed [Google APIs](https://cloud.google.com/vpc/docs/private-service-connect-compatibility#google-apis-global).

Dedicated clusters can use all of the same connection options as standard clusters. Additionally, they support a number of private connectivity options including:

- VPC peering

If you require a private connection between Astro and GCP, Astronomer recommends configuring a dedicated cluster. See [Create a dedicated cluster](create-dedicated-cluster.md).

## Access a public GCP endpoint

All Astro clusters include a set of external IP addresses that persist for the lifetime of the cluster. To facilitate communication between an Astro cluster and your cloud, you can allowlist these external IPs in your cloud. If you have no other security restrictions, this means that any cluster with an allowlisted external IP address can access your GCP resources through a valid Airflow connection.

### Allowlist external IP addresses for a cluster

1. In the Cloud UI, click your Workspace name in the upper left corner, then click **Organization Settings**.
2. Click **Clusters**, then select a cluster.
3. In the Details page, copy the IP addresses listed under **External IPs**.
4. Add the IP addresses to the allowlist of any external services that you want your cluster to access. You can also access these IP addresses from the **Details** page of any Deployment in the cluster.

After you allowlist a cluster's IP addresses, all Deployments in that cluster have network connectivity to GCP. When you use publicly accessible endpoints to connect to GCP, traffic moves directly between your Astro cluster and the GCP API endpoint. Data in this traffic never reaches the Astronomer managed control plane. Note that you still might also need to authorize your Deployment to some resources before it can access them.

## Create a private connection between Astro and GCP

Choose one of the following setups based on the security requirements of your company and your existing infrastructure.

<Tabs
    defaultValue="vpc"
    groupId="create-a-private-connection-between-astro-and-gcp"
    values={[
        {label: 'VPC peering', value: 'vpc'},
        {label: 'Private Service Connect', value: 'Private-Service-Connect'},
    ]}>

<TabItem value="vpc">

:::info

This connection option is available only for dedicated Astro Hosted clusters and Astro Hybrid.

:::

VPC peering ensures private and secure connectivity, reduces network transit costs, and simplifies network layouts.

To create a VPC peering connection between an Astro VPC and a GCP VPC:

1. Contact [Astronomer support](https://cloud.astronomer.io/open-support-request) and provide the following information:

    - Astro cluster ID and name.
    - Google Cloud project ID of the target VPC.
    - VPC NAME of the target VPC.
    - Classless Inter-Domain Routing (CIDR) block of the target VPC.

    After receiving your request, Astronomer support will create a VPC peering connection from your Astro VPC to your target VPC. The support team will then provide you with your Astro cluster GCP project ID and VPC name.

2. Using the information provided by Astronomer support, [create a peering connection](https://cloud.google.com/vpc/docs/using-vpc-peering#creating_a_peering_configuration) from your target VPC to your Astro cluster VPC. For example, you can use the following gcloud CLI command to create the connection:

   ```bash
   gcloud compute networks peerings create <choose-any-name> --network=<your-target-vpc-network-name>  --peer-project=<your-cluster-project-id> --peer-network=<your-cluster-vpc-name>
   ```

After both VPC peering connections have been created, the connection becomes active.

</TabItem>

<TabItem value="Private-Service-Connect">

Use Private Service Connect (PSC) to create private connections from Astro to GCP services without connecting over the public internet. See [Private Service Connect](https://cloud.google.com/vpc/docs/private-service-connect) to learn more.

Astro clusters are by default configured with a PSC endpoint with a target of [All Google APIs](https://cloud.google.com/vpc/docs/private-service-connect-compatibility#google-apis-global). To provide a secure-by-default configuration, a DNS zone is created with a resource record that will route all requests made to `*.googleapis.com` through this PSC endpoint. This ensures that requests made to these services are made over PSC without any additional user configuration. As an example, requests to `storage.googleapis.com` will be routed through this PSC endpoint.

A list of Google services and their associated service names are provided in the [Google APIs Explorer Directory](https://developers.google.com/apis-explorer). Alternatively, you can run the following command in the Google Cloud CLI to return a list of Google services and their associated service names:

```bash
gcloud services list --available --filter="name:googleapis.com"
```

</TabItem>

</Tabs>

## See Also

- [Manage Airflow connections and variables](manage-connections-variables.md)
- [Authorize your Deployment using workload identity](authorize-deployments-to-your-cloud.md)
