---
layout: post
title: "What are Windows eventlogs and why should you care"
date: 2021-04-05 23:00:00 +0200
categories: forensics preparation
tags: incident-response windows eventlogs forensics
---
Hello world,

this post is part one of my [Preparing Windows eventlogs for forensics and other analysis usage (Overview)][windows-eventlogs-overview-post] series. The other posts can be reached through the overview page. The next post will be about _Increasing the Windows eventlog size for selected logs_. So without any further ado, lets get started.

## Table of Contents
1. [What are Windows eventlogs](#definition)
2. [High-Level Structure of Windows eventlogs](#structure)
3. [Use of Windows eventlogs](#useage)

<br/>

## What are Windows eventlogs? <a name="definition"></a>

The Windows eventlogs are the default system event logs for Microsoft Windows OS and are available since Windows NT ('93) through all releases including Windows Server OS. Most Windows components and services (pre-installed or on demand installed) write their log information as an eventlog as well. Eventlogs are written in a separate file for each component/service but there are some more general eventlogs for the generic system events. The SMB Service for example has extra eventlogs but also writes to some extend in the more general system and security eventlogs.

What eventlogs are available depends on the installed software and the Windows version and Windows components installed. All current Windows systems share a base set of eventlogs like ``Security``, ``System`` or ``User Profile Service`` but the majority of eventlogs comes from installed services and applications. For example installing the ActiveDirectory-Role for Windows Server will create multiple new eventlogs as well to log the corresponding domain controller actions.
There is also the possibility for 3rd party vendors to use the existing windows eventlogs and write events to them or to reserve custom eventlogs for their use. For example Kaspersky antivirus is writing a custom eventlog named ``Kaspersky Event Log`` for their Security Center suite but also writes scan results to the "Security" Event log.

> Windows eventlogs are nowadays stored as ``.evtx`` files in the ``C:\Windows\System32\winevt\Logs\`` folder.
> These files are opened by the system at runtime and can't be accessed directly although you may copy them with administrative right to somewhere else without any hassle.

Windows eventlogs can be viewed through the Windows **Event Viewer**. The Event Viewer is the default application to open ``.evtx`` files and you can open also logfile from other windows systems this way. The Event Viewer will give you the possibility to search and filter the eventlogs as well as displaying you the content of the logfile. This is necessary since the log files in comparison to for example *nix are not human readable in their native format. Below is a screenshot of the Event Viewer with the ``Security`` eventlog opened.

![Screenshot: Windows Event Viewer in action](/assets/img/eventviewer.png)

<br/>

<br/>

## High-Level Structure of Windows eventlogs <a name="structure"></a>

We've already talked about different eventlogs and that we can view them using the Event Viewer. 

Each eventlog in the Event Viewer represents one ``.evtx`` file on your disk. The eventlog are grouped in folders and most of them will be in the ``Application and Services Logs/Microsoft/Windows`` folder. As seen in the screenshot above. When selecting one eventlog, all the log entries will be shown and when selected the corresponding ``EventData`` will be displayed. This is everything in the big box below the entries listing. In the example above the ``EventData`` is quite extensive there are other examples with way less information for us.

From most interest are the ``Event ID``, the ``Datetime`` and the ``EventData``. When vieweing logs from a different machine the ``Computer`` information may also become handy.

Lets get through them one by one:
* ``Event ID``: This field is more a less an identifier for log entries of the same type. The ``Security`` eventlog will hold many different information with different event IDs. But when you only want to see Account logons you may filter the log by the event ID **4624**. For logoff event you on the other hand have to search for ID **4647**. 

  > Always keep in mind that the same ``Event ID`` could hold different information in another eventlog. They are not reserved although Microsoft tries to not double them inside their own logs.

  Also there are some rare cases were events with the same event ID hold complete different information although in the same log. But this mostly only happens with 3rd party software using the eventlog as well.

* ``Datetime``: This is just the time when the event was written to the eventlog (mostly the time when the event occurred) in your local system time.

    > This will also display you the timestamps in your local system time when reviewing eventlogs from another system! So you may have to calculate time zone difference in these cases.

* ``EventData``: This is most crucial information since it presents you additional information. To say with our example from above the event ID **4624** only tells you that a account logon occurred but the information which account was authenticated, on which system and through which method (Console, Network, etc.) will only be visible through the event data.

  Most of the time and when you are viewing logs from your own machine the information will be human readable but contain some references you might not directly know. For example the event ID **4624** tells you the **Logon Type** but unfortunately just as numeric value. When you want to know what the values stand for you have to dig through the documentation (which is unfortunately not always that helpful). In our case the event is well documented an we can find on the [Microsoft Docs][windows-logon-types-link] that the Logon Type 2 stand for ``Interactive`` logon, which is the local GUI logon.

  > Even here is something special to keep in mind. The event data on disk is only represented as a few key-value pairs. Everything else will be added by windows as so called **Display information**. These information get installed together with the windows component that writes the eventlog. When you copy paste the eventlogs from one system to another and the viewing system does not have the information available you Event Data will not make much sense to you. You then have the possibility to manually install the component that brings the Display information with it or you could gracefully export the eventlog through the Event Viewer and choose to "Export with Display Information data". 
  
  See below for an example of an Active Directory Log viewed on a server without the ActiveDirectory component and the corresponding XML data of the event.

  ![Screenshot: AD Log viewed on a server without AD component](/assets/img/eventviewer_ad.png)

  ```
    <EventData>
    <Data>7329203334858325282</Data> 
    </EventData>
  ```

  And now after installing the ActiveDirectory component.

   ![Screenshot: AD Log viewed on a server with AD component](/assets/img/eventviewer_ad2.png)

<br/>

<br/>

## Use of Windows eventlogs <a name="useage"></a>

Windows eventlogs have the same uses as all other system and application log files. We will tackle some examples below for completeness sake.

### Troubleshooting

System and application as well as hardware event logs are used to identify issues on the system and resolve them properly.

The most commonly used eventlogs for this use case will be the general core eventlogs:
* System
* Security
* Application

And maybe some more but this is hard to specify without knowing the specific trouble:
* Microsoft-Windows-Group Policy
* Microsoft-Windows-Kernel-*
* Microsoft-Windows-Network Profile
* Application specific eventlogs (3rd party)


### Auditing

Logs are sometimes used for auditing and compliance mostly speaking of user actions in specific application and account usage. This will be mostly the Windows Security and System eventlogs.

Many auditing features of windows eventlogs are disabled by default. We will come to them in part three of this series.

### Forensics

Regarding forensic analysis we have to main targets. One will be identifying **user actions** and the other one will be identifying **malware persistence and actions done by the malware to the system**.

For both means the default windows eventlogs may only provide limited information when not configured properly. Although some of the most common events can be spotted. For example:
* Installed Services or Tasks
* Processes creation
* RDP Logins
* SMB connections
* WMI Consumer
* PowerShell execution
* Account misuse

These would be found in the following logs:
* System
* Security
* PowerShell
* Microsoft-Windows-Powershell
* Microsoft-Windows-WMI Activity
* Microsoft-Windows-TaskScheduler
* Microsoft-Windows-TerminalServices-*
* Microsoft-Windows-SmbClient Security
* Microsoft-Windows-WinRM Operational
* Microsoft-Windows-WLAN-AutoConfig Operational.evtx

> Many more information gets available when properly setting up eventlogs for forensic analysis. Then these logs can be your best friend when tracking changes made to a windows system and investigating an attackers work. 

<br />

---

<br />

That's it for now. As always when you have remarks or question drop me a mail or GitHub issue.

In the next part of the series I will talk you through configuring the log size and location of Windows eventlogs so and why this is important to do.

Best and stay healthy  
Tobias

[windows-eventlogs-overview-post]: {% post_url 2021-04-05-preparing-windows-eventlogs-for-forensics %}
[windows-logon-types-link]: https://docs.microsoft.com/de-de/windows/security/threat-protection/auditing/event-4624#logon-types-and-descriptions