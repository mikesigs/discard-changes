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

## Writing a Terraform Version Switcher in PowerShell

If you're a developer who works with Terraform, you know how important it is to have the right version of Terraform installed for your project. But what if you need to work on multiple projects, each with a different version of Terraform? That's where a Terraform Version Switcher comes in.

In this post, we'll walk through how to write a Terraform Version Switcher in PowerShell. We'll cover how to install and switch between different versions of Terraform, similar to how `nvm` works for Node.js.

### Getting Started

To get started, we'll need to create a new PowerShell script. We'll call our script `tfv.ps1`. This script will contain all the functions we need to install and switch between different versions of Terraform.

### Installing Terraform Versions

The first thing we'll need to do is install different versions of Terraform. We'll create a function called `Install-TerraformVersion` that takes a version number as a parameter and downloads and installs the corresponding version of Terraform.

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

This function downloads the Terraform binary for the specified version and extracts it to the `$env:ProgramData\tfv` directory.

### Switching Between Terraform Versions

Next, we'll create a function called `Set-ActiveTerraformVersion` that sets the active version of Terraform. This function creates an alias for the `terraform` command that points to the appropriate version of Terraform based on the version you have set as active.

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

This function sets the active version of Terraform by creating an alias for the `terraform` command that points to the appropriate version of Terraform based on the version you have set as active.

### Listing Installed Terraform Versions

We'll also create a function called `Show-TerraformVersions` that displays all the versions of Terraform we have installed locally.

```powershell
function Show-TerraformVersions {
    $TerraformVersions = @(Get-ChildItem "$env:ProgramData\tfv" | Select-Object -ExpandProperty Name)

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

This function lists all the installed versions and highlights the active version.

### Setting the Active Terraform Version

Finally, we'll create a function called `Invoke-TerraformVersion` that allows you to set the active version of Terraform, list installed versions, and install new versions.

```powershell
function Invoke-TerraformVersion {
    param(
        [Parameter(Mandatory = $false, Position = 0)]
        [ValidateSet("current", "install", "list", "use", "help" )]
        [string] $Command,
        [Parameter(Mandatory = $false, Position = 1)]
        [string] $Arg
    )

    if ([String]::IsNullOrWhiteSpace($Command)) {
        $Command = "current"
    }

    switch ($Command) {
        "current" {
            Get-ActiveTerraformVersion
        }
        "install" {
            Install-TerraformVersion -Version $Arg
        }
        "list" {
            Show-TerraformVersions
        }
        "use" {
            Set-ActiveTerraformVersion -Version $Arg
        }
        "help" {
            Write-Host "Usage:"
            Write-Host "  tfv current             " -NoNewLine -Foreground Cyan; Write-Host ": Show the currently active Terraform version"
            Write-Host "  tfv install <version>   " -NoNewLine -Foreground Cyan; Write-Host ": Install a new version of Terraform"
            Write-Host "  tfv list                " -NoNewLine -Foreground Cyan; Write-Host ": List all installed versions of Terraform"
            Write-Host "  tfv use <version>       " -NoNewLine -Foreground Cyan; Write-Host ": Set the active version of Terraform"
            Write-Host "  tfv help                " -NoNewLine -Foreground Cyan; Write-Host ": Show this help message"
        }
    }
}

Set-Alias tfv Invoke-TerraformVersion
```

This function allows you to set the active version of Terraform using the `use` command, list installed versions using the `list` command, and install new versions using the `install` command.

### How to use it

To use the Terraform Version Switcher, you'll need to have PowerShell installed on your machine. Once you have PowerShell installed, you can download the script and run it in a PowerShell console.

Here are the basic commands you'll need to know:

- `tfv use <version>`: Set the active version of Terraform
- `tfv list`: List all installed versions of Terraform
- `tfv install <version>`: Install a new version of Terraform
- `tfv uninstall <version>`: Uninstall a version of Terraform

For more information on how to use the Terraform Version Switcher, you can run the `tfv help` command.

### Conclusion

In this post, we walked through how to write a Terraform Version Switcher in PowerShell. We covered how to install and switch between different versions of Terraform, similar to how nvm works for Node.js. With this tool, you can ensure that you always have the right version of Terraform installed for your project.
