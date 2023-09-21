+++ 
draft = true
date = 2023-09-21T06:29:24Z
title = ""
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

## Writing a Terraform Version Manager in PowerShell

If you're a developer who works with Terraform, you know how important it is to have the right version of Terraform installed for your project. But what if you need to work on multiple projects, each with a different version of Terraform? That's where a Terraform version manager comes in.

There are several Terraform version managers available for Linux and Mac, including:

- [tfenv](https://github.com/tfutils/tfenv)
- [tfswitch](https://warrensbox.github.io/terraform-manager/)
- [tfupdate](https://github.com/minamijoyo/tfupdate)

These tools allow you to easily switch between different versions of Terraform on your local machine. These tools are great if you're using Linux or Mac, or if you're using Windows with WSL. However, there is currently no Terraform version manager available for PowerShell on Windows. Fortunately, it's easy to write your own Terraform version manager!

In this post, we'll walk through how to write your own Terraform Version Manager in PowerShell that allows you to easily install and switch between different versions of Terraform on your local machine.

### Getting Started

To get started, we'll need to create a new PowerShell script. We'll call our script `tfv.ps1`. This script will contain all the functions we need to install and switch between different versions of Terraform.

### Listing Available Terraform Versions

The first thing we'll need to do is list the available versions of Terraform we can install. We'll create a function called `Show-TerraformVersions` that returns a list of all the available versions of Terraform.

```powershell
function Show-TerraformVersions {
    $Response = Invoke-WebRequest -Headers @{"accept" = "application/json" } -uri "https://releases.hashicorp.com/terraform"
    $TerraformVersions = [Regex]::Matches($Response.Content, '<a href=".+">terraform_(?<Version>.+)</a>') | ForEach-Object {
        $_.Groups["Version"].Value
    }

    if ($TerraformVersions.Count -eq 0) {
        Write-Host "No Terraform versions installed." -ForegroundColor Red
        return
    }

    $ActiveVersion = Get-ActiveTerraformVersion
    $TerraformVersions | ForEach-Object {
        if ($_ -eq $ActiveVersion) {
            Write-Host "* $_" -ForegroundColor Green
        }
        else {
            Write-Host "  $_"
        }
    }
}
```

This function uses the `Invoke-WebRequest` cmdlet to get the list of available versions of Terraform from the HashiCorp website. It then sorts the version numbers in descending order so that the latest version is first. Finally, it displays the list of versions, highlighting the active version.

### Installing Terraform Versions

Now that we are able to list the available versions of Terraform, we can create a function to install a given version. To do this we'll create a function called `Install-TerraformVersion` that takes a version number as a parameter and downloads the corresponding version of Terraform.

```powershell
function Install-TerraformVersion {
    param(
        [Parameter(Mandatory = $true)]
        [string] $Version
    )

    # Download and extract the Terraform binary
    $TempFile = New-TemporaryFile | Rename-Item -NewName { $_ -replace 'tmp$', 'zip' } –PassThru
    Invoke-WebRequest -Uri "https://releases.hashicorp.com/terraform/$($Version)/terraform_$($Version)_windows_amd64.zip" -OutFile $TempFile
    $TempFile | Expand-Archive -DestinationPath "$env:ProgramData\tfv\$($Version)" -Force | Out-Null
    $TempFile | Remove-Item -Force
}
```

This function downloads the required Terraform binary to a temporary file, extracts it to `$env:ProgramData\tfv\$Version`, then deletes the temporary file.

### Getting the Active Terraform Version

We'll also need a way to determine which version of Terraform is currently active. To do this, we'll create a function called `Get-ActiveTerraformVersion` that returns the active version of Terraform.

```powershell
function Get-ActiveTerraformVersion {
    Get-Content -Path $TerraformVersionFile -ErrorAction SilentlyContinue
}
```

### Listing Installed Terraform Versions

We also need a way to see the installed versions of Terraform so we know what versions are available to switch to. For this we'll just modify the `Show-TerraformVersions` function we created earlier by adding a `-LocalOnly` switch parameter.

```powershell
function Show-TerraformVersions {
    param(
        [Parameter(Mandatory = $false)]
        [switch] $LocalOnly
    )

    if ($LocalOnly) {
        $TerraformVersions = @(Get-ChildItem "$env:ProgramData\tfv" | Select-Object -ExpandProperty Name)
    }
    else {
        $Response = Invoke-WebRequest -Headers @{"accept" = "application/json" } -uri "https://releases.hashicorp.com/terraform"
        $TerraformVersions = [Regex]::Matches($Response.Content, '<a href=".+">terraform_(?<Version>.+)</a>') | ForEach-Object {
            $_.Groups["Version"].Value
        }
    }

    if ($TerraformVersions.Count -eq 0) {
        Write-Host "No Terraform versions installed." -ForegroundColor Red
        return
    }

    $ActiveVersion = Get-ActiveTerraformVersion
    $TerraformVersions | ForEach-Object {
        if ($_ -eq $ActiveVersion) {
            Write-Host "* $_" -ForegroundColor Green
        }
        else {
            Write-Host "  $_"
        }
    }
}
```

This function now uses the `Get-ChildItem` cmdlet to list the installed versions of Terraform. It then sorts the version numbers in descending order so that the latest version is first. Finally, it displays the list of versions, highlighting the active version.

### Setting the Active Terraform Version

Now that we have a way to install different versions of Terraform, we need a way to indicate which version we want to use. To do this, we'll create a function called  `Set-ActiveTerraformVersion` that sets the active version of Terraform.

```powershell
function Set-ActiveTerraformVersion {
    param(
        [Parameter(Mandatory = $true)]
        [string] $Version
    )

    $TerraformExePath = "$env:ProgramData\tfv\$Version\terraform.exe"
    if (!(Test-Path -Path $TerraformExePath)) {
        Write-Host "Terraform version $Version not installed!" -ForegroundColor Red
        return
    }

    Set-Content -Path "$env:UserProfile\.tfv" -Value $Version -Force
    $env:TerraformVersion = $Version
    Set-Alias -Name terraform -Value $TerraformExePath -Scope Global
}
```

This function sets the active version of Terraform by creating an alias for the `terraform` command that points to the appropriate version of Terraform based on the version you have set as active. It also sets the `TerraformVersion` environment variable to the active version, which can be used by other scripts to determine which version of Terraform is active.

### Uninstalling Terraform Versions

Last but not least, we need a way to uninstall a version of Terraform. To do this, we'll create a function called `Uninstall-TerraformVersion` that takes a version number as a parameter and deletes the corresponding version of Terraform.

```powershell
function Uninstall-TerraformVersion {
    param(
        [Parameter(Mandatory = $true)]
        [string] $Version
    )

    if (!(Test-Path -Path "$TerraformVersionsInstallDir\$($Version)")) {
        Write-Host "Terraform version $Version not installed!" -ForegroundColor Red
        return
    }

    if ($Version -eq (Get-ActiveTerraformVersion)) {
        Remove-ActiveTerraformVersion
    }

    Remove-Item -Path "$TerraformVersionsInstallDir\$($Version)" -Recurse -Force
    Write-Host "Terraform version $Version uninstalled." -ForegroundColor Green
}
```

### Putting it all together

Now that we have all the functions we need, we can put them together into a single script. We'll call our script `tfv.ps1` and put it in a folder called `tfv` in our home directory.

```powershell

function Invoke-TerraformVersion {
    param(

        [Parameter(Mandatory = $false, Position = 0)]
        [ValidateSet("current", "install", "list", "uninstall", "use", "set", "clear", "unset", "help" )]
        [string] $Command,
        [Parameter(Mandatory = $false, Position = 1)]
        [string] $Arg
    )

    if ([String]::IsNullOrWhiteSpace($Command)) {
        $Command = "current"
    }

    if ($Command -eq "help") {
        Write-Host "Usage:"
        Write-Host "  tfv current             " -NoNewLine -Foreground Cyan; Write-Host ": Show the currently active Terraform version"
        Write-Host "  tfv install <version>   " -NoNewLine -Foreground Cyan; Write-Host ": Install a new version of Terraform"
        Write-Host "  tfv list [available]    " -NoNewLine -Foreground Cyan; Write-Host ": List all installed versions of Terraform"
        Write-Host "  tfv uninstall <version> " -NoNewLine -Foreground Cyan; Write-Host ": Uninstall a version of Terraform"
        Write-Host "  tfv use <version>       " -NoNewLine -Foreground Cyan; Write-Host ": Set the active version of Terraform [alias: tfv set]"
        Write-Host "  tfv unset               " -NoNewLine -Foreground Cyan; Write-Host ": Unset the active version of Terraform [alias: tfv clear]"
        Write-Host "  tfv help                " -NoNewLine -Foreground Cyan; Write-Host ": Show this help message"
        return
    }

    switch ($Command) {
        "current" {
            Get-ActiveTerraformVersion
        }
        "install" {
            Install-TerraformVersion -Version $Arg
        }
        "list" {
            if ($Arg -eq "available") {
                Show-TerraformVersions
            }
            else {
                Show-TerraformVersions -LocalOnly
            }
        }
        "uninstall" {
            Uninstall-TerraformVersion -Version $Arg
        }
        { $_ -in "use", "set" } {
            Set-ActiveTerraformVersion -Version $Arg
        }
        { $_ -in "unset", "clear" } {
            Remove-ActiveTerraformVersion
        }
    }
}

if (Get-ActiveTerraformVersion -ne $null) { 
    Set-ActiveTerraformVersion -Version (Get-ActiveTerraformVersion)
}

Set-Alias tfv Invoke-TerraformVersion
```

This function uses the `switch` statement to determine which function to call based on the command that was passed in. It also sets up an alias for the `tfv` command that points to the `Invoke-TerraformVersion` function.

### How to use it

To use the Terraform Version Manager, you'll need to have PowerShell installed on your machine. Once you have PowerShell installed, you can download the script and run it in a PowerShell console.

Here are the basic commands you'll need to know:

- `tfv use <version>`: Set the active version of Terraform
- `tfv list`: List all installed versions of Terraform
- `tfv install <version>`: Install a new version of Terraform
- `tfv uninstall <version>`: Uninstall a version of Terraform

For more information on how to use the Terraform Version Manager, you can run the `tfv help` command.

### Conclusion

In this post, we walked through how to write your own Terraform Version Manager in PowerShell that allows you to easily install and switch between different versions of Terraform on your local machine. We also covered how to use the Terraform Version Manager to install and switch between different versions of Terraform.

See the complete code for this post on [GitHub](
