Assess-SQLServer.ps1
=====================

Automated health, performance, and configuration assessment for SQL Server instances.

## Overview

`Assess-SQLServer.ps1` is a PowerShell script that connects to a target SQL Server instance, runs a curated set of DMV and metadata queries, and exports the results as CSV files for review and offline analysis. It focuses on core areas such as waits, I/O, memory, configuration, and high availability topology, and can optionally package all outputs into a timestamped ZIP archive.

The script is intended for DBAs, architects, and support engineers who need a quick, repeatable way to capture the current state of a SQL Server instance for troubleshooting or health checks.

## Features

- Collects key SQL Server metrics via DMVs (wait stats, file I/O, execution stats, memory, index usage, HA status, and more).
- Captures OS CPU specs and basic server information alongside SQL Server metadata.
- Exports each result set as a separate CSV file in a chosen folder.  
- Optionally compresses all CSV outputs into a ZIP file, with an optional timestamp for easy archival.  
- Supports Windows authentication and SQL authentication.  
- Automatically installs and imports the `SqlServer` PowerShell module if missing, then uses `Invoke-Sqlcmd` for all queries.

## Requirements

- PowerShell on Windows.  
- Script must be run **as Administrator** (`#Requires -RunAsAdministrator`).  
- `SqlServer` PowerShell module (installed automatically if not present).
- The executing login on the SQL Server instance needs:  
  - `VIEW SERVER STATE` permission.  
  - `VIEW DATABASE STATE` permission (for the target database).

## Collected Outputs

For a given `-server` value (for example `myserver`), the script generates the following CSV files in the destination folder:

- `myserver-SQLServerWaitStats.csv` – top waits with derived metrics and potential impact.  
- `myserver-SQLServerFiles.csv` – file-level I/O stats from `sys.dm_io_virtual_file_stats`.  
- `myserver-SQLServerExecutionPlanStats.csv` – high-cost queries by average reads and execution time.  
- `myserver-SQLServerOSinfo.csv` – basic OS/SQL instance environment (cores, memory, uptime).  
- `myserver-SQLServerInfo.csv` – instance version, edition, build, clustering info.  
- `myserver-SQLServerMemoryState.csv` – OS memory state from `sys.dm_os_sys_memory`.  
- `myserver-SQLServerIndexUsageStats.csv` – index/DB read vs write activity profile.  
- `myserver-SQLServerConfig.csv` – selected configuration values (e.g., MAXDOP, cost threshold).  
- `myserver-SQLServerPhysicalIO.csv` – physical read/write distribution per database.  
- `myserver-SQLServerHAConfig.csv` – AG / FCI / standalone HA characterization.  
- `myserver-CpuSpecs.csv` – CPU model and max clock speed from WMI.  

If ZIP creation is enabled, all CSVs in the destination folder are compressed into:

- `myserver-SQLAssessment.zip`  
- or `myserver-SQLAssessment_yyyyMMdd_HHmmss.zip` (when `-IncludeTimestamp` is used).  

## Parameters

```powershell
param(
    [Parameter(Mandatory = $true)]
    [string]$server,

    [Parameter(Mandatory = $false)]
    [string]$database = "master",
    
    [Parameter(Mandatory = $false)]
    [string]$DestinationFolder = "C:\Temp",
    
    [Parameter(Mandatory = $false)]
    [switch]$IncludeTimestamp,

    [Parameter(Mandatory = $false)]
    [switch]$UseSqlAuthentication,
    
    [Parameter(Mandatory = $false)]
    [string]$SqlUser,

    [Parameter(Mandatory = $false)]
    [string]$SqlPassword
)
```

- `-server`  
  SQL Server instance or listener name to connect to (for example `SQLPROD01`, `SQLPROD01\INST1`, or an AG listener).

- `-database`  
  Database to connect to; defaults to `master`. Many DMV queries are instance-scoped, but this is required for connectivity.  

- `-DestinationFolder`  
  Folder where CSV and ZIP files will be written. Defaults to `C:\Temp`. The folder is created if it does not exist.  

- `-IncludeTimestamp`  
  When specified, appends a `yyyyMMdd_HHmmss` timestamp to the generated ZIP file name.  

- `-UseSqlAuthentication`  
  Switch to use SQL authentication instead of Windows authentication. When set, `-SqlUser` and `-SqlPassword` should be provided.

- `-SqlUser` / `-SqlPassword`  
  SQL login credentials used when `-UseSqlAuthentication` is enabled.  

## Usage

### 1. Clone or download

Clone this repository or download the script file:

```powershell
git clone https://github.com/barisfurtinalar/tools/Assess-SQLServer.ps1.git
cd <your-repo>
```

Ensure `Assess-SQLServer.ps1` is present in the working directory.

### 2. Run with Windows authentication

```powershell
.\Assess-SQLServer.ps1 `
    -server "node1.cobra.kai" `
    -DestinationFolder "C:\Temp" `
    -IncludeTimestamp
```

This will:

- Connect to `node1.cobra.kai` using your current Windows credentials.  
- Run all assessment queries.  
- Export CSV files into `C:\Temp`.  
- Create a ZIP file named similar to `node1.cobra.kai-SQLAssessment_20251208_101500.zip`.  

### 3. Run with SQL authentication

```powershell
.\Assess-SQLServer.ps1 `
    -server "node1.cobra.kai" `
    -DestinationFolder "C:\Temp" `
    -UseSqlAuthentication `
    -SqlUser "sa" `
    -SqlPassword "YourPassword!" `
    -IncludeTimestamp
```

This uses SQL authentication instead of Windows integrated authentication to connect to the instance.

## How it works

1. Ensures the destination folder exists; creates it if needed.  
2. Ensures the `SqlServer` module is installed and imported so `Invoke-Sqlcmd` is available.
3. Builds a common parameter set (`$sqlParams`) used to call `Invoke-Sqlcmd` with either Windows or SQL authentication.  
4. Sequentially executes a set of T‑SQL queries against DMVs and system views, exporting each result set to its own CSV file.  
5. Queries CPU details via `Get-CimInstance Win32_Processor` and exports the result.  
6. Compresses all CSV files in the destination folder into a single ZIP file (optionally timestamped) and reports the resulting file size.  

## Notes and recommendations

- Run from an elevated PowerShell session (Run as Administrator) due to `#Requires -RunAsAdministrator` and WMI access.
- For production environments, consider storing credentials securely (for example, in the Windows Credential Manager or using `Get-Credential` and `ConvertFrom-SecureString`) instead of plain-text `-SqlPassword`.
- Review CSV outputs in Excel, Power BI, or your preferred analysis tool to trend waits, I/O, and configuration across servers.  
- This script is read-only against SQL Server (DMV and metadata queries only); it does not modify configuration or objects.  

## License

Add your preferred license here (for example MIT, Apache 2.0, etc.) and include the corresponding `LICENSE` file in the repository.

