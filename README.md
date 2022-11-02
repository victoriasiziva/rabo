# rabo

<#
    .SYNOPSIS
        Gets SQL instance logins

    .DESCRIPTION
        Connects to SQL instance and retrieves all logins except system objects and NT* logins.
        Logins can be scripted out if GenerateScript parameter is set.
        The script also tests the logins by checking AD

    .NOTES
        Author:     Bruno Rocha
        Date:       2020-02-13
        Backlog:


   .PARAMETER SqlInstance
         SQL instance name in the format server\instance

   .PARAMETER Login
        Login names to be checked

   .PARAMETER GenerateScript
        Scripts out logins including permissions

   .EXAMPLE
        Get-RSQLLogin -SqlInstance WBEH0197\DEINSTANCE00 -Login 'Domain\LoginName' -GenerateScript
        Gets login 'Domain\LoginName' and generates the script

    #>
function Get-RSQLLogin {

    [cmdletbinding(SupportsShouldProcess)]
    param(
        [parameter(Mandatory)] [string] $SqlInstance,
        [parameter()] [string[]] $Login,
        [parameter()] [switch] $GenerateScript,
        [parameter()] [PSCredential] $Credential
    )

    #$ErrorActionPreference = 'Stop'
    Begin {

        if (-NOT $Credential) {
            try {
                Write-RSQLVerbose -Message "Retrieving credentials" -RequestID $RequestID
                $ThisDomain = (Get-WmiObject Win32_ComputerSystem).Domain
                $Credential = Get-RSQLCredential -AccountType Automation -Domain $ThisDomain
            }
            catch {
                Write-RSQLError -Message $_.exception.message -RequestID $RequestID -Milestone 999
                break
            }
        }

        if ($SqlInstance.IndexOf('\') -gt 0) {
            $ServerInstance = $SqlInstance.split('\')
            $ServerName = $ServerInstance[0]
            $InstanceName = $ServerInstance[1]
        }
        else {
            Write-Error -Message 'SqlInstance parameter must be Server\InstanceName'
            break
        }
    }
    Process {
        try {
            Write-Verbose -Message "Resolving network name for $ServerName"
            if (-NOT ($ServerName = (Resolve-DnsName -Name $ServerName -Type A -ErrorAction SilentlyContinue -WarningAction SilentlyContinue).name)) {
                Write-Error -Message "Network name for $ServerName could not be resolved"
                return
            }

            $SqlInstanceName = $ServerName + '\' + $InstanceName

            Write-Verbose -Message "Attempting to connect to $SqlInstanceName"
            $SqlServer = Connect-DbaInstance -SqlInstance $SqlInstanceName -SqlCredential $Credential -ErrorAction SilentlyContinue -ErrorVariable ErrorConnection
            if (-not $SqlServer.DomainInstanceName) {
                Write-Error -Message "Could not connect to $SqlInstanceName. $ErrorConnection"
                return
            }

            $Logins = $SqlServer.Logins | Where-Object { $_.IsSystemObject -eq $false -and $_.Name -notlike 'NT*'}

            if ($Login) {
                $Logins = $Logins | Where-Object { $_.Name -in $Login }
            }

            $Logins | ForEach-Object {
                $TestResult = $null
                if($_.LoginType -in ('WindowsUser', 'WindowsGroup')){
                    $TestResult = Test-DbaWindowsLogin -SqlInstance $SqlInstanceName -Login $_.Name -SqlCredential $Credential -EnableException
                }

                if ($GenerateScript) {
                    $Script = Export-DbaLogin -SqlInstance $SqlInstanceName -Login $_.Name -SqlCredential $Credential -ExcludeDatabases -ExcludeJobs -BatchSeparator '' -WarningAction SilentlyContinue -Passthru
                }
                Add-Member -Force -InputObject $_ -MemberType NoteProperty -Name Script -value $Script
                Add-Member -Force -InputObject $_ -MemberType NoteProperty -Name FoundInAD -value $TestResult.Found
                Add-Member -Force -InputObject $_ -MemberType NoteProperty -Name LockedOut -value $TestResult.LockedOut
                Add-Member -Force -InputObject $_ -MemberType NoteProperty -Name Domain -value $TestResult.Domain

                Select-Object -InputObject $_ -Property Name, LoginType, CreateDate, LastLogin, HasAccess, IsLocked, IsDisabled, FoundInAD, LockedOut, Domain, Script
            }
        }
        catch {
            Write-Error -Message $_.exception.message
            return
        }
    }
    End { }
}
