---
title: Setting up .NET Core
author: Evan Kaiser
tags:
  - .NET Core
  - Setup Scripts
---

This posts details setting up your environment for .NET Core development, specifically for REST APIs. First, I'll go over the different tools needed to follow along in this series. 

***
This is an ongoing series on .NET Core.

1. Setting up .NET Core
2. [.NET Core REST API in About 30 Minutes](../dotnet-core-rest-api-in-about-30-minutes)

## Tools

:bulb: *Tip*

If you want to skip over the manual installation, you can skip to "Scripts", containing scripts for installing all the tools on Windows 10 or Mac OS (Disclaimer: While I'd love to include Linux, due to the lack of concise package management and variables based on build for Linux, I will assume those on Linux are used to figuring out how to download tools on their own and not go through the machinations of including it here).

***

### VS Code

The first thing you'll want to do is download VS Code if you do not have it already. It can be obtained at [https://code.visualstudio.com/](https://code.visualstudio.com/)

### .NET Core

Next, download the .NET Core 2.0 SDK at [https://www.microsoft.com/net/download/core](https://www.microsoft.com/net/download/core) or use [this direct link for Windows x64](https://download.microsoft.com/download/7/3/A/73A3E4DC-F019-47D1-9951-0453676E059B/dotnet-sdk-2.0.2-win-x64.exe).

### Postman

Using [Postman](https://www.getpostman.com/) will allow you to quickly perform requests to the API.

## Scripts

### Windows 10 (Powershell)

This Windows 10 script uses [Chocolatey](https://chocolatey.org/install) as the install tool to download and install all of the tools mentioned here.

```powershell
# Install and upgrade Chocolatey
Set-ExecutionPolicy Bypass -Scope Process -Force; iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
choco upgrade chocolatey

# Install Tools
cinst visualstudiocode -y
cinst dotnetcore-sdk -y
cinst postman -y
```

### Mac OS

This Mac OS script uses [Homebrew](https://brew.sh/) along with a "tap" known as [Homebrew Cask](https://caskroom.github.io/) which is used for a lot of apps not found in Homebrew itself.

```bash
# Install Homebrew and Homebrew Cask
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
brew tap caskroom/cask

# Install Tools
brew cask install visual-studio-code
brew cask install dotnet-sdk
brew cask install postman
```
