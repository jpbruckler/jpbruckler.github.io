---
date: 2015-04-01 13:16:17
title: "Checking Local Administrators with ConfigMgr Configuration Baselines"
layout: single
tags: [PowerShell, SCCM, ConfigMgr, Collections, Compliance]
---

The other day I received a request from our asset management department. They
wanted to know if I could get them a list of what we call 'FLEX' workstations.
These workstations basically have a specific local administrator on them.

"Sure." I said. "No problem." I said.

First step was figuring out if hardware inventory captures local administrators.

No dice.

Though there is this useful MNSCUG post from the ineffable Sherry Kissinger to
inventory [all members of all local groups](http://mnscug.org/blogs/sherry-kissinger/244-all-members-of-all-local-groups-configmgr-2012)
in ConfigMgr 2012.

Being a good Windows admin, I'm inherently lazy, and Sherry's solution, while
really very good, just seemed like _such_ work. Well, not really - but a custom
hardware inventory does mean that when the site is updated you'll have to remember
that you made that customization. No problem because you document every single
change you make to the system, right?

Besides, I wasn't looking for every member of a local group. I was looking for a
specific member of a local group. This seemed like a job for PowerShell and a Configuration Item!

I created a new Configuration Item in the console, and used the PowerShell script
below as the Setting:

{% highlight powershell %}
$LocalGroup = 'Administrators'
$AccountRegEx = '^flex.*'
$FewAccount = $false
$Group = [ADSI] "WinNT://$($env:ComputerName)/$LocalGroup"

$Members = @($Group.Invoke('Members'))

ForEach ($Member in $Members)
{
    $Name = $Member.GetType().InvokeMember('Name','GetProperty',$null,$Member,$null)
    $ADSpath = $Member.GetType().InvokeMember('ADSPath','GetProperty',$null,$Member,$null)


    if ($Name -match $AccountRegEx -and $ADSPath -match $env:ComputerName)
    {
        $FewAccount = $true
    }
}

if ($FewAccount)
{
    Write-Host 'Compliant'
}
else
{
    Write-Host 'Non-Compliant'
}
{% endhighlight %}

The `$AccountRegEx` variable is the one you'll want to change to meet your needs.
In this case, the regex matches account names that begin with _flex_. If you need
help creating a regex, try [RegExr.com](http://regexr.com/), or the
[Regular-Expressions.info](http://www.regular-expressions.info/) site.

## Setting Up A Configuration Item
1. In the admin console, head to Assets and Compliance, then Compliance Settings, and finally select **Configuration Items**
2. Click _Create Configuration Item_ in the ribbon, give your CI a name, leave the configuration item type set to Windows, then click **Next**
![Create Configuration Item Wizard ](/assets/ConfigurationItem-001.png)
3. On the next screen, select the Operating Systems this CI should run against, then click **Next**
![Create Configuration Item Wizard ](/assets/ConfigurationItem-002.png)
4. On the _Specify settings for this operating system_ page, click **New**
5. Give the setting a name, then change the setting type to **Script**. Ensure Data type is set to **String** then click the **Add Script** button to add a script.  ![Create Configuration Item Script Setting](/assets/ConfigurationItemSetting-001.png)
6. Paste in the script above and modify to suit your needs, then click **OK** ![Create Configuration Item Script Setting](/assets/ConfigurationItemSetting-002.png)
7. Click the _Compliance Rules_ tab, then click **New** to create a new rule ![Create Configuration Item Script Setting](/assets/ConfigurationItemSetting-003.png)
8. Give the rule a meaning name and a description. The _Rule Type_ should be set to **Value**, and the value to check for is **Compliant**. Click **OK** to create the rule and close the dialogue. ![Create Configuration Item Script Setting](/assets/ConfigurationItemSetting-004.png)
9. Back on the _Create Setting_ page, click **OK** to continue ![Create Configuration Item Wizard ](/assets/ConfigurationItem-004.png)
10. Since we've already defined the setting and the compliance rule, just click **Next** through the rest of the wizard. ![Create Configuration Item Wizard ](/assets/ConfigurationItem-010.png)

You'll need to create a configuration baseline and add this CI to the baseline,
then deploy it to a collection in order to start gathering data. Technet has very
complete [documentation](https://technet.microsoft.com/en-us/library/bb632510.aspx)
on creating and deploying Configuration Baselines, so I'm not going to duplicate
the work here.

In order to create a collection based on the results, the baseline **must** be
deployed to a collection. Once the baseline is deployed, you'll be able to continue
with the rest of the steps.

## Creating Collection Queries
The next step in the process is creating a collection for all the compliant
workstations. This is actually a lot easier than I was expecting it to be.

1. In _Assets and Compliance/Device Collections_ create a new collection and name it something meaningful. In this example I'm naming the collection **FLEX Enterprise Workstations_DQLX** because where I work we use suffixes on collection names to visually indicate the general purpose of the collection. The limiting collection is set to _All Workstations_ (a custom query based collection based on Operating System type) and click **Next** ![Collection Creation](/assets/FLEXCollection-001.png)
2. On the next screen, click the **Add Rule** button and select **Query Rule**
3. On the _Query Rule Properties_ page, give the rule a name, then click **Edit Query Statement**

   ![Collection Creation](/assets/FLEXCollectionRule-001.png)

4. Click the _Criteria_ tab, then add a new criteria with the little button that looks like a starburst (![](/assets/NewCriteriaBtn.png))
5. On the _Criterion Properties_ page, click the **Select** button. ![Collection Creation](/assets/FLEXCollectionRule-003.png)
6. For the criterion selection, use the following values:
   * **Attribute Class:** Configuration Item Compliance State
   * **Attribute:** Localized Display Name

   ![Collection Creation](/assets/FLEXCollectionRule-004.png)

 7. Clicking the **Value** button will display a list of deployments. You can select the Configuration Baseline deployment from the list, or you can just type it in. As long as the value for the query criterion matches the localized display name (what you see in the console when looking at baselines) of the configuration baseline, it will work. ![Collection Creation](/assets/FLEXCollectionRule-005.png)
 8. Add another Criterion, use the following values:
    * **Attribute Class:** Configuration Item Compliance State
    * **Attribute:** Compliance State Name

    ![Collection Creation](/assets/FLEXCollectionRule-006.png)
9. Set the _Value_ to **Compliant** ![Collection Creation](/assets/FLEXCollectionRule-007.png)
10. When returned to the Criteria tab, click **OK** to create the query, then **OK** again to commit the rule to the collection.
11. Set the collection refresh schedule to whatever works for your environment, then click **Next** through the rest of the Wizard to create the collection.

As workstations receive their policy and run the compliance rule, they will start
reporting compliance status, and the collection will start populating with workstations
that match our compliance rule.

To report back to the asset management team, I just setup a reporting subscription
for the _All resources in a collection_ built-in report.

This technique could also be used to report back compliance on the local administrators
group memberships for workstations with some modification. If your workstations
should only ever have certain groups/accounts as local administrators, the script
could check the current membership against a list of expected members, then report
compliance back based on that check.

If there's any interest I'll write another post with the code to do just that.
