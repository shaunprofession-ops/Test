@echo off
setlocal enabledelayedexpansion

:: ============================================================================
:: Configuration
:: ============================================================================

:: 1. DEFINE YOUR STARTING SERIAL NUMBERS HERE, SEPARATED BY SPACES.
set "SERIES_LIST=2223744, 2512262"

:: 2. Set how many bonds to find that are VALID and meet the date criteria.
set "VALID_REQUIRED_PER_SERIES=30"

:: 3. Set the cutoff year. Only bonds issued BEFORE this year will be counted.
set "TARGET_YEAR=2000"

:: 4. Other settings
set "REQUEST_DELAY=1"
set "OUTPUT_FILE=valid_bonds_before_%TARGET_YEAR%_%date:~-4,4%%date:~-10,2%%date:~-7,2%_%time:~0,2%%time:~3,2%.log"

:: ============================================================================
:: Main Script Logic
:: ============================================================================

:: Clean hour (fix space in single-digit hours)
set "HOUR=%time:~0,2%"
if "%HOUR:~0,1%"==" " set "HOUR=0%HOUR:~1,1%"
set "OUTPUT_FILE=valid_bonds_before_%TARGET_YEAR%_%date:~-4,4%%date:~-10,2%%date:~-7,2%_%HOUR%%time:~3,2%.log"

echo Starting bond validation...
echo.
echo Series Starters: %SERIES_LIST%
echo Finding %VALID_REQUIRED_PER_SERIES% bonds per series, issued before %TARGET_YEAR%.
echo Output File: %OUTPUT_FILE%
echo ----------------------------------------

> "%OUTPUT_FILE%" echo Valid Bonds Found (Issued before %TARGET_YEAR%)

:: This loop calls the subroutine for each series, ensuring it runs sequentially.
for %%A in (%SERIES_LIST%) do (
    call :ScanSeries %%A
)

echo.
echo 🎉 Scan complete! All series have been processed.
echo Check "%OUTPUT_FILE%" for results.
pause
goto :eof


:: ============================================================================
:: Subroutine to Scan a Single Series
:: ============================================================================
:ScanSeries
setlocal enabledelayedexpansion
set "start_serial=%~1"
set "current_serial=%start_serial%"
set /a valid_found_count=0

echo.
echo ===============================================================
echo 🎯 Starting search from series %start_serial%...
echo ===============================================================

:check_next_serial
echo Testing: !current_serial! (!valid_found_count! of %VALID_REQUIRED_PER_SERIES% found)

:: Added -UseBasicParsing below to prevent the security warning
powershell -NoProfile -Command ^
    "[Net.ServicePointManager]::SecurityProtocol = 'Tls12';" ^
    "$url = 'https://sbvv.treasurydirect.gov/validate?series_code=EE&serial_number=!current_serial!&denom_code=X';" ^
    "$issueDate = $null;" ^
    "try {" ^
    "  $r = Invoke-WebRequest -Uri $url -TimeoutSec 15 -UseBasicParsing -ErrorAction Stop;" ^
    "  if ($r.StatusCode -eq 200) {" ^
    "    $json = ConvertFrom-Json -InputObject $r.Content -ErrorAction Stop;" ^
    "    if ($json.valid -eq $true -and $json.issue_date) { $issueDate = [string]$json.issue_date }" ^
    "  }" ^
    "} catch {};" ^
    "if ($issueDate) {" ^
    "  $year = 0;" ^
    "  try { $year = [int]$issueDate.Substring(0, 4) } catch {};" ^
    "  if ($year -gt 0 -and $year -lt %TARGET_YEAR%) {" ^
    "    $msg = '✅ FOUND: !current_serial! - Issue Date: ' + $issueDate;" ^
    "    Write-Host $msg -ForegroundColor Green;" ^
    "    Out-File -InputObject $msg -Append -FilePath '%OUTPUT_FILE%' -Encoding utf8;" ^
    "    exit 1;" ^
    "  }" ^
    "}" ^
    "exit 0;"

:: Check the exit code. 1 means a valid bond matching our criteria was found.
if !ERRORLEVEL! equ 1 (
    set /a valid_found_count+=1
)

:: If we have found enough, exit this subroutine and return to the main FOR loop.
if !valid_found_count! geq %VALID_REQUIRED_PER_SERIES% (
    echo.
    echo ✨ Found %VALID_REQUIRED_PER_SERIES% matching bonds for series starting at %start_serial%. Moving on...
    endlocal
    goto :eof
)

:: Otherwise, increment the serial number and loop again.
set /a current_serial+=1
timeout /t %REQUEST_DELAY% /nobreak >nul
goto :check_next_serial
