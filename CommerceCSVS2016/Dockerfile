# escape=`

#Depending on the operating system of the host machines(s) that will build or run the containers, the image specified in the FROM statement may need to be changed.
#For more information, please see https://aka.ms/containercompat 

#FROM mcr.microsoft.com/dotnet/framework/aspnet:4.7.2-windowsservercore-ltsc2019
#ARG source
#WORKDIR /inetpub/wwwroot
#COPY ${source:-obj/Docker/publish} .

# ================================================================================================
# Builder Stage
# https://blogs.msdn.microsoft.com/dotnet/2018/06/13/using-net-and-docker-together-dockercon-2018-update/
# ================================================================================================

FROM mcr.microsoft.com/dotnet/framework/sdk:4.7.2-windowsservercore-ltsc2019 as builder

# Build Visual Studio Solution
# TODO: Optimize Nuget restoration into layers
SHELL ["powershell"]

#RUN mkdir C:\site

WORKDIR C:\site

COPY . C:\site

RUN nuget restore

RUN msbuild /p:Configuration=Release C:\site\CommerceCSVS2016\CommerceCSVS2016.csproj

# ================================================================================================
# Runtime Stage
# ================================================================================================

#FROM microsoft/aspnet:4.7.2-windowsservercore-ltsc2016
#FROM microsoft/aspnet:4.7.2-windowsservercore-1803
FROM mcr.microsoft.com/dotnet/framework/aspnet:4.7.2-windowsservercore-ltsc2019
#
EXPOSE 80

WORKDIR C:\site
#
## Setup Operating System
#
## Enable IIS Remote Administration
RUN net user localadmin Qwerty123456 /add; `
	net localgroup Administrators localadmin /add; `
	Install-WindowsFeature Web-Mgmt-Service; `
	Add-WindowsFeature Web-Server; `
    Add-WindowsFeature NET-Framework-45-ASPNET; `
    Add-WindowsFeature Web-Asp-Net45; `
    Add-WindowsFeature Web-Http-Tracing; `
	New-ItemProperty -Path "HKLM:\software\microsoft\WebManagement\Server" -Name "EnableRemoteManagement" -Value 1 -Force;

# Setup IIS Server
RUN New-WebAppPool "GMSAAppPool"; `
	# Configure root AppPool to run as LocalSystem 
	Set-ItemProperty `
		-Path "IIS:\AppPools\GMSAAppPool" `
		-Name "processModel" `
		-Value @{identitytype=0}; `
	# Replace Default Web Site
	Remove-WebSite -Name 'Default Web Site'; `
	New-WebSite `
		-Name "Site" `
		-Port 80 `
		-PhysicalPath "C:\site" `
		-ApplicationPool "GMSAAppPool"; `
	# Configure for Anonymous Authentication
	Set-WebConfigurationProperty `
		-Location "Site" `
		-PSPath IIS:\ `
		-Filter "system.webServer/security/authentication/anonymousAuthentication" `
		-Name "enabled" `
		-Value $true; `
	# Configure monitoring
	Set-WebConfigurationProperty `
		-pspath 'MACHINE/WEBROOT/APPHOST' `
		-filter "system.applicationHost/sites/siteDefaults/logFile" `
		-name "logTargetW3C" -value "File,ETW";


## Copy compiled files from Builder Stage
COPY --from=builder C:\site\CommerceCSVS2016 c:\site\
#
## Create Web Applications and configure authentication
#
## Enable verbose logging for kerberos in the Windows Event Viewer
RUN New-ItemProperty `
	-Path HKLM:\SYSTEM\CurrentControlSet\Control\Lsa\Kerberos\Parameters `
	-Name LogLevel `
	-PropertyType DWord `
	-Value 1 `
	-Force

# Check application health against anonymous endpoint
HEALTHCHECK CMD powershell -command `  
    try { `
     $response = iwr http://localhost:80 -UseBasicParsing; `
     if ($response.StatusCode -eq 200) { return 0} `
     else {return 1}; `
    } catch { return 1 }

# Start IIS Remote Management and monitor IIS
#ENTRYPOINT	Start-Service WMSVC; `
#		C:\\ServiceMonitor.exe w3svc;

# Start the metric hub entrypoint
ENTRYPOINT	C:\\site\\metrichub\\runtime\\MetricHub.Entrypoint.exe;