# SignScript

A PowerShell module for Authenticode-signing PowerShell and VBScript files with a code signing certificate stored in **Azure Key Vault**.

Instead of keeping a PFX on disk or in a local certificate store, the certificate is fetched from Key Vault at signing time, used in memory, and disposed right after. Every signing action is written to a central log file with timestamp and user.

## Features

- Signs `.ps1`, `.psm1`, `.psd1` and `.vbs` files
- Retrieves the signing certificate from Azure Key Vault on demand
- SHA256 signatures with RFC 3161 timestamping (DigiCert)
- Reuses an existing Azure session and reconnects automatically when the token is no longer valid
- Auditable log of every signing attempt (success and failure)
- Works on Windows PowerShell 5.1 and PowerShell 7

## Requirements

| Requirement | Details |
|---|---|
| PowerShell | 5.1 or later (Desktop or Core) |
| Az.Accounts | Version **2.17.0** (installed automatically for the current user if missing) |
| Azure Key Vault | Containing a code signing certificate |
| Azure permissions | Read access to the certificate's secret (see below) |
| Network | Outbound access to `*.vault.azure.net`, `login.microsoftonline.com` and `timestamp.digicert.com` |

### Why Az.Accounts 2.17.0?

The module reads the access token as a plain string (`$token.Token`). Newer Az.Accounts releases return the token as a `SecureString` by default, so the version is pinned to keep the REST call working as written.

## Azure setup

1. **Import the code signing certificate** into your Key Vault. The certificate must have the Code Signing EKU (`1.3.6.1.5.5.7.3.3`).
2. **Make the private key exportable.** The module downloads the PFX through the certificate's secret endpoint, which only returns the private key when the key is marked exportable.
3. **Grant access** to the identity that will sign scripts:
   - Azure RBAC: assign the **Key Vault Secrets User** role on the vault, or
   - Access policies: grant **Get** on **Secrets**.

## Installation

Copy the module folder to one of your module paths. The folder name must match the module name.

```
SignScript/
├── SignScript.psd1
└── SignScript.psm1
```

**Windows PowerShell 5.1**

```powershell
Copy-Item -Path .\SignScript -Destination "$HOME\Documents\WindowsPowerShell\Modules\" -Recurse
```

**PowerShell 7**

```powershell
Copy-Item -Path .\SignScript -Destination "$HOME\Documents\PowerShell\Modules\" -Recurse
```

## Configuration

The module ships with placeholders. Fill these in before use:

| Setting | Function | Value |
|---|---|---|
| `$subscriptionId` | `Initialize-AzureEnvironment` | ID of the subscription that holds the Key Vault |
| `$logPath` | `Write-SigningLog` | Folder for the signing log, for example `C:\Logs\CodeSigning` or a file share |
| `-ResourceUrl` | `Sign-Script` | `https://vault.azure.net` |
| `$secretUri` | `Sign-Script` | `https://<vault-name>.vault.azure.net/secrets/<certificate-name>?api-version=7.4` |
| `$certUri` | `Sign-Script` | Optional, currently not used |
| `GUID` | `SignScript.psd1` | Generate a unique one with `New-Guid` |

> The Key Vault REST API requires the `api-version` query parameter. Without it the request fails with a `400 Bad Request`.

> **Do not commit your filled-in values.** Keep subscription IDs, vault names and internal log paths out of the public repository.

## Usage

### Sign a single file

```powershell
Import-Module SignScript
Sign-Script -FilePath "C:\Packages\MyApp\Invoke-AppDeployToolkit.ps1"
```

On the first run you are prompted to sign in to Azure. Later runs reuse the existing session as long as the token is valid.

### Sign all scripts in a folder

```powershell
Get-ChildItem -Path "C:\Packages\MyApp" -Recurse -Include *.ps1, *.psm1, *.psd1, *.vbs |
    ForEach-Object { Sign-Script -FilePath $_.FullName }
```

### Verify a signature

```powershell
Get-AuthenticodeSignature -FilePath "C:\Packages\MyApp\Invoke-AppDeployToolkit.ps1" |
    Format-List Status, SignerCertificate, TimeStamperCertificate
```

### Output

`Sign-Script` returns the `Signature` object from `Set-AuthenticodeSignature`, so you can use it in a pipeline or check `.Status` in your own scripts.

## Logging

All activity is appended to `ScriptSigningLogs.log` in the configured log folder, in this format:

```
yyyy-MM-dd HH:mm:ss|DOMAIN\user|Message
```

Example:

```
2025-03-14 09:12:44|CONTOSO\jdoe|Starting to sign script: C:\Packages\MyApp\Deploy.ps1
2025-03-14 09:12:46|CONTOSO\jdoe|Successfully signed script: C:\Packages\MyApp\Deploy.ps1 with certificate: CN=Contoso Code Signing
2025-03-14 09:12:46|CONTOSO\jdoe|Certificate disposed
```

Failures (invalid file type, missing file, Azure errors, invalid signature status) are logged with an `ERROR:` prefix, including inner exception details when available.

## How it works

1. Validates the file extension and that the file exists.
2. Ensures Az.Accounts 2.17.0 is installed and loaded.
3. Reuses the current Azure context, or connects / reconnects when needed, then sets the subscription.
4. Requests a Key Vault access token and downloads the certificate (PFX, base64) from the secret endpoint.
5. Loads the certificate in memory and signs the file with SHA256 and a DigiCert timestamp.
6. Fails if the resulting signature status is not `Valid`.
7. Disposes the certificate object and logs the result.

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `401` or `403` when reading the secret | The signed-in account lacks **Key Vault Secrets User** (or **Get** on secrets) |
| `400 Bad Request` | `api-version` missing from `$secretUri` |
| Signature status `UnknownError` | Certificate chain not trusted on the machine, or the certificate has no Code Signing EKU |
| Certificate loads without a private key | Private key is not exportable in Key Vault |
| Assembly or version conflicts with Az.Accounts | Another Az.Accounts version is already loaded. Start a new PowerShell session |
| Warning about unapproved verbs on import | `Sign` is not an approved PowerShell verb. Harmless; suppress with `Import-Module SignScript -DisableNameChecking` |

## Security notes

- The private key is downloaded to the signing machine's memory for the duration of the operation and disposed afterwards. Only run this on trusted, managed devices.
- Limit who can read the certificate's secret in Key Vault. Anyone with that permission can export the signing key.
- Review the signing log regularly to detect unexpected signing activity.

## Author

**Riyadh Sarker** · [TECHLAKE](https://techlake.be)

Copyright (c) 2025 TECHLAKE. All rights reserved.
