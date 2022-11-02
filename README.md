# rabo

BeforeAll{

    . $PSScriptRoot\..\..\src\functions\Get-RSQLLogin.ps1
    . $PSScriptRoot\..\..\src\internal\functions\Write-RSQLVerbose.ps1
    . $PSScriptRoot\..\..\src\internal\functions\Get-RSQLCredential.ps1
    . $PSScriptRoot\..\..\src\internal\functions\Write-RSQLError.ps1
}

Describe "Connecting" {
    Context "Testing Login"{
        BeforeEach{
        Mock -CommandName  Connect-DbaInstance -ParameterFilter {$SqlInstance -eq 'localhost',$SqlCredential -eq 'bogus'} -MockWith {$ConnectionContext=@(@{TrueName='RB920238'})
        return [PSCustomObject]@{ConnectionContext = $ConnectionContext}}

        $password = 'test' | ConvertTo-SecureString -AsPlainText -Force
        $cred = New-Object pscredential('test', $password)
        #Mock -CommandName Get-RSQLCredential -MockWith { return $cred}
        Mock -CommandName Write-RSQLVerbose -MockWith {return $null}
        Mock -CommandName Write-RSQLError -MockWith {return $null}
        Mock -CommandName Get-WmiObject -MockWith {return [PSCustomObject]@{Domain= 'rabonetEU'}}
        Mock -CommandName Get-RSQLCredential -MockWith {return [PSCustomObject]@{  $cred = New-Object pscredential('bogus', $password)}}
        Mock Get-RSQLCredential { return $cred }

    }

        It -Name "Getting list of logins"{
          $result2 = Get-RSQLLogin -SqlInstance 'localhost'
          $Error[0].Exception.Message | Should -Be 'The network path was not found'
        }

        
        It -Name "Connecting to instance"{
            $result = Connect-DbaInstance -SqlInstance 'localhost'  
            $result| Should -Be $Result
           # $Error[0].Exception.Message | Should -Be 'The network path was not found'
        }
       



