---
sidebar_label: 'With GitHub Actions'
title: 'Run your first DAG with GitHub Actions'
id: 'first-dag-github-actions'
---



Astro is the industry's leading managed service for Apache Airflow. To quickly learn how Astro works, follow the steps in this quickstart to create an Airflow environment and run your first DAG with GitHub Actions.

Specifically, you will:

- Authenticate and log in to Astro.
- Create a Deployment.
- Fork an example GitHub repository with a new Astro project.
- Configure GitHub Actions.
- Trigger the GitHub Action to deploy an example DAG to Astro.
- Trigger a run of the example DAG in the Airflow UI.

The steps take about 15 minutes. If you prefer to use a CLI, you can alternatively create and run your first DAG [using the Astro CLI](first-dag-cli.md) in the same amount of time.

This tutorial assumes that you're familiar with basic Apache Airflow concepts. If you're new to Airflow and want a more general introduction, see the [Airflow 101 Learning Path](https://academy.astronomer.io/path/airflow-101).

## Prerequisites

- An Astro account. To start an Astro trial and create your free trial account, see [Start a trial](trial.md).
- A [GitHub account](https://docs.github.com/en/get-started/signing-up-for-github).


:::info

If you're on your organization's network and can't access Astro, make a request to allowlist the following domains on your network:

- `https://cloud.astronomer.io/`
- `https://api.astronomer.io/`
- `https://images.astronomer.cloud/`
- `https://auth.astronomer.io/`
- `https://updates.astronomer.io/`
- `https://install.astronomer.io/`
- `https://install.astronomer.io/`
- `https://astro-<organization-short-name>.datakin.com/`
- `https://<organization-short-name>.astronomer.run/`

:::

## Step 1: Create a Deployment

An Astro _Deployment_ is an instance of Apache Airflow that is powered by all core Airflow components, including a webserver, scheduler, and one or more workers. You deploy DAGs to a Deployment, and you can have one or more Deployments within your Workspace.

1. Log in to the [Cloud UI](https://cloud.astronomer.io).

2. On the **Deployments** page, click **+ Deployment**.

3. In the **Name** field, enter a name for your Deployment. You can leave the other fields at their default values. This creates a basic Deployment on a standard Astronomer-hosted cluster. You can delete the Deployment after you finish testing your example DAG runs.

4. Click **Create Deployment**.

    A confirmation message appears indicating that the Deployment status is **Creating** until all underlying components in the Deployment are healthy. During this time, the Airflow UI is unavailable and you can't deploy code or modify Deployment settings. When the Deployment is ready, the status changes to **Healthy**.

    For more information about possible Deployment health statuses, see [Deployment health](deployment-health-incidents.md). Or, to learn more about how to customize your Deployment settings, see [Deployment settings](deployment-settings.md).

:::tip

Astro contains an in-product tutorial that guides you through Steps 2-4 of this document and includes shortcut buttons for some key Astro actions. If you prefer to finish the quickstart this way, open your **Deployments** page in the Cloud UI and choose your Deployment. In the **Deploy your first DAG** section, click **With GitHub Actions** and follow the steps in the window that appears.

If you don't see the **Deploy your first DAG** option your Deployment page, click **Deploy DAGs ?** to open it.

:::

## Step 2: Fork the example project repository

This repository contains an _Astro project_, which is a collection of files required for running Airflow on Astro. An Astro project includes folders for DAG files, plugins, dependencies, and more. Specifically, this Astro project includes an example DAG which, when you run it, retrieves a list of countries from an Astro S3 data store and filters the list through a data transform.

1. Open [the example project repository](https://github.com/astronomer/astro-example-dags/fork) in a new tab or browser window.

2. **Choose an owner** from your available options.

3. Keep the selection to **Copy the `main` branch only**.

4. Click **Create fork**.

## Step 3: Set up the GitHub Actions Workflow

This example repository also includes a pre-configured [Astronomer deploy action](https://github.com/astronomer/deploy-action), which you can use to set up a CI/CD deployment pipeline. In this step, you'll configure the GitHub action to deploy code from your forked repository to Astro and run the workflow.

1. Open two browser windows: one with the [Cloud UI](https://cloud.astronomer.io), and one with your forked GitHub repository.

2. In the Cloud UI, choose the Deployment where you want to deploy your Astro project.

3. In GitHub, open your forked repository and click **Actions**.

4. Click **I understand my workflows, go ahead and enable them.**

  The [workflow](https://github.com/astronomer/astro-example-dags/blob/main/.github/workflows/deploy-to-astro.yaml) is a script that uses API tokens to deploy DAGs from a GitHub repository to your Deployment, without requiring any local development.

5. Choose the **Astronomer CI - Deploy Code** workflow.

6. Click **Run workflow**. This opens a modal to enter information about your Astro Deployment.

7. In the Cloud UI, copy your **Deployment ID** from the Deployment information.

8. In GitHub, paste your **Deployment ID**.

9. In the Cloud UI, click **API Tokens**.

10. Click **+ API Token** to create a new API token, and give the token a **Name** and an **Expiration**.

11. Click **Create API Token**, then copy the token that appears.

    :::warning

    For security reasons, this is the only opportunity you have to copy your API token. After you exit the modal window, you cannot copy it again. Be sure to save your token in a safe place or paste it immediately.

    :::

12. In GitHub, paste the API Token in the **API Token** field on your GitHub Actions workflow page.

13. Click **Run workflow**.

This automatically deploys the example DAGs in your Astro project to your Deployment.

## Step 4: View your DAG run results

Open your Deployment in the Cloud UI and click **DAGs** in the left sidebar, then click **S3**. From this page, you can see that the `s3` DAG has run exactly once.

![Detailed view of the S3 DAG run outcome.](/img/docs/s3-complete.png)

The **DAGs** page of the Cloud UI includes the most commonly used information and actions from the Airflow UI in one place. If you prefer to view your DAG run in the Airflow UI, click **Open Airflow** in the upper right corner of the page.

Congratulations! You deployed and ran your first DAG on Astro with GitHub Actions.

## Next Steps

- Develop your [Astro project](/astro/cli/run-airflow-locally).
- Read more about [Developing CI/CD workflows](set-up-ci-cd.md).
- Install [the CLI](/astro/cli/install-cli.md) to test DAGs or run Airflow locally.
- [Write your First DAG](https://docs.astronomer.io/learn/get-started-with-airflow).
