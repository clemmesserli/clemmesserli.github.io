---
layout: post
title: "Messkit Series: Working with Google Authenticator"
date: 2025-07-18 10:00:00 +0000
categories: [PowerShell, Cybersecurity]
tags: [messkit, scripting, automation, security, 2fa, totp, google-authenticator]
---

### Introduction

In an era where cybersecurity breaches make headlines daily, implementing robust authentication mechanisms is no longer optional—it's essential. Two-Factor Authentication (2FA) using Time-based One-Time Passwords (TOTP) has become common for securing digital accounts, with Google Authenticator serving as one of the top 5 most utilized authentication apps.

Today, we'll explore two powerful MessKit functions that bring Google Authenticator functionality directly into your PowerShell environment: `New-MKGoogleAuthSecret` and `Get-MKGoogleAuthPin`. Whether you're a system administrator setting up 2FA for users, a developer integrating authentication into applications, or a security professional testing TOTP implementations, these functions provide the tools you need.

### Understanding TOTP and Google Authenticator

Before diving into the functions, let's understand what we're working with:

**Time-based One-Time Password (TOTP)** is an algorithm that generates a unique numeric code based on:

- A shared secret key
- The current time
- A predefined time window (usually 30 seconds)

**Google Authenticator** implements the TOTP standard (RFC 6238) and generates 6-digit codes that refresh every 30 seconds. The magic happens through cryptographic operations using HMAC-SHA1, ensuring that both the server and authenticator app generate identical codes when synchronized.

### The MessKit Google Authenticator Functions

Our MessKit module provides two complementary functions:

1. **`New-MKGoogleAuthSecret`** - Creates new TOTP secrets with QR codes for easy setup
2. **`Get-MKGoogleAuthPin`** - Generates current PIN codes from existing secrets

These functions follow the same TOTP standards used by Google Authenticator, Microsoft Authenticator, and other major authentication apps, ensuring full compatibility.

### Getting Started: Creating Your First Secret

Let's start by generating a new Google Authenticator secret:

````powershell
# Create a new secret for a user account
$auth = New-MKGoogleAuthSecret -AccountName "john.doe@company.com" -Issuer "MyCompany"

# Display the secret information
$auth | Format-List

Secret      : MFRGG2LTMREXGMDJOBZWE4LOMNQXIZLFOR2GCY3PNZWWK
AccountName : john.doe@company.com
Issuer      : MyCompany
Algorithm   : SHA1
Digits      : 6
Period      : 30
KeyUri      : otpauth://totp/MyCompany:john.doe@company.com?secret=MFRGG...
QrCodeUri   : https://api.qrserver.com/v1/create-qr-code/?size=300x300...
````

The function generates a cryptographically secure 20-byte (160-bit) secret and provides everything needed for setup, including a QR code URL that users can scan with their authenticator app.

### Generating PIN Codes

Once you have a secret, generating the current PIN is straightforward:

````powershell
# Generate the current PIN
$pin = Get-MKGoogleAuthPin -Secret $auth.Secret

# Display the result
$pin

PinCode          : 123 456
RawPin           : 123456
SecondsRemaining : 23
ExpiresAt        : 1/18/2025 2:15:30 PM
Algorithm        : SHA1
Digits           : 6
````

The function returns both a formatted PIN (with space for readability) and the raw PIN for programmatic use, along with timing information to know when the code expires.

### Understanding the Parameters

#### New-MKGoogleAuthSecret Parameters

- **`-AccountName`** (Required): Usually an email address that identifies the account
- **`-Issuer`** (Required): Your company or service name
- **`-SecretLength`**: Secret size in bytes (default: 20 for enhanced security)
- **`-Algorithm`**: HMAC algorithm - SHA1 (default), SHA256, or SHA512
- **`-Digits`**: PIN length - 6 (default) or 8 digits
- **`-Period`**: Time window in seconds (default: 30)
- **`-Online`**: Automatically opens QR code in browser
- **`-UseThisSecretCode`**: Use an existing secret instead of generating new

#### Get-MKGoogleAuthPin Parameters

- **`-Secret`** (Required): The BASE32 encoded secret
- **`-TimeWindow`**: Time window in seconds (default: 30)
- **`-Algorithm`**: Must match the secret's algorithm
- **`-Digits`**: Must match the secret's digit count
- **`-NoFormatting`**: Return PIN without space formatting

### Real-World Usage Scenarios

#### Scenario 1: Setting Up 2FA for New Employees

````powershell
# Bulk setup for new employees
$newEmployees = @(
    @{Email="alice@company.com"; Department="IT"}
    @{Email="bob@company.com"; Department="Sales"}
    @{Email="carol@company.com"; Department="HR"}
)

$employees = $newEmployees | ForEach-Object {
    $auth = New-MKGoogleAuthSecret -AccountName $_.Email -Issuer "CompanyPortal"
    [PSCustomObject]@{
        Employee = $_.Email
        Department = $_.Department
        Secret = $auth.Secret
        QRCode = $auth.QrCodeUri
        SetupUri = $auth.KeyUri
    }
}

# Export setup information for IT team
$employees | Export-Csv "2FA_Setup_$(Get-Date -Format 'yyyyMMdd').csv" -NoTypeInformation

# Display QR codes for immediate setup
$employees | ForEach-Object {
    Write-Host "Setup 2FA for $($_.Employee):" -ForegroundColor Green
    Write-Host "QR Code: $($_.QRCode)" -ForegroundColor Cyan
    Write-Host "Manual Entry: $($_.Secret)" -ForegroundColor Yellow
    Write-Host "---"
}
````

#### Scenario 2: PIN Monitoring and Validation

````powershell
# Monitor PIN changes in real-time
function Start-PINMonitor {
    param([string]$Secret, [string]$AccountName)

    Write-Host "Monitoring TOTP codes for: $AccountName" -ForegroundColor Green
    Write-Host "Press Ctrl+C to stop`n" -ForegroundColor Yellow

    while ($true) {
        $pin = Get-MKGoogleAuthPin -Secret $Secret
        Clear-Host

        Write-Host "Account: $AccountName" -ForegroundColor Cyan
        Write-Host "Current PIN: $($pin.PinCode)" -ForegroundColor Green -NoNewline

        # Color code based on time remaining
        $color = if ($pin.SecondsRemaining -le 5) { "Red" }
                elseif ($pin.SecondsRemaining -le 10) { "Yellow" }
                else { "Green" }

        Write-Host " (Expires in $($pin.SecondsRemaining)s)" -ForegroundColor $color
        Write-Host "Next refresh: $($pin.ExpiresAt.ToString('HH:mm:ss'))" -ForegroundColor Gray

        Start-Sleep 1
    }
}

# Start monitoring
Start-PINMonitor -Secret "JBSWY3DPEHPK3PXP" -AccountName "test@example.com"
````

#### Scenario 3: Batch PIN Generation for Multiple Accounts

````powershell
# Validate multiple accounts at once
$accounts = @(
    @{Name="Work Email"; Secret="T5UOE4YLIFYTZQA2"}
    @{Name="Personal Gmail"; Secret="JBSWY3DPEHPK3PXP"}
    @{Name="Azure Account"; Secret="MFRGG2LTMREXGMDJ"}
)

$accounts | ForEach-Object {
    $pin = Get-MKGoogleAuthPin -Secret $_.Secret
    [PSCustomObject]@{
        Account = $_.Name
        PIN = $pin.RawPin
        ExpiresAt = $pin.ExpiresAt.ToString("HH:mm:ss")
        TimeRemaining = "$($pin.SecondsRemaining)s"
    }
} | Format-Table -AutoSize

Account         PIN    ExpiresAt TimeRemaining
-------         ---    --------- -------------
Work Email      123456 14:32:15  23s
Personal Gmail  789012 14:32:15  23s
Azure Account   345678 14:32:15  23s
````

#### Scenario 4: Automated PIN to Clipboard

````powershell
# Save PIN to clipboard for easy pasting
function Copy-TOTPPin {
    param([string]$Secret, [string]$AccountName)

    $pin = Get-MKGoogleAuthPin -Secret $Secret
    $pin.RawPin | Set-Clipboard

    Write-Host "✓ PIN $($pin.PinCode) copied to clipboard" -ForegroundColor Green
    Write-Host "  Account: $AccountName" -ForegroundColor Cyan
    Write-Host "  Expires in: $($pin.SecondsRemaining) seconds" -ForegroundColor Yellow

    return $pin
}

# Usage
Copy-TOTPPin -Secret "JBSWY3DPEHPK3PXP" -AccountName "GitHub"
````

### Advanced Features and Customization

#### Custom Time Windows and Algorithms

Some services use non-standard TOTP configurations:

````powershell
# Create secret with 60-second time window and SHA256
$customAuth = New-MKGoogleAuthSecret `
    -AccountName "admin@secure-service.com" `
    -Issuer "SecureService" `
    -Period 60 `
    -Algorithm "SHA256" `
    -Digits 8

# Generate PIN with matching parameters
$customPin = Get-MKGoogleAuthPin `
    -Secret $customAuth.Secret `
    -TimeWindow 60 `
    -Algorithm "SHA256" `
    -Digits 8

Write-Host "8-digit PIN with 60s window: $($customPin.RawPin)"
````

#### Working with Existing Secrets

Sometimes you need to work with secrets generated elsewhere:

````powershell
# Use an existing secret from another system
$existingSecret = "GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ"

$auth = New-MKGoogleAuthSecret `
    -UseThisSecretCode $existingSecret `
    -AccountName "migrated-user@company.com" `
    -Issuer "MigratedSystem"

# Generate QR code for the existing secret
Write-Host "QR Code for existing secret: $($auth.QrCodeUri)"
````

#### Built-in PIN Generation Method

The secret object includes a convenient method for PIN generation:

````powershell
# Create secret
$auth = New-MKGoogleAuthSecret -AccountName "test@example.com" -Issuer "TestApp"

# Use built-in method (equivalent to Get-MKGoogleAuthPin)
$pin = $auth.GetCurrentPin()
Write-Host "Current PIN: $($pin.PinCode)"
````

### Integration with Password Managers and Scripts

#### Creating Setup Documents

````powershell
# Generate comprehensive setup documentation
function New-2FASetupDoc {
    param([string]$AccountName, [string]$Issuer)

    $auth = New-MKGoogleAuthSecret -AccountName $AccountName -Issuer $Issuer

    $doc = @"
# 2FA Setup Instructions for $AccountName

## Account Information
- **Account**: $($auth.AccountName)
- **Service**: $($auth.Issuer)
- **Algorithm**: $($auth.Algorithm)
- **Digits**: $($auth.Digits)
- **Period**: $($auth.Period) seconds

## Setup Methods

### Option 1: QR Code (Recommended)
1. Open your authenticator app
2. Scan this QR code: $($auth.QrCodeUri)

### Option 2: Manual Entry
1. Open your authenticator app
2. Choose "Enter setup key manually"
3. Enter this secret: `$($auth.Secret)`

## Backup Information
- **Secret Key**: $($auth.Secret)
- **Setup URI**: $($auth.KeyUri)

## Test Code
Current PIN: $(($auth.GetCurrentPin()).PinCode)
(Generated at $(Get-Date))

---
*Keep this information secure and accessible only to authorized personnel.*
"@

    return $doc
}

# Generate setup documentation
$setupDoc = New-2FASetupDoc -AccountName "ceo@company.com" -Issuer "ExecutivePortal"
$setupDoc | Out-File "2FA_Setup_CEO.md" -Encoding UTF8
````

#### Automated Testing and Validation

````powershell
# Test TOTP implementation against known test vectors
function Test-TOTPImplementation {
    # RFC 6238 test vectors
    $testVectors = @(
        @{Secret="GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ"; Time=59; Expected="94287082"}
        @{Secret="GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ"; Time=1111111109; Expected="07081804"}
        @{Secret="GEZDGNBVGY3TQOJQGEZDGNBVGY3TQOJQ"; Time=1111111111; Expected="14050471"}
    )

    foreach ($vector in $testVectors) {
        # Note: This requires modifying the function to accept specific timestamps
        # for testing purposes (not shown in the main implementation)
        Write-Host "Testing vector for time $($vector.Time)..." -ForegroundColor Yellow
        Write-Host "Expected: $($vector.Expected)" -ForegroundColor Gray
        # Implementation would generate PIN for specific timestamp here
    }
}
````

### Security Considerations and Best Practices

#### Secret Storage and Handling

- **Never log secrets in plain text** - Use secure storage mechanisms
- **Implement proper access controls** - Limit who can generate/view secrets
- **Use secure random generation** - The functions use cryptographically secure RNG
- **Consider secret rotation policies** - Regularly update secrets for high-security accounts

#### Backup and Recovery

````powershell
# Create encrypted backup of 2FA secrets
function Backup-2FASecrets {
    param([array]$Accounts, [string]$BackupPath)

    $backup = $Accounts | ConvertTo-Json -Depth 3
    $secureBackup = $backup | ConvertTo-SecureString -AsPlainText -Force
    $encryptedBackup = $secureBackup | ConvertFrom-SecureString

    $encryptedBackup | Out-File $BackupPath
    Write-Host "✓ Encrypted backup saved to: $BackupPath" -ForegroundColor Green
}

# Usage
$accounts = @(
    @{Name="Service1"; Secret="SECRET1"; Account="user@domain.com"}
    @{Name="Service2"; Secret="SECRET2"; Account="admin@domain.com"}
)

Backup-2FASecrets -Accounts $accounts -BackupPath "2FA_Backup_$(Get-Date -Format 'yyyyMMdd').txt"
````

### Troubleshooting Common Issues

#### Clock Synchronization

TOTP codes are time-sensitive. If codes don't match:

````powershell
# Check time synchronization
$ntpTime = (Get-Date)
Write-Host "System time: $ntpTime"
Write-Host "Verify this matches your authenticator app's time"

# Generate PIN with time tolerance
$pin = Get-MKGoogleAuthPin -Secret "YOUR_SECRET"
Write-Host "Current window PIN: $($pin.RawPin)"
Write-Host "Consider ±30 second tolerance for clock drift"
````

#### Secret Validation

````powershell
# Validate BASE32 secret format
function Test-Base32Secret {
    param([string]$Secret)

    $cleanSecret = $Secret.ToUpper() -replace '[^A-Z2-7]', ''

    if ($cleanSecret.Length -eq 0) {
        Write-Warning "Invalid secret: Contains no valid BASE32 characters"
        return $false
    }

    if ($Secret -ne $cleanSecret) {
        Write-Warning "Secret contains invalid characters, cleaned: $cleanSecret"
    }

    Write-Host "✓ Valid BASE32 secret: $cleanSecret" -ForegroundColor Green
    return $true
}

# Test a secret
Test-Base32Secret -Secret "JBSWY3DPEHPK3PXP"
````

### Conclusion

The `New-MKGoogleAuthSecret` and `Get-MKGoogleAuthPin` functions bring enterprise-grade TOTP functionality directly to your PowerShell environment. Whether you're automating 2FA deployment, integrating authentication into applications, or managing security for multiple accounts, these tools provide the reliability and compatibility you need.

Key benefits include:

- **Full RFC 6238 compliance** ensuring compatibility with all major authenticator apps
- **Cryptographically secure secret generation** using proper random number generators
- **Flexible configuration** supporting different algorithms, time windows, and PIN lengths
- **Easy QR code generation** for seamless user onboarding
- **Comprehensive output** providing both human-readable and programmatic interfaces

### Next Steps

- Experiment with different TOTP configurations to understand their impact
- Integrate these functions into your user onboarding workflows
- Build monitoring tools to track 2FA adoption across your organization
- Consider creating wrapper functions for your specific organizational needs

### Security Reminder

Always handle TOTP secrets with the same care as passwords. Store them securely, limit access appropriately, and consider implementing audit logging for secret generation and access.

---

*This post is part of my PowerShell MessKit module series. Stay tuned for more articles as I continue exploring the security and utility functions that make system administration more efficient and secure.*
