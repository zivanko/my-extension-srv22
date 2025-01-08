# Windows Server 2022 Features Installation and Configuration Script
# This script installs and configures IIS, DNS, DHCP, and Remote Desktop Services

#Requires -RunAsAdministrator

# Function to check if feature installation was successful
function Test-FeatureInstallation {
    param (
        [string]$FeatureName
    )
    $feature = Get-WindowsFeature -Name $FeatureName
    return $feature.Installed
}

# Function to verify and set static IP
function Set-StaticIP {
    $adapter = Get-NetAdapter | Where-Object {$_.Status -eq "Up"}
    $currentIP = Get-NetIPAddress -InterfaceIndex $adapter.ifIndex -AddressFamily IPv4
    
    if ($currentIP.PrefixOrigin -ne "Manual") {
        Write-Host "Configuring static IP..." -ForegroundColor Yellow
        # Backup current IP configuration
        $currentIPAddress = $currentIP.IPAddress
        $currentPrefixLength = $currentIP.PrefixLength
        $currentGateway = (Get-NetRoute -InterfaceIndex $adapter.ifIndex -DestinationPrefix "0.0.0.0/0").NextHop

        # Remove existing IP configuration
        Remove-NetIPAddress -InterfaceIndex $adapter.ifIndex -AddressFamily IPv4 -Confirm:$false
        Remove-NetRoute -InterfaceIndex $adapter.ifIndex -AddressFamily IPv4 -Confirm:$false

        # Set static IP
        New-NetIPAddress -InterfaceIndex $adapter.ifIndex -AddressFamily IPv4 `
            -IPAddress $currentIPAddress -PrefixLength $currentPrefixLength `
            -DefaultGateway $currentGateway

        # Configure DNS
        Set-DnsClientServerAddress -InterfaceIndex $adapter.ifIndex -ServerAddresses $currentIPAddress
    }
}

# Install Windows Features
Write-Host "Installing required Windows Features..." -ForegroundColor Green

# Configure Static IP first
Set-StaticIP

# 1. IIS Installation and Configuration
Write-Host "Installing IIS and related features..." -ForegroundColor Yellow
$iisFeatures = @(
    'Web-Server',
    'Web-Common-Http',
    'Web-Default-Doc',
    'Web-Dir-Browsing',
    'Web-Http-Errors',
    'Web-Static-Content',
    'Web-Http-Logging',
    'Web-Stat-Compression',
    'Web-Filtering',
    'Web-Mgmt-Console',
    'Web-Mgmt-Tools'
)

foreach ($feature in $iisFeatures) {
    Install-WindowsFeature -Name $feature -IncludeManagementTools
}

# Configure IIS Default Website
Import-Module WebAdministration
Set-ItemProperty 'IIS:\Sites\Default Web Site' -Name serverAutoStart -Value $true

# 2. DNS Server Installation and Configuration
Write-Host "Installing DNS Server..." -ForegroundColor Yellow
Install-WindowsFeature -Name DNS -IncludeManagementTools

# Configure DNS Server
Add-DnsServerPrimaryZone -Name "internal.local" -ZoneFile "internal.local.dns"
Add-DnsServerForwarder -IPAddress "8.8.8.8" -PassThru

# 3. DHCP Server Installation and Configuration
Write-Host "Installing DHCP Server..." -ForegroundColor Yellow
Install-WindowsFeature -Name DHCP -IncludeManagementTools

# Get current IP address for DHCP configuration
$currentIP = (Get-NetIPAddress -AddressFamily IPv4 | Where-Object {$_.PrefixOrigin -eq "Manual"}).IPAddress

# Configure DHCP Server with current IP
Add-DhcpServerv4Scope -Name "MainScope" -StartRange "192.168.1.100" -EndRange "192.168.1.200" `
    -SubnetMask "255.255.255.0" -Description "Main DHCP Scope"

Set-DhcpServerv4OptionValue -ScopeId "192.168.1.0" -Router $currentIP `
    -DnsServer $currentIP -DnsDomain "internal.local"

# Authorize DHCP server in Active Directory
Add-DhcpServerInDC

# 4. Remote Desktop Services Installation (Without Hyper-V dependency)
Write-Host "Installing Remote Desktop Services..." -ForegroundColor Yellow
Install-WindowsFeature -Name Remote-Desktop-Services,RDS-RD-Server,RDS-Connection-Broker,RDS-Web-Access `
    -IncludeManagementTools

# Configure Remote Desktop Services
Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' `
    -Name "fDenyTSConnections" -Value 0

Enable-NetFirewallRule -DisplayGroup "Remote Desktop"

# Verify installations
Write-Host "`nVerifying installations..." -ForegroundColor Green
$features = @{
    'Web-Server' = 'IIS'
    'DNS' = 'DNS Server'
    'DHCP' = 'DHCP Server'
    'Remote-Desktop-Services' = 'Remote Desktop Services'
}

foreach ($feature in $features.Keys) {
    $status = Test-FeatureInstallation -FeatureName $feature
    if ($status) {
        Write-Host "$($features[$feature]) installed successfully." -ForegroundColor Green
    } else {
        Write-Host "$($features[$feature]) installation failed!" -ForegroundColor Red
    }
}

Write-Host "`nConfiguration complete. Please review the following:" -ForegroundColor Yellow
Write-Host "1. Verify IIS is accessible at http://localhost"
Write-Host "2. Configure DNS forwarders if needed"
Write-Host "3. Review DHCP scope settings"
Write-Host "4. Test Remote Desktop connectivity"
Write-Host "5. Check Windows Firewall rules for all services"
