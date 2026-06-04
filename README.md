# Windows Server Datacenter - GitHub Actions RDP Environment

A comprehensive configuration guide and GitHub Actions workflow designed to deploy a temporary, cloud-hosted Windows Server Datacenter environment. This project enables Remote Desktop Protocol (RDP) access to a GitHub-hosted runner utilizing a secure Ngrok network tunnel.

## 🚀 Purpose & Use Cases

This repository serves as an automated sandbox environment tailored for:
* **CI/CD Debugging:** Troubleshooting complex, GUI-dependent build steps or installer verification that cannot be diagnosed via standard console logs.
* **Environment Testing:** Verifying software behavior across clean enterprise Windows Server instances.
* **Pipeline Prototyping:** Rapidly testing script execution, registry modifications, or dependency installations before integrating them into main deployment pipelines.

---

## 📋 System Specifications

The environment deployed by GitHub Actions features the following baseline hardware and software characteristics:
* **Operating System:** Windows Server Datacenter (Latest standard GitHub-hosted image)
* **User Account:** `runneradmin` (Administrative privileges enabled)
* **Default Runtime Limit:** Up to 6 hours (`360 minutes`) per initialization
* **Network Protocol:** RDP over TCP (Port 3389) wrapped in an encrypted Ngrok tunnel

---

## 🛠️ Prerequisites

Before executing the workflow, ensure the following requirements are met:
1. **GitHub Account:** Access to a repository with GitHub Actions enabled.
2. **Ngrok Account:** A free or premium account from [ngrok.com](https://ngrok.com) to provide the external routing endpoint.
3. **Ngrok Authtoken:** Retrieved from your Ngrok dashboard under the "Your Authtoken" section.

---

## ⚙️ Setup & Deployment Guide

### Step 1: Configure Repository Secrets
To maintain network security and protect your tunneling credentials, you must store your Ngrok token securely within GitHub Secrets.

1. Navigate to your repository on GitHub.
2. Click on **Settings** in the top navigation bar.
3. In the left sidebar, expand **Secrets and variables** and select **Actions**.
4. Click the **New repository secret** button.
5. Configure the secret with the following details:
   * **Name:** `NGROK_AUTH_TOKEN`
   * **Secret:** *[Paste your actual Ngrok Authtoken here]*
6. Click **Add secret** to save.

### Step 2: Create the Workflow File
If you have not already added the workflow to your repository, create a file at `.github/workflows/rdp.yml` and insert the following automation script:

```yaml
name: Windows Server RDP

on:
  workflow_dispatch: # Enables manual triggering from the GitHub Actions UI

jobs:
  initialize-environment:
    runs-on: windows-latest
    timeout-minutes: 360 # Matches maximum allowed runner runtime

    steps:
    - name: Download and Extract Ngrok Binaries
      run: |
        Invoke-WebRequest [https://bin.equinox.io/c/b3419c53-1f1b-417d-94d2-4a572a13c544/ngrok-v3-stable-windows-amd64.zip](https://bin.equinox.io/c/b3419c53-1f1b-417d-94d2-4a572a13c544/ngrok-v3-stable-windows-amd64.zip) -OutFile ngrok.zip
        Expand-Archive ngrok.zip -DestinationPath .

    - name: Authenticate Ngrok Agent
      run: |
        .\ngrok.exe config add-authtoken ${{ secrets.NGROK_AUTH_TOKEN }}

    - name: Configure System RDP & Firewall Rules
      run: |
        # Enable Remote Desktop Connections
        Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server' -Name "fDenyTSConnections" -Value 0
        # Allow RDP Traffic Through Windows Defender Firewall
        Enable-NetFirewallRule -DisplayGroup "Remote Desktop"
        # Enable Network Level Authentication (NLA) for Secure Access
        Set-ItemProperty -Path 'HKLM:\System\CurrentControlSet\Control\Terminal Server\WinStations\RDP-Tcp' -Name "UserAuthentication" -Value 1

    - name: Initialize Network Tunnel & Keep-Alive
      run: |
        # Start the TCP tunnel over port 3389 in the background
        Start-Process .\ngrok.exe -ArgumentList "tcp 3389"
        
        Write-Host "=========================================================="
        Write-Host "ENVIRONMENT READY FOR CONNECTION"
        Write-Host "1. Fetch the active endpoint from your Ngrok Dashboard."
        Write-Host "2. Target Account Username: runneradmin"
        Write-Host "=========================================================="
        
        # Keeps the runner active to maintain the connection (6 Hours)
        Start-Sleep -Seconds 21600
