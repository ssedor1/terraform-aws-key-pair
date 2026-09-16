#requires -Version 5.1

<#
.SYNOPSIS
    Installs Az.Accounts, Az.Storage, and IIS Web-Scripting-Tools.

.NOTES
    Designed for Windows Server 2025 Datacenter.
    Intended for Azure VM Run Command / Terraform automation.
    Safe to run repeatedly; already-installed components are skipped.

    To disable a component later:
      - Set $InstallAzModules = $false
      - Set $InstallWebScriptingTools = $false
#>

$ErrorActionPreference = 'Stop'

# ============================================================================
# Configuration
# ============================================================================

$InstallAzModules          = $true
$InstallWebScriptingTools  = $true

$AzModules = @(
    'Az.Accounts',
    'Az.Storage'
)

$WindowsFeatures = @(
    'Web-Scripting-Tools'
)

$LogDirectory = 'C:\ProgramData\ServerBootstrap'
$LogFile      = Join-Path $LogDirectory 'server-bootstrap.log'
$TranscriptFile = Join-Path $LogDirectory 'server-bootstrap-transcript.log'

# ============================================================================
# Logging
# ============================================================================

if (-not (Test-Path -LiteralPath $LogDirectory)) {
    New-Item -Path $LogDirectory -ItemType Directory -Force | Out-Null
}

function Write-Log {
    param (
        [Parameter(Mandatory)]
        [string]$Message,

        [ValidateSet('INFO', 'WARN', 'ERROR')]
        [string]$Level = 'INFO'
    )

    $Timestamp = Get-Date -Format 'yyyy-MM-dd HH:mm:ss'
    $Line = "[$Timestamp] [$Level] $Message"

    Write-Output $Line
    Add-Content -LiteralPath $LogFile -Value $Line
}

try {
    Start-Transcript -Path $TranscriptFile -Append -ErrorAction SilentlyContinue | Out-Null
}
catch {
    # Transcript failure should not prevent installation.
}

try {

    Write-Log "============================================================"
    Write-Log "Windows Server bootstrap started."
    Write-Log "Computer: $env:COMPUTERNAME"
    Write-Log "PowerShell: $($PSVersionTable.PSVersion)"
    Write-Log "============================================================"

    # ========================================================================
    # Dependency / Environment Checks
    # ========================================================================

    Write-Log "Checking operating system."

    $OS = Get-CimInstance -ClassName Win32_OperatingSystem

    if ($OS.Caption -notmatch 'Windows Server') {
        throw "This script requires Windows Server. Detected: $($OS.Caption)"
    }

    Write-Log "Operating system: $($OS.Caption)"

    Write-Log "Checking administrative privileges."

    $CurrentIdentity = [Security.Principal.WindowsIdentity]::GetCurrent()
    $Principal = New-Object Security.Principal.WindowsPrincipal($CurrentIdentity)

    if (-not $Principal.IsInRole(
        [Security.Principal.WindowsBuiltInRole]::Administrator
    )) {
        throw "PowerShell must be running with Administrator privileges."
    }

    Write-Log "Administrative privileges confirmed."

    # ========================================================================
    # Windows Feature Dependencies
    # ========================================================================

    if ($InstallWebScriptingTools) {

        Write-Log "Checking ServerManager module."

        if (-not (Get-Module -ListAvailable -Name ServerManager)) {
            throw "ServerManager PowerShell module is not available."
        }

        Import-Module ServerManager -ErrorAction Stop

        Write-Log "ServerManager module available."
    }

    # ========================================================================
    # Az PowerShell Modules
    # ========================================================================

    if ($InstallAzModules) {

        Write-Log "------------------------------------------------------------"
        Write-Log "Az PowerShell module installation enabled."
        Write-Log "Modules: $($AzModules -join ', ')"
        Write-Log "------------------------------------------------------------"

        # --------------------------------------------------------------------
        # PackageManagement dependency
        # --------------------------------------------------------------------

        if (-not (Get-Command -Name Install-PackageProvider -ErrorAction SilentlyContinue)) {
            throw "Install-PackageProvider is unavailable. PackageManagement is required."
        }

        Write-Log "PackageManagement dependency confirmed."

        # --------------------------------------------------------------------
        # NuGet dependency
        # --------------------------------------------------------------------

        Write-Log "Checking NuGet package provider."

        $NuGetProvider = Get-PackageProvider `
            -Name NuGet `
            -ListAvailable `
            -ErrorAction SilentlyContinue

        if (-not $NuGetProvider) {

            Write-Log "NuGet provider not found. Installing NuGet 2.8.5.201 or newer."

            Install-PackageProvider `
                -Name NuGet `
                -MinimumVersion 2.8.5.201 `
                -Force `
                -ErrorAction Stop | Out-Null

            Write-Log "NuGet provider installed."
        }
        else {
            Write-Log "NuGet provider already installed: $($NuGetProvider.Version)"
        }

        # --------------------------------------------------------------------
        # PowerShell Gallery
        # --------------------------------------------------------------------

        Write-Log "Checking PowerShell Gallery."

        $PSGallery = Get-PSRepository `
            -Name PSGallery `
            -ErrorAction SilentlyContinue

        if (-not $PSGallery) {

            Write-Log "PSGallery is not registered. Registering PSGallery."

            Register-PSRepository -Default -ErrorAction Stop

            $PSGallery = Get-PSRepository `
                -Name PSGallery `
                -ErrorAction Stop
        }

        Write-Log "PSGallery available."

        # Avoid interactive trust prompts during automation.
        if ($PSGallery.InstallationPolicy -ne 'Trusted') {

            Write-Log "Setting PSGallery installation policy to Trusted."

            Set-PSRepository `
                -Name PSGallery `
                -InstallationPolicy Trusted `
                -ErrorAction Stop
        }

        # --------------------------------------------------------------------
        # Install requested Az modules
        # --------------------------------------------------------------------

        foreach ($ModuleName in $AzModules) {

            Write-Log "Checking PowerShell module: $ModuleName"

            $InstalledModule = Get-Module `
                -ListAvailable `
                -Name $ModuleName `
                -ErrorAction SilentlyContinue |
                Sort-Object Version -Descending |
                Select-Object -First 1

            if ($InstalledModule) {

                Write-Log "$ModuleName already installed. Version: $($InstalledModule.Version)"
                continue
            }

            Write-Log "$ModuleName is not installed. Installing from PSGallery."

            Install-Module `
                -Name $ModuleName `
                -Repository PSGallery `
                -Scope AllUsers `
                -Force `
                -AllowClobber `
                -ErrorAction Stop

            # Verify installation immediately.
            $InstalledModule = Get-Module `
                -ListAvailable `
                -Name $ModuleName `
                -ErrorAction SilentlyContinue |
                Sort-Object Version -Descending |
                Select-Object -First 1

            if (-not $InstalledModule) {
                throw "Installation verification failed for module: $ModuleName"
            }

            Write-Log "$ModuleName successfully installed. Version: $($InstalledModule.Version)"
        }
    }
    else {
        Write-Log "Az module installation disabled."
    }

    # ========================================================================
    # Windows Web Scripting Tools
    # ========================================================================

    if ($InstallWebScriptingTools) {

        Write-Log "------------------------------------------------------------"
        Write-Log "Windows Web-Scripting-Tools installation enabled."
        Write-Log "------------------------------------------------------------"

        foreach ($FeatureName in $WindowsFeatures) {

            Write-Log "Checking Windows feature: $FeatureName"

            $Feature = Get-WindowsFeature `
                -Name $FeatureName `
                -ErrorAction Stop

            if ($Feature.Installed) {

                Write-Log "$FeatureName is already installed."
                continue
            }

            Write-Log "$FeatureName is not installed. Installing."

            $InstallResult = Install-WindowsFeature `
                -Name $FeatureName `
                -ErrorAction Stop `
                -LogPath (Join-Path $LogDirectory "$FeatureName-install.log")

            if (-not $InstallResult.Success) {
                throw "Installation reported failure for Windows feature: $FeatureName"
            }

            Write-Log "$FeatureName installation completed."
            Write-Log "Restart required: $($InstallResult.RestartNeeded)"

            # Verify installation.
            $Feature = Get-WindowsFeature `
                -Name $FeatureName `
                -ErrorAction Stop

            if (-not $Feature.Installed) {
                throw "Installation verification failed for Windows feature: $FeatureName"
            }

            Write-Log "Verified: $FeatureName is installed."
        }
    }
    else {
        Write-Log "Web-Scripting-Tools installation disabled."
    }

    # ========================================================================
    # Final Verification
    # ========================================================================

    Write-Log "------------------------------------------------------------"
    Write-Log "Final verification."
    Write-Log "------------------------------------------------------------"

    if ($InstallAzModules) {

        foreach ($ModuleName in $AzModules) {

            $Module = Get-Module `
                -ListAvailable `
                -Name $ModuleName `
                -ErrorAction SilentlyContinue |
                Sort-Object Version -Descending |
                Select-Object -First 1

            if ($Module) {
                Write-Log "VERIFIED: $ModuleName $($Module.Version)"
            }
            else {
                throw "FINAL VERIFICATION FAILED: $ModuleName"
            }
        }
    }

    if ($InstallWebScriptingTools) {

        foreach ($FeatureName in $WindowsFeatures) {

            $Feature = Get-WindowsFeature `
                -Name $FeatureName `
                -ErrorAction Stop

            if ($Feature.Installed) {
                Write-Log "VERIFIED: $FeatureName"
            }
            else {
                throw "FINAL VERIFICATION FAILED: $FeatureName"
            }
        }
    }

    Write-Log "============================================================"
    Write-Log "Windows Server bootstrap completed successfully."
    Write-Log "Log file: $LogFile"
    Write-Log "Transcript: $TranscriptFile"
    Write-Log "============================================================"

    exit 0
}
catch {

    Write-Log "============================================================" -Level ERROR
    Write-Log "Windows Server bootstrap FAILED." -Level ERROR
    Write-Log "Error: $($_.Exception.Message)" -Level ERROR
    Write-Log "============================================================" -Level ERROR

    exit 1
}
finally {

    try {
        Stop-Transcript -ErrorAction SilentlyContinue | Out-Null
    }
    catch {
        # Ignore transcript shutdown errors.
    }
}
