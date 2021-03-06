---
title: Azure Automation Update Management overview
description: This article provides an overview of the Update Management feature that implements updates for your Windows and Linux machines.
services: automation
ms.subservice: update-management
ms.date: 04/01/2021
ms.topic: conceptual
---

# Update Management overview

You can use Update Management in Azure Automation to manage operating system updates for your Windows and Linux virtual machines in Azure, in on-premises environments, and in other cloud environments. You can quickly assess the status of available updates on all agent machines and manage the process of installing required updates for servers.

As a service provider, you may have onboarded multiple customer tenants to [Azure Lighthouse](../../lighthouse/overview.md). Azure Lighthouse allows you to perform operations at scale across several Azure Active Directory (Azure AD) tenants at once, making management tasks like Update Management more efficient across those tenants you're responsible for.

> [!NOTE]
> You can't use a machine configured with Update Management to run custom scripts from Azure Automation. This machine can only run the Microsoft-signed update script.

> [!NOTE]
> At this time, enabling Update Management directly from an Arc enabled server is not supported. See [Enable Update Management from your Automation account](../../automation/update-management/enable-from-automation-account.md) to understand requirements and how to enable for your server.

To download and install available *Critical* and *Security* patches automatically on your Azure VM, review [Automatic VM guest patching](../../virtual-machines/automatic-vm-guest-patching.md) for Windows VMs.

Before deploying Update Management and enabling your machines for management, make sure that you understand the information in the following sections.

## About Update Management

Machines that are managed by Update Management rely on the following to perform assessment and to deploy updates:

* [Log Analytics agent](../../azure-monitor/agents/log-analytics-agent.md) for Windows or Linux
* PowerShell Desired State Configuration (DSC) for Linux
* Automation Hybrid Runbook Worker (automatically installed when you enable Update Management on the machine)
* Microsoft Update or [Windows Server Update Services](/windows-server/administration/windows-server-update-services/get-started/windows-server-update-services-wsus) (WSUS) for Windows machines
* Either a private or public update repository for Linux machines

The following diagram illustrates how Update Management assesses and applies security updates to all connected Windows Server and Linux servers in a workspace:

![Update Management workflow](./media/overview/update-mgmt-updateworkflow.png)

Update Management can be used to natively deploy to machines in multiple subscriptions in the same tenant, or across tenants using [Azure delegated resource management](../../lighthouse/concepts/azure-delegated-resource-management.md).

After a package is released, it takes 2 to 3 hours for the patch to show up for Linux machines for assessment. For Windows machines, it takes 12 to 15 hours for the patch to show up for assessment after it's been released. When a machine completes a scan for update compliance, the agent forwards the information in bulk to Azure Monitor logs. On a Windows machine, the compliance scan is run every 12 hours by default. For a Linux machine, the compliance scan is performed every hour by default. If the Log Analytics agent is restarted, a compliance scan is started within 15 minutes.

In addition to the scan schedule, the scan for update compliance is started within 15 minutes of the Log Analytics agent being restarted, before update installation, and after update installation.

Update Management reports how up to date the machine is based on what source you're configured to sync with. If the Windows machine is configured to report to [Windows Server Update Services](/windows-server/administration/windows-server-update-services/get-started/windows-server-update-services-wsus) (WSUS), depending on when WSUS last synced with Microsoft Update, the results might differ from what Microsoft Update shows. This behavior is the same for Linux machines that are configured to report to a local repo instead of to a public repo.

> [!NOTE]
> To properly report to the service, Update Management requires certain URLs and ports to be enabled. To learn more about these requirements, see [Network configuration](../automation-hybrid-runbook-worker.md#network-planning).

You can deploy and install software updates on machines that require the updates by creating a scheduled deployment. Updates classified as optional aren't included in the deployment scope for Windows machines. Only required updates are included in the deployment scope.

The scheduled deployment defines which target machines receive the applicable updates. It does so either by explicitly specifying certain machines or by selecting a [computer group](../../azure-monitor/logs/computer-groups.md) that's based on log searches of a specific set of machines (or on an [Azure query](query-logs.md) that dynamically selects Azure VMs based on specified criteria). These groups differ from [scope configuration](../../azure-monitor/insights/solution-targeting.md), which is used to control the targeting of machines that receive the configuration to enable Update Management. This prevents them from performing and reporting update compliance, and install approved required updates.

While defining a deployment, you also specify a schedule to approve and set a time period during which updates can be installed. This period is called the maintenance window. A 20-minute span of the maintenance window is reserved for reboots, assuming one is needed and you selected the appropriate reboot option. If patching takes longer than expected and there's less than 20 minutes in the maintenance window, a reboot won't occur.

Updates are installed by runbooks in Azure Automation. You can't view these runbooks, and they don't require any configuration. When an update deployment is created, it creates a schedule that starts a master update runbook at the specified time for the included machines. The master runbook starts a child runbook on each agent to install the required updates.

At the date and time specified in the update deployment, the target machines execute the deployment in parallel. Before installation, a scan is run to verify that the updates are still required. For WSUS client machines, if the updates aren't approved in WSUS, update deployment fails.

Having a machine registered for Update Management in more than one Log Analytics workspace (also referred to as multihoming) isn't supported.

## Clients

### Supported operating systems

The following table lists the supported operating systems for update assessments and patching. Patching requires a system Hybrid Runbook Worker, which is automatically installed when you enable the virtual machine or server for management by Update Management. For information on Hybrid Runbook Worker system requirements, see [Deploy a Windows Hybrid Runbook Worker](../automation-windows-hrw-install.md) and a [Deploy a Linux Hybrid Runbook Worker](../automation-linux-hrw-install.md).

> [!NOTE]
> Update assessment of Linux machines is only supported in certain regions as listed in the Automation account and Log Analytics workspace [mappings table](../how-to/region-mappings.md#supported-mappings).

|Operating system  |Notes  |
|---------|---------|
|Windows Server 2019 (Datacenter/Standard including Server Core)<br><br>Windows Server 2016 (Datacenter/Standard excluding Server Core)<br><br>Windows Server 2012 R2(Datacenter/Standard)<br><br>Windows Server 2012 | |
|Windows Server 2008 R2 (RTM and SP1 Standard)| Update Management supports assessments and patching for this operating system. The [Hybrid Runbook Worker](../automation-windows-hrw-install.md) is supported for Windows Server 2008 R2. |
|CentOS 6, 7, and 8 (x64)      | Linux agents require access to an update repository. Classification-based patching requires `yum` to return security data that CentOS doesn't have in its RTM releases. For more information on classification-based patching on CentOS, see [Update classifications on Linux](view-update-assessments.md#linux).          |
|Red Hat Enterprise 6, 7, and 8 (x64)     | Linux agents require access to an update repository.        |
|SUSE Linux Enterprise Server 12, 15, and 15.1 (x64)     | Linux agents require access to an update repository. For SUSE 15.x, Python 3 is required on the machine.      |
|Ubuntu 14.04 LTS, 16.04 LTS, and 18.04 LTS (x64)      |Linux agents require access to an update repository.         |

> [!NOTE]
> Update Management does not support safely automating update management across all instances in an Azure virtual machine scale set. [Automatic OS image upgrades](../../virtual-machine-scale-sets/virtual-machine-scale-sets-automatic-upgrade.md) is the recommended method for managing OS image upgrades on your scale set.

### Unsupported operating systems

The following table lists operating systems not supported by Update Management:

|Operating system  |Notes  |
|---------|---------|
|Windows client     | Client operating systems (such as Windows 7 and Windows 10) aren't supported.<br> For Azure Windows Virtual Desktop (WVD), the recommended method<br> to manage updates is [Microsoft Endpoint Configuration Manager](../../virtual-desktop/configure-automatic-updates.md) for Windows 10 client machine patch management. |
|Windows Server 2016 Nano Server     | Not supported.       |
|Azure Kubernetes Service Nodes | Not supported. Use the patching process described in [Apply security and kernel updates to Linux nodes in Azure Kubernetes Service (AKS)](../../aks/node-updates-kured.md)|

### System requirements

The following information describes operating system-specific requirements. For additional guidance, see [Network planning](#ports). To understand requirements for TLS 1.2, see [TLS 1.2 enforcement for Azure Automation](../automation-managing-data.md#tls-12-enforcement-for-azure-automation).

#### Windows

Software Requirements:

- .NET Framework 4.6 or later is required. ([Download the .NET Framework](/dotnet/framework/install/guide-for-developers).
- Windows PowerShell 5.1 is required ([Download Windows Management Framework 5.1](https://www.microsoft.com/download/details.aspx?id=54616).)

Windows agents must be configured to communicate with a WSUS server, or they require access to Microsoft Update. For hybrid machines, we recommend installing the Log Analytics agent for Windows by first connecting your machine to [Azure Arc enabled servers](../../azure-arc/servers/overview.md), and then use Azure Policy to assign the [Deploy Log Analytics agent to Windows Azure Arc machines](../../governance/policy/samples/built-in-policies.md#monitoring) built-in policy. Alternatively, if you plan to monitor the machines with Azure Monitor for VMs, instead use the [Enable Azure Monitor for VMs](../../governance/policy/samples/built-in-initiatives.md#monitoring) initiative.

You can use Update Management with Microsoft Endpoint Configuration Manager. To learn more about integration scenarios, see [Integrate Update Management with Windows Endpoint Configuration Manager](mecmintegration.md). The [Log Analytics agent for Windows](../../azure-monitor/agents/agent-windows.md) is required for Windows servers managed by sites in your Configuration Manager environment.

By default, Windows VMs that are deployed from Azure Marketplace are set to receive automatic updates from Windows Update Service. This behavior doesn't change when you add Windows VMs to your workspace. If you don't actively manage updates by using Update Management, the default behavior (to automatically apply updates) applies.

> [!NOTE]
> You can modify Group Policy so that machine reboots can be performed only by the user, not by the system. Managed machines can get stuck if Update Management doesn't have rights to reboot the machine without manual interaction from the user. For more information, see [Configure Group Policy settings for Automatic Updates](/windows-server/administration/windows-server-update-services/deploy/4-configure-group-policy-settings-for-automatic-updates).

#### Linux

Software Requirements:

- The machine requires access to an update repository, either private or public.
- TLS 1.1 or TLS 1.2 is required to interact with Update Management.
- Python 2.x installed.

> [!NOTE]
> Update assessment of Linux machines is only supported in certain regions. See the Automation account and Log Analytics workspace [mappings table](../how-to/region-mappings.md#supported-mappings).

For hybrid machines, we recommend installing the Log Analytics agent for Linux by first connecting your machine to [Azure Arc enabled servers](../../azure-arc/servers/overview.md), and then use Azure Policy to assign the [Deploy Log Analytics agent to Linux Azure Arc machines](../../governance/policy/samples/built-in-policies.md#monitoring) built-in policy. Alternatively, if you plan to monitor the machines with Azure Monitor for VMs, instead use the [Enable Azure Monitor for VMs](../../governance/policy/samples/built-in-initiatives.md#monitoring) initiative.

VMs created from the on-demand Red Hat Enterprise Linux (RHEL) images that are available in Azure Marketplace are registered to access the [Red Hat Update Infrastructure (RHUI)](../../virtual-machines/workloads/redhat/redhat-rhui.md) that's deployed in Azure. Any other Linux distribution must be updated from the distribution's online file repository by using methods supported by the distribution.

## Permissions

To create and manage update deployments, you need specific permissions. To learn about these permissions, see [Role-based access - Update Management](../automation-role-based-access-control.md#update-management-permissions).

## Update Management components

Update Management uses the resources described in this section. These resources are automatically added to your Automation account when you enable Update Management.

### Hybrid Runbook Worker groups

After you enable Update Management, any Windows machine that's directly connected to your Log Analytics workspace is automatically configured as a system Hybrid Runbook Worker to support the runbooks that support Update Management.

Each Windows machine that's managed by Update Management is listed in the Hybrid worker groups pane as a System hybrid worker group for the Automation account. The groups use the `Hostname FQDN_GUID` naming convention. You can't target these groups with runbooks in your account. If you try, the attempt fails. These groups are intended to support only Update Management. To learn more about viewing the list of Windows machines configured as a Hybrid Runbook Worker, see [view Hybrid Runbook Workers](../automation-hybrid-runbook-worker.md#view-system-hybrid-runbook-workers).

You can add the Windows machine to a user Hybrid Runbook Worker group in your Automation account to support Automation runbooks if you use the same account for Update Management and the Hybrid Runbook Worker group membership. This functionality was added in version 7.2.12024.0 of the Hybrid Runbook Worker.

### Management packs

If your Operations Manager management group is [connected to a Log Analytics workspace](../../azure-monitor/agents/om-agents.md), the following management packs are installed in Operations Manager. These management packs are also installed for Update Management on directly connected Windows machines. You don't need to configure or manage these management packs.

* Microsoft System Center Advisor Update Assessment Intelligence Pack (Microsoft.IntelligencePacks.UpdateAssessment)
* Microsoft.IntelligencePack.UpdateAssessment.Configuration (Microsoft.IntelligencePack.UpdateAssessment.Configuration)
* Update Deployment MP

> [!NOTE]
> If you have an Operations Manager 1807 or 2019 management group connected to a Log Analytics workspace with agents configured in the management group to collect log data, you need to override the parameter `IsAutoRegistrationEnabled` and set it to True in the **Microsoft.IntelligencePacks.AzureAutomation.HybridAgent.Init** rule.

For more information about updates to management packs, see [Connect Operations Manager to Azure Monitor logs](../../azure-monitor/agents/om-agents.md).

> [!NOTE]
> For Update Management to fully manage machines with the Log Analytics agent, you must update to the Log Analytics agent for Windows or the Log Analytics agent for Linux. To learn how to update the agent, see [How to upgrade an Operations Manager agent](/system-center/scom/deploy-upgrade-agents). In environments that use Operations Manager, you must be running System Center Operations Manager 2012 R2 UR 14 or later.

## Data collection

### Supported sources

The following table describes the connected sources that Update Management supports:

| Connected source | Supported | Description |
| --- | --- | --- |
| Windows agents |Yes |Update Management collects information about system updates from Windows agents and then starts installation of required updates. |
| Linux agents |Yes |Update Management collects information about system updates from Linux agents and then starts installation of required updates on supported distributions. |
| Operations Manager management group |Yes |Update Management collects information about system updates from agents in a connected management group.<br/><br/>A direct connection from the Operations Manager agent to Azure Monitor logs isn't required. Data is forwarded from the management group to the Log Analytics workspace. |

### Collection frequency

Update Management scans managed machines for data using the following rules. It can take between 30 minutes and 6 hours for the dashboard to display updated data from managed machines.

* Each Windows machine - Update Management does a scan twice per day for each machine.

* Each Linux machine - Update Management does a scan every hour.

The average data usage by Azure Monitor logs for a machine using Update Management is approximately 25 MB per month. This value is only an approximation and is subject to change, depending on your environment. We recommend that you monitor your environment to keep track of your exact usage. For more information about analyzing Azure Monitor Logs data usage, see [Manage usage and cost](../../azure-monitor/logs/manage-cost-storage.md).

## <a name="ports"></a>Network planning

Check [Azure Automation Network Configuration](../automation-network-configuration.md#hybrid-runbook-worker-and-state-configuration) for detailed information on the ports, URLs, and other networking details required for Update Management.

For Windows machines, you must also allow traffic to any endpoints required by Windows Update. You can find an updated list of required endpoints in [Issues related to HTTP/Proxy](/windows/deployment/update/windows-update-troubleshooting#issues-related-to-httpproxy). If you have a local [Windows Update server](/windows-server/administration/windows-server-update-services/plan/plan-your-wsus-deployment), you must also allow traffic to the server specified in your [WSUS key](/windows/deployment/update/waas-wu-settings#configuring-automatic-updates-by-editing-the-registry).

For Red Hat Linux machines, see [IPs for the RHUI content delivery servers](../../virtual-machines/workloads/redhat/redhat-rhui.md#the-ips-for-the-rhui-content-delivery-servers) for required endpoints. For other Linux distributions, see your provider documentation.

For more information about ports required for the Hybrid Runbook Worker, see [Update Management addresses for Hybrid Runbook Worker](../automation-hybrid-runbook-worker.md#update-management-addresses-for-hybrid-runbook-worker).

If your IT security policies do not allow machines on the network to connect to the internet, you can set up a [Log Analytics gateway](../../azure-monitor/agents/gateway.md) and then configure the machine to connect through the gateway to Azure Automation and Azure Monitor.

## Update classifications

The following table defines the classifications that Update Management supports for Windows updates.

|Classification  |Description  |
|---------|---------|
|Critical updates     | An update for a specific problem that addresses a critical, non-security-related bug.        |
|Security updates     | An update for a product-specific, security-related issue.        |
|Update rollups     | A cumulative set of hotfixes that are packaged together for easy deployment.        |
|Feature packs     | New product features that are distributed outside a product release.        |
|Service packs     | A cumulative set of hotfixes that are applied to an application.        |
|Definition updates     | An update to virus or other definition files.        |
|Tools     | A utility or feature that helps complete one or more tasks.        |
|Updates     | An update to an application or file that currently is installed.        |

The next table defines the supported classifications for Linux updates.

|Classification  |Description  |
|---------|---------|
|Critical and security updates     | Updates for a specific problem or a product-specific, security-related issue.         |
|Other updates     | All other updates that aren't critical in nature or that aren't security updates.        |

>[!NOTE]
>Update classification for Linux machines is only available when used in supported Azure public cloud regions. There is no classification of Linux updates when using Update Management in the following national cloud regions:
>
>* Azure US Government
>* 21Vianet in China
>
> Instead of being classified, updates are reported under the **Other updates** category.
>
> Update Management uses data published by the supported distributions, specifically their released [OVAL](https://oval.mitre.org/) (Open Vulnerability and Assessment Language) files. Because internet access is restricted from these national clouds, Update Management cannot access the files.

For Linux, Update Management can distinguish between critical updates and security updates in the cloud under classification **Security** and **Others**, while displaying assessment data due to data enrichment in the cloud. For patching, Update Management relies on classification data available on the machine. Unlike other distributions, CentOS does not have this information available in the RTM version. If you have CentOS machines configured to return security data for the following command, Update Management can patch based on classifications.

```bash
sudo yum -q --security check-update
```

There's currently no supported method to enable native classification-data availability on CentOS. At this time, limited support is provided to customers who might have enabled this feature on their own.

To classify updates on Red Hat Enterprise version 6, you need to install the yum-security plugin. On Red Hat Enterprise Linux 7, the plugin is already a part of yum itself and there's no need to install anything. For more information, see the following Red Hat [knowledge article](https://access.redhat.com/solutions/10021).

When you schedule an update to run on a Linux machine, that for example is configured to install only updates matching the **Security** classification, the updates installed might be different from, or are a subset of the updates matching this classification. When an assessment of OS updates pending for your Linux machine is performed, [Open Vulnerability and Assessment Language](https://oval.mitre.org/) (OVAL) files provided by the Linux distro vendor is used by Update Management for classification.

Categorization is done for Linux updates as **Security** or **Others** based on the OVAL files, which includes updates addressing security issues or vulnerabilities. But when the update schedule is run, it executes on the Linux machine using the appropriate package manager like YUM, APT or ZYPPER to install them. The package manager for the Linux distro may have a different mechanism to classify updates, where the results may differ from the ones obtained from OVAL files by Update Management. To manually check the machine and understand which updates are security relevant by your package manager, see [Troubleshoot Linux update deployment](../troubleshoot/update-management.md#updates-linux-installed-different).

## Integrate Update Management with Configuration Manager

Customers who have invested in Microsoft Endpoint Configuration Manager for managing PCs, servers, and mobile devices also rely on the strength and maturity of Configuration Manager to help manage software updates. To learn how to integrate Update Management with Configuration Manager, see [Integrate Update Management with Windows Endpoint Configuration Manager](mecmintegration.md).

## Third-party updates on Windows

Update Management relies on the locally configured update repository to update supported Windows systems, either WSUS or Windows Update. Tools such as [System Center Updates Publisher](/configmgr/sum/tools/updates-publisher) allow you to import and publish custom updates with WSUS. This scenario allows Update Management to update machines that use Configuration Manager as their update repository with third-party software. To learn how to configure Updates Publisher, see [Install Updates Publisher](/configmgr/sum/tools/install-updates-publisher).

## Enable Update Management

Here are the ways that you can enable Update Management and select machines to be managed:

- Using an Azure [Resource Manager template](enable-from-template.md) to deploy Update Management to a new or existing Automation account and Azure Monitor Log Analytics workspace in your subscription. It does not configure the scope of machines that should be managed, this is performed as a separate step after using the template.

- From your [Automation account](enable-from-automation-account.md) for one or more Azure and non-Azure machines, including Arc enabled servers.

- Using the **Enable-AutomationSolution** [runbook](enable-from-runbook.md) method.

- For a [selected Azure VM](enable-from-vm.md) from the **Virtual machines** page in the Azure portal. This scenario is available for Linux and Windows VMs.

- For [multiple Azure VMs](enable-from-portal.md) by selecting them from the **Virtual machines** page in the Azure portal.

> [!NOTE]
> Update Management requires linking a Log Analytics workspace to your Automation account. For a definitive list of supported regions, see [Azure Workspace mappings](../how-to/region-mappings.md). The region mappings don't affect the ability to manage VMs in a separate region from your Automation account.

## Next steps

* For details of working with Update Management, see [Manage updates for your VMs](manage-updates-for-vm.md).

* Review commonly asked questions about Update Management in the [Azure Automation frequently asked questions](../automation-faq.md).