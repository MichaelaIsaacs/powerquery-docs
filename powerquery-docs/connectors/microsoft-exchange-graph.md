---
title: Power Query Microsoft Exchange (Graph) connector
description: Provides basic information and connection instructions for the Microsoft Exchange (Graph) connector, which reads mailbox data using Microsoft Graph.
author: misaacs
ms.topic: concept-article
ms.date: 08/05/2026
ms.update-cycle: 1095-days
ms.author: misaacs
ms.subservice: connectors
ms.custom: sfi-image-nochange
---

# Microsoft Exchange (Graph)

> [!NOTE]
> The Microsoft Exchange (Graph) connector is in **preview** and released in **Power BI Desktop only** for the initial rollout. It reads mailbox data through Microsoft Graph using an organizational (Microsoft Entra) account.
>
> This connector is planned as the successor to the legacy [Microsoft Exchange](microsoft-exchange.md) and [Microsoft Exchange Online](microsoft-exchange-online.md) connectors. For migration guidance, see <!-- TODO: link migration KB when published -->.

## Summary

| Item | Description |
| ---- | ----------- |
| Release State | Preview |
| Products | Power BI Desktop |
| Authentication Types Supported | Organizational account (Microsoft Entra ID) |
| Function Reference Documentation | <!-- TODO: link once M function reference is published --> |

## What's supported in the initial preview

The initial preview release includes:

- **Power BI Desktop** as the host.
- **Mail** as the supported mailbox data entry point.
- **Organizational account** sign-in via Microsoft Entra ID with delegated Microsoft Graph permissions.

## Coming soon

The following are planned but **not available in the initial preview**:

- **Additional mailbox data entry points**: Calendar, Contacts, and Tasks. These become available as the first-party application scopes (`Mail.Read`, `Mail.Read.Shared`, and Calendars scopes) are finalized.
- **Excel** as a host. The Power Query build shipping with the September Excel update turns on the new connector behind a feature flag for customer validation, with broader Excel availability planned after validation.
- **Power BI service and dataflows, Fabric Dataflow Gen2, Power Apps dataflows, Customer Insights dataflows, and Power Query Online**. These surfaces are planned after Power BI Desktop preview stabilizes.

Timelines are subject to change based on preview feedback.

## Prerequisites

- A Microsoft 365 mailbox accessible through Microsoft Graph.
- Sign-in with an organizational (Microsoft Entra) account that has permission to read the mailbox data.
- <!-- TODO: confirm any tenant admin consent requirements for the delegated Graph scopes (Mail.Read, Mail.Read.Shared) shipped with the first-party app -->

## Capabilities supported

- Import

## Connect to Microsoft Exchange (Graph) from Power Query Desktop

To connect to Microsoft Exchange (Graph) from Power BI Desktop:

1. In the **Get Data** experience, select **Online Services**, select **Microsoft Exchange (Graph)**, and then select **Connect**.

   <!-- TODO: add Get Data screenshot -->

2. Choose the mailbox data entry point you want to read (**Mail** in the initial preview), and then select **OK**.

   <!-- TODO: add entry point selection screenshot -->

3. When prompted to sign in, select **Sign in** and complete authentication with your organizational account.

   <!-- TODO: add sign-in screenshot -->

4. In **Navigator**, browse the returned folders and select the data to import. Select **Load** to load the table, or **Transform Data** to open the Power Query Editor to filter and refine the set of data you want to use.

   <!-- TODO: add Navigator screenshot -->

## Known limitations and considerations

- **Preview scope**: The initial preview supports **Power BI Desktop** and **Mail** only. Other entry points and product surfaces are not yet available. See [Coming soon](#coming-soon).
- **Sort and order query folding**: A known query-folding limitation affects sort and order operations against Graph in the current preview build. <!-- TODO: describe symptom, workaround, and ADO tracking ID once eng finalizes wording -->
- **Get Data experiences**: <!-- TODO: reflect resolution of the modern vs. legacy Get Data behavior surfaced during the July 29 bug bash once confirmed -->
- <!-- TODO: confirm sovereign cloud, GCC, LTS/SAC scope from the open items list before public preview -->

## Migrating from the legacy Exchange connectors

If you're using the existing [Microsoft Exchange](microsoft-exchange.md) or [Microsoft Exchange Online](microsoft-exchange-online.md) connectors, see the migration guide: <!-- TODO: link migration KB when published -->
