> [!IMPORTANT]
> This project is _community supported_. This means that you should not contact
> Qlik Support for help with these apps. See below for additional information
> on how to raise issues on this repository to ask for help.

# Qlik Cloud AI Analyzer

> **Note**
> This application is one of a set of community built and supported monitoring apps for Qlik Cloud.
> For links to all monitoring apps, visit the [Qlik Cloud Monitoring Apps](https://github.com/qlik-oss/qlik-cloud-monitoring-apps) repository.

The AI Analyzer is a Qlik Sense application built for Qlik Cloud, which helps you to analyze and monitor Qlik Answers and MCP usage across your tenant. It replaces the legacy Answers Analyzer.

The goal of the app is to track the **who / what / where** of AI usage in your tenant. It shows which users are consuming questions, in which spaces that consumption is happening, and through what medium (for example, Qlik Answers within a specific application, or usage via MCP).

It provides information such as:

* **Who** – which users are asking questions, how many they've asked this month vs. last month, and how their usage is trending
* **Where** – which spaces and assets (resources) the questions are being asked against
* **What / How** – the source of each question (e.g. Qlik Answers in a given app, or MCP), and for MCP, which tools are being called, by which user agent, and with what latency
* **Trends** – monthly usage (current year vs. prior), daily usage (current month vs. prior), and accumulated usage (current month vs. prior)

**Overview sheet**

![Overview sheet in the AI Analyzer](images/Overview%20Sheet.png)

**MCP Detail sheet**

![MCP Detail sheet in the AI Analyzer](images/MCP%20Sheet.png)

## Installing / Updating

These applications can be installed via an interactive Qlik Automate [installer](https://community.qlik.com/t5/Official-Support-Articles/Qlik-Cloud-Monitoring-Apps-Workflow-Guide/ta-p/2134140) or [manually](/../../releases) using the PDF guide.

## Configuration

After importing the app, open the load script and go to the **Variable Set Up / Config** tab. Four variables need to be populated before the first reload:

| Variable | Description |
| --- | --- |
| `vDataConnection` | The REST data connection the app uses to call the Qlik Cloud APIs. See [Data connection requirements](#data-connection-requirements) below. |
| `vQVDSpaceName` | The name of an **existing** space where the QVDs produced by this app will be stored. |
| `vQVDFolderName` | The name of the folder within the above space that the QVDs will be placed in. This folder does **not** need to exist beforehand. |
| `vIncrementalMode` | `0` = full load on every reload. `1` = incremental reloads after the initial full load. |

### Data connection requirements

If you already have other Qlik Cloud monitoring apps installed, you can reuse the data connection those apps use.

If not, create a REST connection to any Qlik Cloud API endpoint on your tenant and make sure that:

* The **Allow "WITH CONNECTION"** option is checked.
* The owner of the connection (specifically, the owner of the API key used in the connection) has:
  * the **Tenant Admin** role,
  * the **Audit Admin** role, and
  * permission for **Features and Actions > Agentic AI > Data analysis**, granted either through the default user permissions or a custom role.

### Multi-tenant support

The app can also report on AI usage across multiple tenants. In multi-tenant mode, each tenant stores the QVDs it produces in its own cloud storage location (for example, Amazon S3), and a designated parent app pulls that data together into a single view. The following variables control this behavior:

| Variable | Description |
| --- | --- |
| `vu_multi_tenant_enabled` | `0` = multi-tenant support off. `1` = multi-tenant support on. |
| `vu_is_parent_app` | When running in multi-tenant mode, set to `1` to make this app the parent, which pulls data from the other tenants into itself. |
| `vMulti_Tenant_Storage_Location` | The data connection for the cloud storage location (e.g. S3) where the QVDs produced for multi-tenant reporting are stored. Each tenant writes its QVDs to the storage location specified for it, and the parent app reads them from there. |

## Known limitations

* **Question counts may be slightly lower than actual consumption.** Questions used by Bias detection in Qlik Predict are not yet tracked, so they are not included in the app's question counts.

## Subscription Compatibility

Some monitoring apps are designed for specific Qlik Cloud subscription types. Refer to the compatibility matrix within the [Qlik Cloud Monitoring Apps](https://github.com/qlik-oss/qlik-cloud-monitoring-apps?tab=readme-ov-file#applications) repository.

## Support policy

This app is provided as-is and is not supported by Qlik. Over time, the APIs and
metrics used by the app may change, so it is advised to monitor this repository
for updates, and to update the app promptly when new versions are available.

If you have issues while using this app, support is provided on a best-efforts
basis by contributors to this project.

If you have an issue:

* Review the [Known limitations](#known-limitations) section in this readme to see if your issue is a common one
* Review open and closed [issues](/../../issues)
* Open a [new issue](/../../issues/new), following the issue template

If you are able to resolve the issue, then close your issue with the resolution,
and if you feel inclined, open a PR with your proposed changes so that we can
consider including the improvement in the next release of the app.

Thank you for your support!
