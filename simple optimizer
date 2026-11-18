
# ==========================================================
#                  FPS AND PING OPTIMIZER
# ==========================================================

$ErrorActionPreference = "Continue"

$ApplyNagleTweaks = $true

$IPv4MTUTargets = @(
    "1.1.1.1",
    "8.8.8.8"
)

# ==========================================================
# ADMIN CHECK
# ==========================================================

$IsAdmin = ([Security.Principal.WindowsPrincipal](
    [Security.Principal.WindowsIdentity]::GetCurrent()
)).IsInRole(
    [Security.Principal.WindowsBuiltInRole]::Administrator
)

if (-not $IsAdmin) {
    Write-Host "Run this script as Administrator." -ForegroundColor Red
    Read-Host "Press Enter to exit"
    exit
}

Write-Host ""
Write-Host "Starting FPS AND PING OPTIMIZER" -ForegroundColor Cyan

# ==========================================================
# HELPER FUNCTIONS
# ==========================================================

function Set-RegDword {
    param(
        [string]$Path,
        [string]$Name,
        [long]$Value
    )

    try {
        if (-not (Test-Path $Path)) {
            New-Item -Path $Path -Force | Out-Null
        }

        New-ItemProperty `
            -Path $Path `
            -Name $Name `
            -Value ([uint32]$Value) `
            -PropertyType DWord `
            -Force | Out-Null
    }
    catch {
        Write-Host "Registry setting failed: $Name" -ForegroundColor Yellow
    }
}

function Add-RegDwordIfMissing {
    param(
        [string]$Path,
        [string]$Name,
        [long]$Value
    )

    try {
        if (-not (Test-Path $Path)) {
            New-Item -Path $Path -Force | Out-Null
        }

        $Existing = Get-ItemProperty `
            -Path $Path `
            -Name $Name `
            -ErrorAction SilentlyContinue

        if ($null -eq $Existing) {
            New-ItemProperty `
                -Path $Path `
                -Name $Name `
                -Value ([uint32]$Value) `
                -PropertyType DWord `
                -Force | Out-Null
        }
    }
    catch {}
}

function Invoke-Netsh {
    param([string[]]$Arguments)

    try {
        & netsh.exe @Arguments | Out-Null
    }
    catch {}
}

# ==========================================================
# RAM DETECTION
# ==========================================================

Write-Host "Detecting installed RAM" -ForegroundColor Cyan

try {
    $RAMBytes = (Get-CimInstance Win32_ComputerSystem -ErrorAction Stop).TotalPhysicalMemory
    $RAMGB = [Math]::Ceiling($RAMBytes / 1GB)

    Write-Host "Detected RAM: $RAMGB GB" -ForegroundColor Green
}
catch {
    Write-Host "RAM detection failed. RAM-based settings skipped." -ForegroundColor Red
    $RAMGB = 0
}

# ==========================================================
# MEMORY COMPRESSION
# ==========================================================

Write-Host "Configuring memory compression" -ForegroundColor Cyan

if ($RAMGB -gt 0) {
    try {
        if ($RAMGB -le 16) {
            Enable-MMAgent -MemoryCompression -ErrorAction Stop
            Write-Host "Memory compression enabled." -ForegroundColor Green
        }
        else {
            Disable-MMAgent -MemoryCompression -ErrorAction Stop
            Write-Host "Memory compression disabled." -ForegroundColor Green
        }
    }
    catch {
        Write-Host "Memory compression configuration failed: $($_.Exception.Message)" -ForegroundColor Yellow
    }
}

# ==========================================================
# SVCHOST SPLIT THRESHOLD - RAM BASED
# ==========================================================

Write-Host "Configuring SvcHostSplitThresholdInKB" -ForegroundColor Cyan

if ($RAMGB -gt 0) {

    switch ($RAMGB) {
        { $_ -le 8 } {
            $SvcHostValue = 8388608
            break
        }

        { $_ -le 16 } {
            $SvcHostValue = 16777216
            break
        }

        { $_ -le 32 } {
            $SvcHostValue = 33554432
            break
        }

        default {
            $SvcHostValue = 67108864
            break
        }
    }

    $ControlPath = "HKLM:\SYSTEM\CurrentControlSet\Control"

    Set-RegDword $ControlPath "SvcHostSplitThresholdInKB" $SvcHostValue

    Write-Host "SvcHostSplitThresholdInKB set to $SvcHostValue." -ForegroundColor Green
}

# ==========================================================
# KEYBOARD CLASS PARAMETERS
# ==========================================================

Write-Host "Configuring keyboard class parameters" -ForegroundColor Cyan

$KeyboardPath = "HKLM:\SYSTEM\CurrentControlSet\Services\kbdclass\Parameters"

try {
    if (-not (Test-Path $KeyboardPath)) {
        New-Item -Path $KeyboardPath -Force | Out-Null
    }

    Set-RegDword $KeyboardPath "ConnectMultiplePorts" 0
    Set-RegDword $KeyboardPath "KeyboardDataQueueSize" 75
    Set-RegDword $KeyboardPath "MaximumPortsServiced" 3
    Set-RegDword $KeyboardPath "SendOutputToAllPorts" 1

    New-ItemProperty `
        -Path $KeyboardPath `
        -Name "KeyboardDeviceBaseName" `
        -Value "KeyboardClass" `
        -PropertyType String `
        -Force | Out-Null

    New-ItemProperty `
        -Path $KeyboardPath `
        -Name "WppRecorder_TraceGuid" `
        -Value "{09281f1f-f66e-485a-99a2-91638f782c49}" `
        -PropertyType String `
        -Force | Out-Null

    Write-Host "Keyboard parameters configured." -ForegroundColor Green
}
catch {
    Write-Host "Keyboard configuration failed: $($_.Exception.Message)" -ForegroundColor Yellow
}

# ==========================================================
# LANMAN SERVER PARAMETERS
# ==========================================================

Write-Host "Configuring LanmanServer parameters" -ForegroundColor Cyan

$LanmanPath = "HKLM:\SYSTEM\CurrentControlSet\Services\LanmanServer\Parameters"

Set-RegDword $LanmanPath "autodisconnect" 4294967295
Set-RegDword $LanmanPath "Size" 3
Set-RegDword $LanmanPath "IRPStackSize" 32
Set-RegDword $LanmanPath "SharingViolationDelay" 0

Write-Host "LanmanServer parameters configured." -ForegroundColor Green

# ==========================================================
# TEMPORARY FILE CLEANUP
# ==========================================================

Write-Host "Cleaning temporary files" -ForegroundColor Cyan

$CleanupPaths = @(
    "$env:TEMP\*",
    "$env:WINDIR\Temp\*",
    "$env:WINDIR\Prefetch\*"
)

foreach ($Path in $CleanupPaths) {
    try {
        Remove-Item -Path $Path -Recurse -Force -ErrorAction SilentlyContinue
    }
    catch {}
}

# ==========================================================
# AUTOMATIC RAM DETECTION AND CUSTOM PAGE FILE
# ==========================================================

Write-Host "Page file configuration" -ForegroundColor Cyan

$PageFileChoice = Read-Host "Do you want to configure a custom page file? (Y/N)"

if ($PageFileChoice -match '^(Y|YES)$') {

    Write-Host "Detecting RAM and configuring page file" -ForegroundColor Cyan

    try {
        if ($RAMGB -le 8) {
            $PageFileInitial = 12288
            $PageFileMaximum = 24576
            $RAMProfile = "8GB"
        }
        elseif ($RAMGB -le 16) {
            $PageFileInitial = 8192
            $PageFileMaximum = 16384
            $RAMProfile = "16GB"
        }
        else {
            $PageFileInitial = 16384
            $PageFileMaximum = 32768
            $RAMProfile = "32GB+"
        }

        $ComputerSystem = Get-CimInstance Win32_ComputerSystem -ErrorAction Stop

        Set-CimInstance `
            -InputObject $ComputerSystem `
            -Property @{ AutomaticManagedPagefile = $false } `
            -ErrorAction Stop | Out-Null

        $PageFile = Get-CimInstance Win32_PageFileSetting -ErrorAction SilentlyContinue |
            Where-Object { $_.Name -eq "C:\pagefile.sys" }

        if ($null -eq $PageFile) {
            New-CimInstance `
                -ClassName Win32_PageFileSetting `
                -Property @{
                    Name = "C:\pagefile.sys"
                    InitialSize = $PageFileInitial
                    MaximumSize = $PageFileMaximum
                } `
                -ErrorAction Stop | Out-Null
        }
        else {
            Set-CimInstance `
                -InputObject $PageFile `
                -Property @{
                    InitialSize = $PageFileInitial
                    MaximumSize = $PageFileMaximum
                } `
                -ErrorAction Stop | Out-Null
        }

        Write-Host "RAM detected: $RAMGB GB" -ForegroundColor Green
        Write-Host "Profile: $RAMProfile" -ForegroundColor Green
        Write-Host "Initial size: $PageFileInitial MB" -ForegroundColor Green
        Write-Host "Maximum size: $PageFileMaximum MB" -ForegroundColor Green
    }
    catch {
        Write-Host "Page file configuration failed: $($_.Exception.Message)" -ForegroundColor Red
    }
}
elseif ($PageFileChoice -match '^(N|NO)$') {
    Write-Host "Page file configuration skipped. Existing settings unchanged." -ForegroundColor Yellow
}
else {
    Write-Host "Invalid selection. Page file configuration skipped." -ForegroundColor Yellow
}

# ==========================================================
# MULTIMEDIA SYSTEM PROFILE
# ==========================================================

Write-Host "Applying throttle settings" -ForegroundColor Cyan

$SystemProfile = "HKLM:\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Multimedia\SystemProfile"

Set-RegDword $SystemProfile "NetworkThrottlingIndex" 4294967295
Set-RegDword $SystemProfile "SystemResponsiveness" 10

# ==========================================================
# GAMES TASK PRIORITY
# ==========================================================

Write-Host "Configuring game priority" -ForegroundColor Cyan

$GamesPath = "$SystemProfile\Tasks\Games"

try {
    if (-not (Test-Path $GamesPath)) {
        New-Item -Path $GamesPath -Force | Out-Null
    }

    New-ItemProperty `
        -Path $GamesPath `
        -Name "Scheduling Category" `
        -Value "High" `
        -PropertyType String `
        -Force | Out-Null

    Set-RegDword $GamesPath "Priority" 2
    Set-RegDword $GamesPath "GPU Priority" 8
    Set-RegDword $GamesPath "SFIO Priority" 0
}
catch {}

# ==========================================================
# QOS SETTINGS
# ==========================================================

Write-Host "Applying QoS settings" -ForegroundColor Cyan

$QoSPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\Psched"

Set-RegDword $QoSPath "NonBestEffortLimit" 0

$TcpQoSPath = "HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Internet Settings\Connections"

try {
    if (-not (Test-Path $TcpQoSPath)) {
        New-Item -Path $TcpQoSPath -Force | Out-Null
    }

    New-ItemProperty `
        -Path $TcpQoSPath `
        -Name "Do not use NLA" `
        -Value "1" `
        -PropertyType String `
        -Force | Out-Null
}
catch {}

# ==========================================================
# DNS / HOST RESOLUTION PRIORITIES
# ==========================================================

Write-Host "Applying host resolution settings" -ForegroundColor Cyan

$ServiceProviderPath = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\ServiceProvider"

Set-RegDword $ServiceProviderPath "LocalPriority" 4
Set-RegDword $ServiceProviderPath "HostsPriority" 5
Set-RegDword $ServiceProviderPath "DnsPriority" 6
Set-RegDword $ServiceProviderPath "NetbtPriority" 7

# ==========================================================
# TCP REGISTRY SETTINGS
# ==========================================================

Write-Host "Applying TCP registry settings" -ForegroundColor Cyan

$TcpParameters = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters"

Add-RegDwordIfMissing $TcpParameters "DefaultTTL" 64
Add-RegDwordIfMissing $TcpParameters "Tcp1323Opts" 1
Add-RegDwordIfMissing $TcpParameters "MaxUserPort" 65535
Add-RegDwordIfMissing $TcpParameters "TCPNoDelay" 1
Add-RegDwordIfMissing $TcpParameters "TCPDelAckTicks" 0
Add-RegDwordIfMissing $TcpParameters "TCPAckFrequency" 1

# ==========================================================
# CPU PRIORITY
# ==========================================================

Write-Host "Applying CPU priority settings" -ForegroundColor Cyan

$PriorityPath = "HKLM:\SYSTEM\CurrentControlSet\Control\PriorityControl"

Set-RegDword $PriorityPath "Win32PrioritySeparation" 38

# ==========================================================
# SYSTEM SETTINGS
# ==========================================================

Write-Host "Applying system memory settings" -ForegroundColor Cyan

$MemoryManagement = "HKLM:\SYSTEM\CurrentControlSet\Control\Session Manager\Memory Management"

Set-RegDword $MemoryManagement "LargeSystemCache" 0

$InternetSettings = "HKCU:\Software\Microsoft\Windows\CurrentVersion\Internet Settings"

try {
    if (-not (Test-Path $InternetSettings)) {
        New-Item -Path $InternetSettings -Force | Out-Null
    }

    New-ItemProperty -Path $InternetSettings -Name "MaxConnectionsPerServer" -Value 10 -PropertyType DWord -Force | Out-Null
    New-ItemProperty -Path $InternetSettings -Name "MaxConnectionsPer1_0Server" -Value 10 -PropertyType DWord -Force | Out-Null
}
catch {}

# ==========================================================
# PRIVACY, TELEMETRY AND SERVICE CONFIGURATION
# ==========================================================

Write-Host "Configuring privacy, telemetry and services" -ForegroundColor Cyan

$DisabledServices = @(
    "DiagTrack",
    "WerSvc",
    "CscService",
    "SharedAccess",
    "lfsvc"
)

$ManualServices = @(
    "MapsBroker",
    "StorSvc"
)

foreach ($Service in $DisabledServices) {
    try {
        Stop-Service -Name $Service -Force -ErrorAction SilentlyContinue
        Set-Service -Name $Service -StartupType Disabled -ErrorAction SilentlyContinue
    }
    catch {}
}

foreach ($Service in $ManualServices) {
    try {
        Set-Service -Name $Service -StartupType Manual -ErrorAction SilentlyContinue
    }
    catch {}
}

# ==========================================================
# PRIVACY REGISTRY SETTINGS
# ==========================================================

$PrivacyRegistry = @(
    @{ Path = "HKCU\Software\Microsoft\Windows\CurrentVersion\AdvertisingInfo"; Name = "Enabled"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKCU\Software\Microsoft\Windows\CurrentVersion\Privacy"; Name = "TailoredExperiencesWithDiagnosticDataEnabled"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKCU\Software\Microsoft\Speech_OneCore\Settings\OnlineSpeechPrivacy"; Name = "HasAccepted"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKCU\Software\Microsoft\Input\TIPC"; Name = "Enabled"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKCU\Software\Microsoft\InputPersonalization"; Name = "RestrictImplicitInkCollection"; Type = "REG_DWORD"; Value = "1" },
    @{ Path = "HKCU\Software\Microsoft\InputPersonalization"; Name = "RestrictImplicitTextCollection"; Type = "REG_DWORD"; Value = "1" },
    @{ Path = "HKCU\Software\Microsoft\InputPersonalization\TrainedDataStore"; Name = "HarvestContacts"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKCU\Software\Microsoft\Personalization\Settings"; Name = "AcceptedPrivacyPolicy"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\DataCollection"; Name = "AllowTelemetry"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKCU\Software\Microsoft\Windows\CurrentVersion\Explorer\Advanced"; Name = "Start_TrackProgs"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\CapabilityAccessManager\ConsentStore\location"; Name = "Value"; Type = "REG_SZ"; Value = "Deny" },
    @{ Path = "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Sensor\Overrides\{BFA794E4-F964-4FDB-90F6-51056BFE4B44}"; Name = "SensorPermissionState"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKLM\SYSTEM\Maps"; Name = "AutoUpdateEnabled"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKLM\SOFTWARE\Policies\Microsoft\Windows\DeliveryOptimization"; Name = "DODownloadMode"; Type = "REG_DWORD"; Value = "0" },
    @{ Path = "HKLM\SOFTWARE\Policies\Microsoft\Windows\Device Metadata"; Name = "PreventDeviceMetadataFromNetwork"; Type = "REG_DWORD"; Value = "1" }
)

foreach ($Entry in $PrivacyRegistry) {
    & reg.exe add $Entry.Path `
        /v $Entry.Name `
        /t $Entry.Type `
        /d $Entry.Value `
        /f | Out-Null
}

# ==========================================================
# ACTIVITY HISTORY
# ==========================================================

Write-Host "Configuring Activity History" -ForegroundColor Cyan

$ActivityHistoryPath = "HKLM:\SOFTWARE\Policies\Microsoft\Windows\System"

try {
    if (-not (Test-Path $ActivityHistoryPath)) {
        New-Item -Path $ActivityHistoryPath -Force | Out-Null
    }

    Set-RegDword $ActivityHistoryPath "EnableActivityFeed" 1
    Set-RegDword $ActivityHistoryPath "PublishUserActivities" 0
    Set-RegDword $ActivityHistoryPath "UploadUserActivities" 0
}
catch {}

# ==========================================================
# FEEDBACK SETTINGS
# ==========================================================

Write-Host "Configuring feedback settings" -ForegroundColor Cyan

& reg.exe delete "HKCU\Software\Microsoft\Siuf\Rules" `
    /v "PeriodInNanoSeconds" /f | Out-Null

Set-RegDword "HKCU:\Software\Microsoft\Siuf\Rules" "NumberOfSIUFInPeriod" 0

# ==========================================================
# MICROSOFT DEFENDER SAMPLE SUBMISSION
# ==========================================================

Write-Host "Configuring Defender sample submission" -ForegroundColor Cyan

try {
    Set-MpPreference -SubmitSamplesConsent 2 -ErrorAction Stop
}
catch {}

# ==========================================================
# POWERSHELL TELEMETRY
# ==========================================================

Write-Host "Configuring PowerShell telemetry" -ForegroundColor Cyan

[Environment]::SetEnvironmentVariable(
    "POWERSHELL_TELEMETRY_OPTOUT",
    "1",
    "Machine"
)

# ==========================================================
# TCP GLOBAL SETTINGS
# ==========================================================

Write-Host "Configuring TCP global settings" -ForegroundColor Cyan

Invoke-Netsh @("int", "tcp", "set", "global", "rss=enabled")
Invoke-Netsh @("int", "tcp", "set", "global", "autotuninglevel=normal")
Invoke-Netsh @("int", "tcp", "set", "global", "ecncapability=disabled")
Invoke-Netsh @("int", "tcp", "set", "global", "timestamps=disabled")
Invoke-Netsh @("int", "tcp", "set", "global", "initialrto=2500")
Invoke-Netsh @("int", "tcp", "set", "global", "maxsynretransmissions=2")
Invoke-Netsh @("int", "tcp", "set", "global", "nonsackrttresiliency=disabled")

Invoke-Netsh @("int", "tcp", "set", "heuristics", "wsh=disabled", "forcews=default")

Invoke-Netsh @("int", "tcp", "set", "supplemental", "template=internet", "congestionprovider=ctcp")
Invoke-Netsh @("int", "tcp", "set", "supplemental", "template=internet", "minrto=250")
Invoke-Netsh @("int", "tcp", "set", "supplemental", "template=internet", "delayedacktimeout=100")
Invoke-Netsh @("int", "tcp", "set", "supplemental", "template=internet", "rack=enabled")
Invoke-Netsh @("int", "tcp", "set", "supplemental", "template=internet", "taillossprobe=enabled")

# ==========================================================
# ACTIVE PHYSICAL NETWORK ADAPTERS
# ==========================================================

Write-Host "Configuring active network adapters" -ForegroundColor Cyan

$Adapters = @(
    Get-NetAdapter -Physical |
    Where-Object { $_.Status -eq "Up" }
)

foreach ($Adapter in $Adapters) {

    $AdapterName = $Adapter.Name

    try {
        Enable-NetAdapterRss -Name $AdapterName -ErrorAction Stop
    }
    catch {}

    try {
        Enable-NetAdapterRsc -Name $AdapterName -ErrorAction Stop
    }
    catch {}

    try {
        Enable-NetAdapterChecksumOffload -Name $AdapterName -ErrorAction Stop
    }
    catch {}

    try {
        Disable-NetAdapterLso -Name $AdapterName -ErrorAction Stop
    }
    catch {}

    if ($ApplyNagleTweaks) {
        try {
            $AdapterGuid = $Adapter.InterfaceGuid.ToString("B")

            $AdapterTcpPath = "HKLM:\SYSTEM\CurrentControlSet\Services\Tcpip\Parameters\Interfaces\$AdapterGuid"

            Set-RegDword $AdapterTcpPath "TCPNoDelay" 1
            Set-RegDword $AdapterTcpPath "TcpAckFrequency" 1
            Set-RegDword $AdapterTcpPath "TcpDelAckTicks" 0
        }
        catch {}
    }
}

# ==========================================================
# CLOUDFLARE DNS
# ==========================================================

Write-Host "Configuring Cloudflare DNS" -ForegroundColor Cyan

foreach ($Adapter in $Adapters) {

    try {
        Set-DnsClientServerAddress `
            -InterfaceIndex $Adapter.ifIndex `
            -ServerAddresses @("1.1.1.1", "1.0.0.1") `
            -ErrorAction Stop
    }
    catch {}
}

try {
    Clear-DnsClientCache
}
catch {
    ipconfig.exe /flushdns | Out-Null
}

# ==========================================================
# IPV4 MTU DETECTION
# ==========================================================

Write-Host "Checking IPv4 path MTU" -ForegroundColor Cyan

function Test-IPv4Payload {
    param(
        [string]$Target,
        [int]$Payload
    )

    $SuccessCount = 0

    for ($i = 0; $i -lt 3; $i++) {
        & ping.exe -4 -n 1 -w 1500 -f -l $Payload $Target | Out-Null

        if ($LASTEXITCODE -eq 0) {
            $SuccessCount++
        }
    }

    return ($SuccessCount -ge 2)
}

function Get-IPv4PathMTU {
    param([string]$Target)

    $Low = 548
    $High = 1472
    $Best = 0

    while ($Low -le $High) {

        $Middle = [int][Math]::Floor(($Low + $High) / 2)

        if (Test-IPv4Payload -Target $Target -Payload $Middle) {
            $Best = $Middle
            $Low = $Middle + 1
        }
        else {
            $High = $Middle - 1
        }
    }

    if ($Best -gt 0) {
        return ($Best + 28)
    }

    return $null
}

foreach ($Target in $IPv4MTUTargets) {

    $DetectedMTU = Get-IPv4PathMTU -Target $Target

    if ($null -ne $DetectedMTU) {
        Write-Host "$Target IPv4 path MTU: $DetectedMTU" -ForegroundColor Green
    }
    else {
        Write-Host "$Target IPv4 MTU detection failed." -ForegroundColor Yellow
    }
}

# ==========================================================
# IPV6 MTU - FORCE 1500
# ==========================================================

Write-Host "Setting IPv6 MTU to 1500" -ForegroundColor Cyan

foreach ($Adapter in $Adapters) {

    try {
        Set-NetIPInterface `
            -InterfaceIndex $Adapter.ifIndex `
            -AddressFamily IPv6 `
            -NlMtuBytes 1500 `
            -ErrorAction Stop
    }
    catch {}
}

# ==========================================================
# CONNECTIVITY TEST
# ==========================================================

Write-Host "Testing IPv4 and IPv6 connectivity" -ForegroundColor Cyan

Write-Host ""
Write-Host "IPv4 connectivity:" -ForegroundColor Gray
ping.exe -4 -n 2 1.1.1.1

Write-Host ""
Write-Host "IPv6 connectivity:" -ForegroundColor Gray
ping.exe -6 -n 2 2606:4700:4700::1111

# ==========================================================
# GPU DETECTION AND VENDOR-SPECIFIC REGISTRY SETTINGS
# ==========================================================

Write-Host "Detecting installed GPUs" -ForegroundColor Cyan

try {
    $VideoControllers = @(
        Get-CimInstance Win32_VideoController -ErrorAction Stop
    )

    $HasNVIDIA = [bool](
        $VideoControllers | Where-Object {
            $_.Name -match 'NVIDIA' -or
            $_.PNPDeviceID -match 'VEN_10DE'
        }
    )

    $HasAMD = [bool](
        $VideoControllers | Where-Object {
            $_.Name -match 'AMD|Radeon|Advanced Micro Devices' -or
            $_.PNPDeviceID -match 'VEN_1002'
        }
    )

    # ------------------------------------------------------
    # NVIDIA GPU DETECTION AND REGISTRY SETTINGS
    # ------------------------------------------------------

    if ($HasNVIDIA) {
        Write-Host "NVIDIA GPU detected." -ForegroundColor Green

        $NVIDIAPath = "HKLM:\SYSTEM\CurrentControlSet\Services\nvlddmkm"

        if (Test-Path $NVIDIAPath) {

            Write-Host "Applying NVIDIA registry settings..." -ForegroundColor Cyan

            try {
                Set-Item `
                    -Path $NVIDIAPath `
                    -Value 5 `
                    -ErrorAction Stop

                Write-Host "nvlddmkm default value set to 5." -ForegroundColor Green
            }
            catch {
                Write-Host "Failed to set nvlddmkm default value: $($_.Exception.Message)" -ForegroundColor Red
            }

            try {
                New-ItemProperty `
                    -Path $NVIDIAPath `
                    -Name "PerfLevelSrc" `
                    -PropertyType DWord `
                    -Value 0x2222 `
                    -Force `
                    -ErrorAction Stop | Out-Null

                Write-Host "PerfLevelSrc set to 0x2222." -ForegroundColor Green
            }
            catch {
                Write-Host "Failed to set PerfLevelSrc: $($_.Exception.Message)" -ForegroundColor Red
            }
        }
        else {
            Write-Host "NVIDIA driver registry key not found. NVIDIA edits skipped." -ForegroundColor Yellow
        }
    }

    # ------------------------------------------------------
    # AMD GPU DETECTION
    # ------------------------------------------------------

    if ($HasAMD) {
        Write-Host "AMD GPU detected. Checking AMD display-adapter registry keys." -ForegroundColor Green

        $AMDClassPath = "HKLM:\SYSTEM\CurrentControlSet\Control\Class\{4d36e968-e325-11ce-bfc1-08002be10318}"
        $AMDKeysUpdated = 0

        if (Test-Path $AMDClassPath) {
            $AMDDeviceIDs = @(
                $VideoControllers |
                Where-Object {
                    $_.Name -match 'AMD|Radeon|Advanced Micro Devices' -or
                    $_.PNPDeviceID -match 'VEN_1002'
                } |
                ForEach-Object {
                    [string]$_.PNPDeviceID
                }
            )

            foreach ($DeviceKey in (
                Get-ChildItem -Path $AMDClassPath -ErrorAction SilentlyContinue |
                Where-Object {
                    $_.PSChildName -match '^\d{4}$'
                }
            )) {

                $DeviceProperties = Get-ItemProperty `
                    -Path $DeviceKey.PSPath `
                    -ErrorAction SilentlyContinue

                $DriverDescription = [string]$DeviceProperties.DriverDesc
                $MatchingDeviceID = [string]$DeviceProperties.MatchingDeviceId
                $HardwareID = [string]$DeviceProperties.HardwareID

                $IsAMDDevice = (
                    $DriverDescription -match 'AMD|Radeon|Advanced Micro Devices'
                ) -or (
                    $MatchingDeviceID -match 'VEN_1002'
                ) -or (
                    $HardwareID -match 'VEN_1002'
                )

                if (-not $IsAMDDevice) {
                    foreach ($AMDDeviceID in $AMDDeviceIDs) {

                        if (
                            $AMDDeviceID -and (
                                ($MatchingDeviceID -and $AMDDeviceID -like "*$MatchingDeviceID*") -or
                                ($HardwareID -and $AMDDeviceID -like "*$HardwareID*")
                            )
                        ) {
                            $IsAMDDevice = $true
                            break
                        }
                    }
                }

                if (-not $IsAMDDevice) {
                    continue
                }

                $AMDKeyPath = $DeviceKey.PSPath

                $ExistingAMDValues = Get-ItemProperty `
                    -Path $AMDKeyPath `
                    -ErrorAction SilentlyContinue

                if ($null -ne $ExistingAMDValues.PSObject.Properties['EnableUlps']) {
                    Set-RegDword $AMDKeyPath "EnableUlps" 0
                }
                else {
                    Write-Host "EnableUlps not present in $($DeviceKey.PSChildName); skipped." -ForegroundColor Yellow
                }

                if ($null -ne $ExistingAMDValues.PSObject.Properties['EnableAspm']) {
                    Set-RegDword $AMDKeyPath "EnableAspm" 0
                }

                if ($null -ne $ExistingAMDValues.PSObject.Properties['PP_SclkDeepSleepDisable']) {
                    Set-RegDword $AMDKeyPath "PP_SclkDeepSleepDisable" 1
                }

                $AMDKeysUpdated++

                Write-Host "Applied available AMD settings to display adapter key $($DeviceKey.PSChildName)." -ForegroundColor Green
            }
        }

        if ($AMDKeysUpdated -eq 0) {
            Write-Host "AMD was detected, but no matching AMD display-adapter registry key was identified. AMD edits skipped." -ForegroundColor Yellow
        }
        else {
            $DwmPath = "HKLM:\SOFTWARE\Microsoft\Windows\Dwm"

            Set-RegDword $DwmPath "OverlayTestMode" 5

            Write-Host "DWM OverlayTestMode set to 5." -ForegroundColor Green
        }
    }

    # ------------------------------------------------------
    # NO SUPPORTED VENDOR GPU
    # ------------------------------------------------------

    if (-not $HasNVIDIA -and -not $HasAMD) {
        Write-Host "No NVIDIA or AMD GPU detected. Vendor-specific GPU registry edits skipped." -ForegroundColor Yellow
    }

    # ------------------------------------------------------
    # DISPLAY ALL DETECTED GPUS
    # ------------------------------------------------------

    foreach ($VideoController in $VideoControllers) {
        Write-Host "GPU: $($VideoController.Name)" -ForegroundColor Gray
    }
}
catch {
    Write-Host "GPU detection or vendor-specific configuration failed: $($_.Exception.Message)" -ForegroundColor Red
}

# ==========================================================
# FINISH
# ==========================================================

Write-Host ""
Write-Host "============================================" -ForegroundColor Cyan
Write-Host " FPS AND PING OPTIMIZER FINISHED"
Write-Host "============================================" -ForegroundColor Cyan
Write-Host ""
Write-Host "RAM detected: $RAMGB GB"
Write-Host "Memory compression: RAM-based configuration"
Write-Host "SvcHost split threshold: $SvcHostValue"
Write-Host "Keyboard data queue size: 75"
Write-Host "LanmanServer parameters: Configured"
Write-Host "Page file: Custom configuration offered"
Write-Host "Activity History: Publishing and uploading disabled"
Write-Host "Privacy: Configured"
Write-Host "IPv6 MTU: 1500 requested on active physical adapters"
Write-Host "IPv4 MTU: Path detection only; adapter MTU unchanged"
Write-Host "DNS: Cloudflare 1.1.1.1 / 1.0.0.1"
Write-Host ""
Write-Host "Some settings may require a restart to take effect."
Write-Host "Check your connection after applying the changes."
Write-Host ""

Read-Host "Press Enter to exit"
