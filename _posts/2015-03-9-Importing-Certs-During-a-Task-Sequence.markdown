---
date: 2015-03-09 23:40:17
title: "Importing Certs During a Task Sequence"
tags: [PowerShell, WSUS, MDT]
category:
---

I like to try to keep IT Security happy. Happy security guys means more cooperation. Sometimes though, keeping those guys happy is a real PITA.

We've recently started getting serious about separating engineering and ops tasks where I work (a good thing) which meant that our in-house build lab needed to move off to a production server, rather than a VM that sits on a borrowed HP server in our lab.

Since the production server already had IIS and a certificate, and since I needed a WSUS server for the MDT builds, and since I love keeping my Security bros happy, I thought it would be a great idea to enable HTTPS WSUS traffic.

[TechNet](https://technet.microsoft.com/en-us/library/bb633246.aspx) has a great article on enabling all the pieces you need to get going with SSL on your WSUS server.

If you're like me, your server certificate was issued by your domain's PKI infrastructure (or maybe you aren't like me and have a self signed certificate). You probably won't think anything of this until you go to build your next gold image and the client fails to contact your WSUS server to get updates. Heck, you might even get an error in ZTIUpdates.log along the lines of **80072F8F**.

Eventually it will dawn on you that you're trying to use a certificate issued from an authority that your non-domain joined VM (you're capturing on VMs, right?) doesn't know from Adam. The solution is, of course, to import the certificates.

Being a lover of all things PowerShell, I figured there had to be a way to import certs using PowerShell. What I found was that there's no Microsoft Cmdlet for importing certificates, and being in a Task Sequence, I didn't want to look for a module that provided what I needed. So, I hit up Google to search for ways to import certificates in .NET, which led me to the [X509Certificate](https://msdn.microsoft.com/en-us/library/System.Security.Cryptography.X509Certificates.X509Certificate(v=vs.110).aspx) class on MSDN.

I'll spare you the frustration of figuring out which methods and classes to use to get it all working, and instead just dump the function on you:

{% highlight powershell %}
Function Import-Certificate
{
    param(
        [string] $Path,
        [System.Security.Cryptography.X509Certificates.StoreName] $StoreName,
        [System.Security.Cryptography.X509Certificates.StoreLocation] $StoreLocation
    )

    process
    {
        $Cert = New-Object System.Security.Cryptography.X509Certificates.X509Certificate2
        $Cert.Import($Path)

        $Store = New-Object System.Security.Cryptography.X509Certificates.X509Store($StoreName,$StoreLocation)
        $Store.Open("MaxAllowed")
        $Store.Add($Cert)
        $Store.Close()
    }
}
{% endhighlight %}

Usage is basically:

{% highlight powershell %}
Import-Certificate -Path .\RootCA.cer -StoreName Root -StoreLocation LocalMachine
Import-Certificate -Path .\WSUS.cer -StoreName TrustedPublisher -StoreLocation LocalMachine
{% endhighlight %}

Save the function and the 2 calls to a file in %SCRIPTROOT%. Place your certs somewhere safe, then update the `-Path` parameter to the path where you've stored your certs. In your task sequence, before connecting to your WSUS server, add a 'Run Powershell Script' step, giving it the name of the script to run.

That's it, when the Task Sequence runs, it will import the certs into the appropriate stores so that when your client connects to the WSUS server it will trust the SSL certificate being presented.
