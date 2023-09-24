+++
draft = true
date = 2023-09-21T06:29:24Z
title = "Creating a Terraform Version Manager with PowerShell"
description = ""
slug = ""
authors = []
tags = []
categories = []
externalLink = ""
series = []
+++

If you're a developer who works with Terraform, you know how important it is to have the right version of Terraform installed for your project. But what if you need to work on multiple projects, each with a different version of Terraform? That's where a Terraform version manager comes in.

There are a couple Terraform version managers available for Linux and Mac, including:

- [tfenv](https://github.com/tfutils/tfenv)
- [tfswitch](https://github.com/warrensbox/terraform-switcher)

These tools allow you to easily switch between different versions of Terraform on your local machine. These tools are great if you're using Linux or Mac, or if you're using WSL on Windows. However, there are very few options available for those who prefer to use PowerShell on Windows.

In this post, we'll walk through how to write a Terraform Version Manager in PowerShell. We'll cover all the essential functionality so you'll have a comprehensive and user-friendly tool at your disposal.

## Overview

Before we dive into the actual code, let's take a look at the basic functionality we'll need to implement:

- List available versions on the HashiCorp website
- Install/uninstall a specific version
- Set/unset the active version
- List our installed versions
- Show the active version

## Let's Get Started

First things first, we'll create a new file called `tfv.ps1` in the same directory as our Powershell profile and open it in our favourite editor.

```powershell
$ScriptPath = "$(Split-Path -Path $PROFILE -Parent)\tfv.ps1"
New-Item -Path $ScriptPath -ItemType File
code $ScriptPath
```

We need a place to store the Terraform binaries we'll be installing, as well as a file to keep track of the current version. Let's use a few variables to keep track of these paths. We'll also ensure that these directories exist each time the script runs.

```powershell
$TfvBaseDirectory = "~/.tfv/"
$TfvInstallDirectory = "~/.tfv/versions"
$TfvCurrentVersionFile = "~/.tfv/current"

if (!(Test-Path -Path $TfvBaseDirectory)) {
    New-Item -ItemType Directory -Path $TfvBaseDirectory | Out-Null
}
if (!(Test-Path -Path $TfvInstallDirectory)) {
    New-Item -ItemType Directory -Path $TfvInstallDirectory | Out-Null
}
```

## Listing the Versions Available to Install

The first function we'll need is one that can list the available versions of Terraform on the HashiCorp website. This function will make an HTTP request to their [Releases](https://releases.hashicorp.com/terraform) page, then parse the HTML to get a list of version numbers.

```powershell
function Show-TerraformVersions {
    $Response = Invoke-WebRequest -Uri "https://releases.hashicorp.com/terraform"
    $TerraformVersions = [Regex]::Matches($Response.Content, '<a href=".+">terraform_(.+)</a>') | Select-Object { $_.Groups[1].Value }
    $TerraformVersions
}
```

{{< notice note >}}
I couldn't find a JSON API with a list of available versions, so we're just parsing the HTML. If you know a better way to do this, let me know in the comments!
{{< /notice >}}

## Installing a Specific Version

Now that we can retrieve a list of available versions, we need a function to install one. The Terraform releases are just zip files containing a single binary, so all we need to do is download the file, extract the binary, and place it in a version-specific directory in `~/.tfv/versions/`.

```powershell
function Install-Terraform {
    param(
        [Parameter(Mandatory = $true)]
        [string] $Version
    )

    $TempFile = New-TemporaryFile | Rename-Item -NewName { $_ -replace 'tmp$', 'zip' } –PassThru
    $Url = "https://releases.hashicorp.com/terraform/$Version/terraform_$($Version)_windows_amd64.zip"
    Invoke-WebRequest -Uri $Url -OutFile $TempFile
    $TempFile | Expand-Archive -DestinationPath "$TfvInstallDirectory/$Version" -Force | Out-Null
    $TempFile | Remove-Item -Force
}
```

## Setting the Active Version

Next we need a way to indicate which version we want to use. Let's create a function that takes a version number as a parameter. It will create an alias for the `terraform` command that points to the indicated version of the Terraform binary. We'll also keep track of the active version in the `~/.tfv/current` file.

```powershell
function Set-ActiveTerraformVersion {
    param(
        [Parameter(Mandatory = $true)]
        [string] $Version
    )

    # Check if the specified Terraform version is installed
    $TerraformPath = "$TfvInstallDirectory/$Version"
    if (!(Test-Path -Path $TerraformPath)) {
        throw "Terraform version $Version not installed"
    }

    # Store the active version in a file so it will persist between sessions
    Set-Content -Path $TfvCurrentVersionFile -Value $Version

    # Create aliases for the Terraform executable
    Set-Alias -Name terraform -Value "$TerraformPath/terraform.exe" -Scope Global
}
```

{{< notice info >}}
You _could_ stop here. This is all you really need to install and switch between different versions of Terraform. But we can do better. Let's add a few more functions to make this tool even more useful.
{{< /notice >}}

## Displaying the Active Terraform Version

In order to display the active version of Terraform we just need to read the value from the `~/.tfv/current` file. If the file doesn't exist, that means no version has been set, so we'll return `$null` and display a helpful warning message.

```powershell
function Get-ActiveTerraformVersion {
    if (!(Test-Path -Path $TfvCurrentVersionFile)) {
        Write-Warning 'No active Terraform version set! Use `tfv set <version>` to set the active version.'
        return $null
    }

    Get-Content -Path $TfvCurrentVersionFile
}
```

{{< notice warning >}}
You could use the `terraform -v` command directly, but when there is no active version the `terraform` alias will not exist. So invoking it will fail with an unfriendly error message. We want to display a helpful message instead.
{{< /notice >}}

## Listing Installed Versions

We'll definitely need a way to see what we already have installed. For this we'll modify the `Show-TerraformVersions` function we created earlier by adding a `-LocalOnly` switch. When the switch is set, we'll simply need to list the contents of the `~/.tfv/versions/` directory. We'll also highlight the active version if one is set.

Replace the `Show-TerraformVersions` function with the following:

```powershell
function Show-TerraformVersions {
    param(
        [Parameter(Mandatory = $false)]
        [switch] $LocalOnly
    )

    if ($LocalOnly) {
        $TerraformVersions = @(Get-ChildItem "$TfvInstallDirectory" -Directory | Select-Object -ExpandProperty Name)

        if ($TerraformVersions.Count -eq 0) {
            throw 'No Terraform versions installed! Use `tfv install <version>` to install a new version.'
        }
    }
    else {
        $Response = Invoke-WebRequest -Uri "https://releases.hashicorp.com/terraform"
        $TerraformVersions = [Regex]::Matches($Response.Content, '<a href=".+">terraform_(.+)</a>') | Select-Object { $_.Groups[1].Value }
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

## Uninstalling Terraform Versions

To complete our Terraform Version Switcher we need to have a way to uninstall a version we're no longer using. We'll also make sure to unset the active version if we're uninstalling it. We'll split this functionality into two functions: `Remove-ActiveTerraformVersion` and `Uninstall-Terraform`.

```powershell
function Remove-ActiveTerraformVersion {
    Remove-Alias -Name terraform -Scope Global -ErrorAction SilentlyContinue
}

function Uninstall-Terraform {
    param(
        [Parameter(Mandatory = $true)]
        [string] $Version
    )

    if (!(Test-Path -Path "$TfvInstallDirectory/$Version")) {
        throw "Terraform version $Version not installed!"
    }

    if ($Version -eq (Get-ActiveTerraformVersion)) {
        Remove-ActiveTerraformVersion
    }

    Remove-Item -Path "$TfvInstallDirectory/$Version" -Recurse -Force
}
```

## Putting it all together

At this point we have everything we need to `install`, `uninstall`, `list`, `set`, and `unset` different versions of Terraform. But who's going to remember all those functions? Not to mention the verbosity of typical PowerShell function names! What we need is a single command to handle it all.

Let's create one last function (_with another long-winded name_) called `Invoke-TerraformVersionManager` and create a much needed alias `tfv` to give us a simple entrypoint to all of this functionality.

```powershell
function Invoke-TerraformVersionManager {
    param(
        [Parameter(Mandatory = $false, Position = 0)]
        [ValidateSet("current", "install", "list", "ls", "ls-remote", "uninstall", "use", "set", "clear", "unset", "help" )]
        [string] $Command,
        [Parameter(Mandatory = $false, Position = 1)]
        [string] $Arg
    )

    # Default to "current" if no command is specified
    if ([String]::IsNullOrWhiteSpace($Command)) {
        $Command = "current"
    }

    switch ($Command) {
        "current" {
            Get-ActiveTerraformVersion
        }
        "install" {
            Install-Terraform -Version $Arg
        }
        { $_ -in "ls", "list" } {
            Get-TerraformVersions -LocalOnly
        }
        "ls-remote" {
            Get-TerraformVersions
        }
        "uninstall" {
            Uninstall-Terraform -Version $Arg
        }
        { $_ -in "set", "use" } {
            Set-ActiveTerraformVersion -Version $Arg
        }
        { $_ -in "unset", "clear" } {
            Remove-ActiveTerraformVersion
        }
        "help" {
            Show-Help
        }
        default {
            throw "Invalid command: $Command"
        }
    }
}

Set-Alias tfv Invoke-TerraformVersionManager
```

## Show Help Documentation

Oops! If you were looking closely, you may have noticed that we're missing a function called `Show-Help`. Let's add that now so we can see a helpful message when we run `tfv help`.

```powershell
function Show-Help {
    Write-Host "Usage:"
    Write-Host "  tfv current             " -NoNewLine -Foreground Cyan; Write-Host ": Show the currently active Terraform version"
    Write-Host "  tfv install <version>   " -NoNewLine -Foreground Cyan; Write-Host ": Install a new version of Terraform"
    Write-Host "  tfv uninstall <version> " -NoNewLine -Foreground Cyan; Write-Host ": Uninstall a version of Terraform"
    Write-Host "  tfv ls                  " -NoNewLine -Foreground Cyan; Write-Host ": List all installed versions of Terraform. [alias: tfv list]"
    Write-Host "  tfv ls-remote           " -NoNewLine -Foreground Cyan; Write-Host ": List all available versions of Terraform"
    Write-Host "  tfv set <version>       " -NoNewLine -Foreground Cyan; Write-Host ": Set the active version of Terraform [alias: tfv use]"
    Write-Host "  tfv unset               " -NoNewLine -Foreground Cyan; Write-Host ": Unset the active version of Terraform [alias: tfv clear]"
    Write-Host "  tfv help                " -NoNewLine -Foreground Cyan; Write-Host ": Show this help message"
}
```

## Set the Active Version on Startup

There's one last thing we must do before this tool is complete. Each time the script loads we need to reinitialize our `terraform` alias if there is a current version stored in the `~/.tfv/current` file. We can do this by adding the following code to the end of our script.

```powershell
# Set the active version if one is already set
if (Get-ActiveTerraformVersion) {
    Set-ActiveTerraformVersion -Version (Get-ActiveTerraformVersion)
}
```

## Loading the Script into our PowerShell Profile

To use our shiny new `tfv` command, we need to add the following line to your PowerShell profile. This will ensure that the `tfv` command is available in all of our PowerShell sessions.

```powershell
. ([IO.Path]::Combine($PSScriptRoot, "tfv.ps1"))

```

## Conclusion

In this post we learned how to create a Terraform Version Manager in PowerShell. We covered all the essential functionality so we'll have a comprehensive and user-friendly tool at our disposal.

Here are the basic commands we can now use:

- `tfv install <version>`: Install a new version of Terraform
- `tfv uninstall <version>`: Uninstall a version of Terraform
- `tfv ls`: List all installed versions of Terraform
- `tfv ls-remote`: List all available versions of Terraform
- `tfv set <version>`: Set the active version of Terraform
- `tfv unset`: Unset the active version of Terraform
- `tfv help`: Show the help message

## Next Steps

If you want to take this even further, here are a few ideas:

- Add support for Linux and Mac
- Add support for other HashiCorp tools (e.g. Packer, Vault, Consul)
- Add support for other tools (e.g. kubectl, helm, az, aws)

## Plans for the Future

I am working on providing this as a fully-functional and cross-platform PowerShell module available on the PowerShell Gallery. You can check out the [GitHub repo](https://github.com/mikesigs/tfv) to see the current progress and submit a Pull Request if you'd like to help out. I welcome any contributions and feedback to make this tool even better!
