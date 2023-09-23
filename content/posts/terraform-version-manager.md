+++
draft = true
date = 2023-09-21T06:29:24Z
title = "Creating your own Terraform Version Manager with PowerShell"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

## Creating your own Terraform Version Manager with PowerShell

If you're a developer who works with Terraform, you know how important it is to have the right version of Terraform installed for your project. But what if you need to work on multiple projects, each with a different version of Terraform? That's where a Terraform version manager comes in.

There are several Terraform version managers available for Linux and Mac, including:

- [tfenv](https://github.com/tfutils/tfenv)
- [tfswitch](https://warrensbox.github.io/terraform-manager/)
- [tfupdate](https://github.com/minamijoyo/tfupdate)

These tools allow you to easily switch between different versions of Terraform on your local machine. These tools are great if you're using Linux or Mac, or if you're using Windows with WSL. However, there is currently no Terraform version manager available for PowerShell on Windows. Fortunately, it's easy to create your own Terraform version manager in PowerShell, and we're going to see how!

In this post, we'll walk through how to write your own Terraform Version Manager in PowerShell that allows you to easily install multiple versions of Terraform on your local machine and easily switch between them.

### Getting Started

To get started, we'll create a new PowerShell script named `tfv.ps1`. This script will contain all the functions we need to manage our different Terraform versions.

### Listing Available Terraform Versions

The first thing we'll need is a function to list the versions of Terraform available for us to install. We'll create a function called `Show-TerraformVersions` that gets a list of available versions from the HashiCorp website and displays them in a nice format.

```powershell
function Show-TerraformVersions {
    $Response = Invoke-WebRequest -Uri "https://releases.hashicorp.com/terraform" -Headers @{"accept" = "application/json" }
    $TerraformVersions = [Regex]::Matches($Response.Content, 'terraform_(.+?)</a>') | ForEach-Object { $_.Groups[1].Value }
    $TerraformVersions
}
```

{{< notice note >}}
I couldn't find a JSON API with a list of available versions, so we're just parsing the HTML of their releases page.
If you know of a better way to do this, please let me know in the comments!
{{< /notice >}}

### Installing Terraform Versions

Now that we can retrieve a list of available versions, we need a function that will install a given version.
To do this we'll create a function called `Install-TerraformVersion` that takes a version number as a parameter, downloads, and installs the corresponding version.

```powershell
function Install-TerraformVersion {
    param(
        [Parameter(Mandatory = $true)]
        [string] $Version
    )

    $TempFile = New-TemporaryFile | Rename-Item -NewName { $_ -replace 'tmp$', 'zip' } –PassThru
    Invoke-WebRequest -Uri "https://releases.hashicorp.com/terraform/$($Version)/terraform_$($Version)_windows_amd64.zip" -OutFile $TempFile
    $TempFile | Expand-Archive -DestinationPath "$env:ProgramData\tfv\$($Version)" -Force | Out-Null
    $TempFile | Remove-Item -Force
}
```

This function downloads and extracts the Terraform binary for the requested version to a version-specific directory under `$env:ProgramData\tfv\`.

### Setting the Active Terraform Version

Now that we have a way to install different versions of Terraform, we need a way to specify which one we want to use.
To do this, we'll create a function called `Set-ActiveTerraformVersion`.

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
    Set-Alias -Name terraform -Value $TerraformExePath -Scope Global
}
```

This function starts with a guard clause that will bail out if the requested version is not installed.
Otherwise, it tracks the active version in a file in the user's home directory `~/.tfv` and sets an alias for the `terraform` command that points to the correct version of the Terraform binary.

### That's the basics

At this point we have all we need to install and switch between different versions of Terraform. But, there are a few more things we can do to make our Terraform Version Manager even better.

### Getting the Active Terraform Version

It sure would be nice to know what the current active version is. You might think we should call `terraform -v`, but what if there is no active version set? In that case there'd be not `terraform` alias set, and we'd get an error.

Instead we'll create a function called `Get-ActiveTerraformVersion` that gets the active version from the `~/.tfv` file we created earlier, and handle the case when there is no version set.

```powershell
function Get-ActiveTerraformVersion {
    $Version = Get-Content -Path $TerraformVersionFile -ErrorAction SilentlyContinue
    if ([string]::IsNullOrWhitespace($Version)) {
        return "None"
    }
    return $Version
}
```

### Listing Installed Terraform Versions

Another essential feature we'll need is to see all the versions we have installed so we know what we can actually use. For this we're going to modify the `Show-TerraformVersions` function we created earlier by adding a `-LocalOnly` switch. When set, this function will simply list the contents of the `$env:ProgramData\tfv` directory. We'll include another guard clause to handle when there are no versions installed.

And while we're at it, let's make the output a little nicer by highlighting the active version.

```powershell
function Show-TerraformVersions {
    param(
        [Parameter(Mandatory = $false)]
        [switch] $LocalOnly
    )

    if ($LocalOnly) {
        $TerraformVersions = @(Get-ChildItem "$env:ProgramData\tfv" | Select-Object -ExpandProperty Name)

        if ($TerraformVersions.Count -eq 0) {
            Write-Host "No Terraform versions installed." -ForegroundColor Red
            return
        }
    }
    else {
        $Response = Invoke-WebRequest -Uri "https://releases.hashicorp.com/terraform" -Headers @{"accept" = "application/json" }
        $TerraformVersions = [Regex]::Matches($Response.Content, 'terraform_(.+?)</a>') | ForEach-Object { $_.Groups[1].Value } 
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

### Uninstalling Terraform Versions

To complete our Terraform Version Switcher, we need a way to uninstall a version of Terraform. And what if we try to uninstall the active version? We'll need to handle that case too. We'll create two functions to handle this: `Uninstall-TerraformVersion` and `Remove-ActiveTerraformVersion`.

```powershell
function Remove-ActiveTerraformVersion {
    Remove-Item -Path "$env:UserProfile\.tfv" -ErrorAction SilentlyContinue    
    Remove-Alias -Name terraform -Scope Global -ErrorAction SilentlyContinue
}

function Uninstall-TerraformVersion {
    param(
        [Parameter(Mandatory = $true)]
        [string] $Version
    )

    if (!(Test-Path -Path "$env:ProgramData\.tfv\$Version")) {
        Write-Host "Terraform version $Version not installed!" -ForegroundColor Red
        return
    }

    if ($Version -eq (Get-ActiveTerraformVersion)) {
        Remove-ActiveTerraformVersion
    }

    Remove-Item -Path "$env:ProgramData\tfv\$Version" -Recurse -Force
    Write-Host "Terraform version $Version uninstalled." -ForegroundColor Green
}
```

### Putting it all together

At this point we have all the functions we need to install, uninstall, list, switch, set, and unset different versions of Terraform. But who's going to remember all those functions? We need a way to tie them all together into a single command.

We'll create a function called `Invoke-TerraformVersion` that takes two parameters: the `command` and an optional `argument`. Then we'll create a convenient alias for this function called `tfv`.

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
        { $_ -in "set","use" } {
            Set-ActiveTerraformVersion -Version $Arg
        }
        { $_ -in "unset","clear" } {
            Remove-ActiveTerraformVersion
        }
    }
}

Set-Alias tfv Invoke-TerraformVersion

# Set the active version if one is already set
if (Get-ActiveTerraformVersion -ne $null) {
    Set-ActiveTerraformVersion -Version (Get-ActiveTerraformVersion)
}
```

### How to use it

To use the Terraform Version Manager, you'll obviously need to have PowerShell installed on your machine. This code was written using PowerShell 7, so your results may vary if you're using an older version.

The best way to use this code is to create the `tfv.ps1` file in your PowerShell profile directory. Then invoke the script from your PowerShell profile. This will allow you to use the `tfv` command from any PowerShell session.

```powershell
$ProfileDir = Split-Path -Path $PROFILE -Parent
New-Item -Path "$ProfileDir\tfv.ps1" -ItemType File
'. ([IO.Path]::Combine($PSScriptRoot, "tfv.ps1"))' | Out-File -FilePath $PROFILE.CurrentUserAllHosts -Append

```

Here are the basic commands you'll need to know:

- `tfv use <version>`: Set the active version of Terraform
- `tfv list`: List all installed versions of Terraform
- `tfv install <version>`: Install a new version of Terraform
- `tfv uninstall <version>`: Uninstall a version of Terraform

For more information on how to use the Terraform Version Manager, you can run the `tfv help` command.

### Conclusion

In this post, we walked through how to write your own Terraform Version Manager in PowerShell that allows you to easily install and switch between different versions of Terraform on your local machine. We also covered how to use the Terraform Version Manager to install and switch between different versions of Terraform.

See the complete code for this post on [GitHub](
