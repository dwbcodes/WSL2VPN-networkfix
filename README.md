# WSL2VPN-networkfix Fixes the networking whilst using Global Protect and or Pulse Secure.

## Problem.

Global Protect and Pulse Secure when enabled add their network interface to the network route table with a higher priority than the network route for the WSL network.

This process will need to be run every time you connect to your VPN, although it can be scripted following the instructions below.

WSL Network Interface.

    eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1400 qdisc mq state UP group default qlen 1000
    link/ether 00:15:5d:xx:xx:xx brd ff:ff:ff:ff:ff:ff
    inet 172.28.21.64/20 brd 172.28.31.255 scope global eth0
       valid_lft forever preferred_lft forever

Before VPN is connected.

      172.28.16.0    255.255.240.0         On-link       172.28.16.1   5256
      172.28.16.1  255.255.255.255         On-link       172.28.16.1   5256

After the VPN is connected.

      172.28.16.0    255.255.240.0         On-link       172.28.16.1   5256
      172.28.16.0    255.255.240.0         On-link      10.183.12.31      1
      172.28.16.1  255.255.255.255         On-link       172.28.16.1   5256

Script to remove the VPN network route.

      .\wsl2vpn-fix.ps1
        WSL Network Mask:
        VPN Interface Number: 23
        IP Address: 172.28.16.1
        Network Address:
        VPN Description: PANGP Virtual Ethernet Adapter Secure

        Route for network 172.28.16.0/20 with interface number 23 found.
        Deleting route for network address 172.28.16.0 with interface number 23.
        Deleting route for network address 172.28.16.0 with interface number 23.

After the script is run.

      172.28.16.0    255.255.240.0         On-link       172.28.16.1   5256
      172.28.16.1  255.255.255.255         On-link       172.28.16.1   5256

## Automating the process.

Global Protect provides a way to run scripts after the VPN is connected.
  - https://docs.paloaltonetworks.com/globalprotect/9-1/globalprotect-admin/globalprotect-apps/deploy-app-settings-transparently/deploy-app-settings-to-windows-endpoints/deploy-scripts-using-the-windows-registry


1. Create a Scheduled Task as Administrator named "WSL-reset-network".
    - Action / Start a program: wsl.exe powershell.exe -Command "$(cat /home/dave/git/dwbconsulting/WSL2VPN-networkfix/wsl2vpn-fix.ps1)"
    - Ensure the task is "Run with Highest Permissions"
2. Create a registry post connect command for Global Connect
```    
$registryPath = "HKLM:\SOFTWARE\Palo Alto Networks\GlobalProtect\Settings\post-vpn-connect"
$scriptPath = "schtasks /run /tn WSL-reset-network"
#$scriptPath = "\\wsl$\Ubuntu-22.04\home\dave\git\dwbconsulting\WSL2VPN-networkfix\wsl2vpn-fix.ps1"
if (!(Test-Path $registryPath)) {
  New-Item -Path $registryPath -Force
}
Set-ItemProperty -Path $registryPath -Name "command" -Value $scriptPath

```
3. Test the task works from a non privileged user.
    - Windows
      - schtasks /run /tn "WSL-reset-network"
    - WSL Linux
      - schtasks.exe /run /tn "WSL-reset-network"


## Prerequisites.

* You should ensure you have a working WSL2 instance with a functioning network prior running this to fix your VPN routes.
* System Tested on.

    ```
    /etc/wsl.conf
    [network]
    generateHosts = false
    generateResolvConf = true
    #
    $USERPROFILE\.wslconfig
    [wsl2]
    guiApplications=true
    dnsProxy=false
    dnsTunneling=true
    [experimental]
    useWindowsDnsCache=true 

    wsl -v   
    WSL version: 2.0.14.0
    Kernel version: 5.15.133.1-1
    WSLg version: 1.0.59
    MSRDC version: 1.2.4677
    Direct3D version: 1.611.1-81528511
    DXCore version: 10.0.25131.1002-220531-1700.rs-onecore-base2-hyp
    Windows version: 10.0.22000.2777

    wsl -l -v
    NAME            STATE           VERSION
    Ubuntu-22.04    Running         2
    ```


