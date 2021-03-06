# Write-up [```LEHACK19```](https://lehack.org/)  
Write-up for the Active Directory Lab I have created for [Akerva](https://www.akerva.com) exhibition stand @ leHACK19 (Paris)

<p align="center"><img src="https://i.ibb.co/tD63L8k/D-8-MFc-FXUAAR6-EI.jpg"></p>

## Index

| Title          | Description    |
| -------------- |:-------------- |
| [About](#About)    | About the challenge |
| [Reconnaissance](#Reconnaissance)    | Reconnaissance |
| [Initial Foothold](#Initial-Foothold)    | Tomcat's host-manager exploitation |
| [Procdump lsass](#Procdump-lsass)    | Procdump lsass process to retrieve credentials |
| [Cartography with Bloodhound](#Cartography-with-Bloodhound)    | AD Cartography with Bloodhound |
| [Browsing Shares](#Browsing-Shares)    | Browsing shares to retrieve credentials |
| [RBCD Exploitation](#RBCD-Exploitation)    | RBCD Exploitation |
| [Looting](#Looting)    | Looting juicy information|

## About
WonkaChall 2 created by Akerva for leHACK19 (Paris) is divided into three parts:
* [part 1 (AWS)](https://akerva.com/blog/wonkachall-akerva-lehack-2019-write-up-part-1-web/)
* [part 2 (Active Directory)](https://akerva.com/blog/wonkachall-2-lehack-2019-write-up-part-2-windows/) <= This write-up deals with the Active Directory part I have created.
* [part 3 (Linux)](https://akerva.com/blog/wonkachall-2-lehack-2019-write-up-part-3-linux/)

## Reconnaissance
In the previous part 1, we retrieved a VPN configuration file on a AWS bucket. We can now connect to the wonka internal network with the VPN config file (wonka_internal.ovpn).

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/1-1.png"></p>

When connected, we want to determine what networks we have access to. For that purpose, we can simply look at routes pushed by our VPN.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/2-1.png"></p>

172.16.42.0/24 subnetwork looks interesting. Let's see what is alive in it by doing a quick nmap ping.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/3-1.png"></p>

Three machines looks to be UP:
* 172.16.42.5
* 172.16.42.11
* 172.16.42.101 

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/4-1.png"></p>

Juding from the listening services, these three machines look to be Windows, and one seems to be a domain controller. We can note the open port 8080 on 172.16.42.11. Could it be a Tomcat, easy prey?

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/5-1.png"></p>

We confirm these three machines are part of a domain FACTORY.LAN.

| Hostname | IP address | OS |
| -------- |:-----------|:-----------|
| DC01-WW2 | 172.16.42.5 | Windows Server 2019 |
| SRV01-INTRANET | 172.16.42.11 | Windows Server 2019 |
| PC01-DEV | 172.16.42.101	| Windows 10 |

## Initial Foothold
By reaching SRV01-INTRANET's port 8080 with a web browser, we face what looks like an internal wiki of the Wonka organization.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/6-1.png"></p>

An interesting schema is freely available and give a good sight of the company infrastructure.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/7-1.png"></p>

We can learn the existence of a subnetwork 172.16.69.0/24 with several machines and that PC01-DEV is the computer of a dev/admin.
By reaching an inexistant page, the error betrays the web service is indeed a Tomcat. We are aware Tomcat on Windows are often launched with NT AUTHORITY\SYSTEM privileges.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/8-1.png"></p>

We then try to reach the Tomcat's manager page to compromise the machine in case of default credentials, but the manager is unavailable.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/9-1.png"></p>

We launch a GoBuster, and find out that the host-manager page is active.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/10-1.png"></p>

We go on the web page and face to a basic authentication form. We try tomcat's default credentials « tomcat / tomcat ».

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/11-1.png"></p>

The credentials are valid allowing us to reach the host-manager.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/12-1.png"></p>

The host-manager exploitation is a simple variant of the manager exploitation. There are many resources on the internet explaining how to exploit default credentials on a Tomcat's host-manager in order to take control of the underlying server.
The idea here is to force the Tomcat's host manager to look for a war application on a smb share we control, in order to deploy it. Our war is a webshell allowing us to execute commands as NT AUTHORITY\SYSTEM. From here, we could be able to perform interesting actions like extracting credentials from memory.

Another thing to take into consideration is antivirus detections. A meterpreter war generated by msfvenom will be detected by the server's antivirus, a simple Mimikatz too. A solution is to use a home made war and a procdump from sysinternals to dump lsass process, in order to extract passwords locally with Mimikatz.

Let's put the war and ProcDump into a directory.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/13-1.png"></p>

Let's start an Impacket's SMB server in our directory.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/14-1.png"></p>

Then, we can write our SMB server's UNC path on the host-manager, \\\10.8.0.42\aka.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/15-1.png"></p>

akerva.factory.lan must point to the Tomcat's IP address. We edit our /etc/hosts to reflect that.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/16-1.png"></p>

We click on Add et see the Tomcat retrieving and deploying our war directly on our SMB share.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/17-1.png"></p>
<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/18-1.png"></p>

Our application (web shell) is now available on the host-manager.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/19-1.png"></p>

We can reach our webshell via: http://akerva.factory.lan:8080/akerva/cmd.jsp. From here, we can enter commands like whoami to verify we are indeed NT AUTHORITY\SYSTEM.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/20-1.png"></p>

The flag 5 is on the adminServer's Desktop.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/21-1.png"></p>

We then dump the lsass process thank's to Sysinternal's ProcDump tool.

## Procdump lsass
<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/22-1.png"></p>

We bring back our dump.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/23-1.png"></p>
<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/24-1.png"></p>

And we parse it locally to retrieve credentials..

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/25-1.png"></p>

Flag 6 is the SHA256 of the adminServer's password.

## Cartography with Bloodhound
Now we have a domain account, we can use Bloodhound to cartography the domain. We carefully take the latest version of Bloodhound and Sharphound. No need to add a Windows machine to the domain to launch Sharphound.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/26-1.png"></p>

Our Bloodhound does not show domain admin shortest path but a primitive « AddAllowedToAct » between domain account « SvcJoinComputerToDom » and the domain controller catch our attention.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/27-1.png"></p>

After googling, we gather several resources on Resources-Based Constrained Delegation :
* https://posts.specterops.io/a-case-study-in-wagging-the-dog-computer-takeover-2bcb7f94c783
* https://binged.it/2kaWdnx
* https://beta.hackndo.com/resource-based-constrained-delegation-attack/

Very succintly, the idea is to rewrite the « AllowedToActOnBehalfOfOtherIdentity » property of a machine with a structure based on a SPN account's SID. Then, this SPN account should be able to impersonate any users on the aimed machine, and so, compromise it.

Two conditions are required for this RBCD exploitation:
* To control a SPN account
* To be able to rewrite the « AllowedToActOnBehalfOfOtherIdentity » property of the domain controller

For the first condition, a simple technic is to join a machine to the domain. Indeed, this joined machine will have SPN. By default on Active Directory environment, every domain user can join up to 10 machines to the domain. But it is not the case here because the domain administrator has hardened this configuration. However, a service account SvcJoinComputerToDom seems to have this privilege.

For the second point, our Bloodhound clearly states that SvcJoinComputerToDom has this right.

So, we have to find this SvcJoinComputerToDom. We can imagine this service account was created by the domain administrator in order to delegate the right to join machines to the domain to the adminServer and adminWorkstation, without giving them the domain administrator privilege.

## Browsing Shares
We check to what our adminServer account has access to.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/28-1.png"></p>

A share « provisioning » looks to be available for adminServer. From this folder, we can retrieve SvcJoinComputerToDom's credentials.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/29-1.png"></p>

And aldo flag 7.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/30-1.png"></p>

## RBCD Exploitation
Everything is in place for our RBCD exploitation. We will need four tools (PowerView, Powermad, Rubeus and Mimikatz).
* https://github.com/PowerShellMafia/PowerSploit/blob/dev/Recon/PowerView.ps1
* https://github.com/Kevin-Robertson/Powermad/blob/master/Powermad.ps1
* https://github.com/GhostPack/Rubeus
* https://github.com/gentilkiwi/mimikatz

First we start by importing PowerView and Powermad, then we specify the domain controller as target. We retrieve the SID of the SvcJoinComputerToDomain account.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/31-1.png"></p>

Then we can join a machine to the domaine with SvcJoinComputerToDom. We retrieve the joined machine SID, and craft the structure based on that SID. Then we rewrite the domain controller's « AllowedToActOnBehalfOfOtherIdentity » property with that structure.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/32-1.png"></p>

Now, we can impersonate the domain administrator account on the domain controller with our joined machine account.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/33-1.png"></p>

Here the Rubeus output.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/34.png"></p>
<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/35.png"></p>
<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/36.png"></p>
<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/37.png"></p>
<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/38.png"></p>

Our ticket is imported, let's DCSync with Mimikatz the NTLM hash of Administrator and Krbtgt (flag 8).

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/39.png"></p>

## Looting
Since we have retrieved a domain administrator NTLM hash, the domaine is compromised.
Since the domain is compromised, we can try to explore machines not exploited yet. For example the dev/admin machine which may hide secrets about the dev subnetwork.
For that purpose, we use PSExec with the Administrator NTLM hash but it fails. Probably because Administrator is a domain « Protected Users ».

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/40.png"></p>

We think and decide to DCSync the « adminWorkstation » account and try again with PsExec. It works. After enumeration, we can find a WinSCP shortcut on the adminWorkstation's Desktop.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/41.png"></p>

We are aware that WinSCP saves credentials into register, in a reversible manner. We can retrieve these credentials by simple executing the CrackMapExec's Invoke_sessiongopher module.

<p align="center"><img src="https://akerva.com/wp-content/uploads/2019/07/42.png"></p>

These credentials allows us to go deeper in the information system of the company. We can now explore a new subnetwork, the dev one... (part 3 of WonkaChall 2).

## The end  
Thanks for reading this.  
I hope you enjoyed it as much as I enjoyed making this Active Directory lab.  
#
*Created by [Lydéric Lefebvre](https://twitter.com/lydericlefebvre)*
