---
title:  "List Available WMI Namespaces and Classes"
date:   2014-09-21 08:08:23
tags: [PowerShell, WMI]
---

Just a quick post to show how you can list the available WMI Namespaces and Classes
on the local workstation, and also on remote computers.

Listing Namespaces
----
To list the available WMI namespaces, you can use the code below. Since Get-WmiObject
supports the ComputerName parameter, you can use the same command to retrieve the
namespaces on a remote computer.

{% highlight powershell %}
Get-WmiObject -Namespace root -Class __Namespace | Select-Object Name
{% endhighlight %}

Listing Classes in a Namespace
---
Listing classes is really straightforward once you know the namespace whose class you want to list.

{% highlight powershell %}
Get-WmiObject -Namespace root\default -List
{% endhighlight %}
