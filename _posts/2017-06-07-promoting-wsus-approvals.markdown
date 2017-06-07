---
layout: post
title:  "Promoting WSUS approvals"
date: 2017-06-07 19:30:00
categories: devops
---

## WSUS release schedules

Like many organisations we have debated automated Windows Server Update Services
(WSUS) patch approval versus a more controlled staging/testing deployment
schedule in depth. We've decided on a tiered release schedule depending on how
critical we perceive the risk of leaving devices unpatched.

* Critical patches that we believe are serious threat to the business get
approved immediately.
* Server patches get approved immediately, but the servers are configured to
notify for install.
* All other patches get released to a 'Test PCs' group in WSUS.

After a period of testing all patches will be released to all PCs.

## WSUS approvals and PowerShell

In order to avoid manually re-approving all patches applied to Test PCs we can
use PowerShell to copy the approvals over to the 'All Computers' group. We'll
also remove all the specific approvals so that they switch back to 'Inherit'.

**Warning**: This process will find all updates approved for any group in the
last month and approve them for the All Computers group. You'll have to modify
this process if you need to keep update approvals separate for different groups.

1. Connect to the WSUS server

	```powershell
	$wsus = Get-WsusServer -Name wsus.server.name -UseSsl -portnumber 443
	```

1. Retrieve all updates from the last 30 days that are approved

	```powershell
	$date_to = Get-Date
	$date_from = $date_to.AddDays(-30)

	$updates = $wsus.GetUpdates('LatestRevisionApproved', $date_from, $date_to, $null, $null)
	```

1. Retrieve information about the 'All Computers' group

	```powershell
	$computer_groups = $wsus.GetComputerTargetGroups()
	$all_computers = $computer_groups | Where Name -eq 'All Computers'
	```

1. Move the approval for each update to 'All Computers', and remove the
sub-group approval.

	```powershell
	$updates | Where IsApproved -eq $true | Foreach-Object {
		$update = $_
		$update.title | Write-Host
		$approvals = $update.GetUpdateApprovals()
		$approvals | Where ComputerTargetGroupId -ne $all_computers.Id | %{
			"Removing approval for group $( ($computer_groups | Where Id -eq $_.ComputerTargetGroupId).Name )" | Write-Host
			$_.Delete()
		}
		if (!($approvals | Where ComputerTargetGroupId -eq $all_computers.Id)) {
			"Approving for all computers" | Write-Host
			$_.Approve('Install', $all_computers)
		}
	}
	```

All update approvals have now been moved and should roll out to everyone the
next time they check in to WSUS.
