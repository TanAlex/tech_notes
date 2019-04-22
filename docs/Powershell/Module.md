# PS Modules

## Use Set-Alias to link function in module to a script

Typically, functions are in the psm1 file but you can write in regular script files  
then use Set-Alias to link them in the psm1 module file

```Powershell
Set-Alias -Name Build-Checkpoint -Value (Join-Path $PSScriptRoot Build-Checkpoint.ps1)
Set-Alias -Name Build-Parallel -Value (Join-Path $PSScriptRoot Build-Parallel.ps1)
Set-Alias -Name Invoke-Build -Value (Join-Path $PSScriptRoot Invoke-Build.ps1)
Export-ModuleMember -Alias Build-Checkpoint, Build-Parallel, Invoke-Build
```

## Import all function scripts in a folder

This snipet is from [PSSlack.psm1](https://github.com/RamblingCookieMonster/PSSlack/blob/master/PSSlack/PSSlack.psm1)

```powershell
#Get public and private function definition files.
    $Public  = @( Get-ChildItem -Path $PSScriptRoot\Public\*.ps1 -ErrorAction SilentlyContinue )
    $Private = @( Get-ChildItem -Path $PSScriptRoot\Private\*.ps1 -ErrorAction SilentlyContinue )
    $ModuleRoot = $PSScriptRoot

#Dot source the files
Foreach($import in @($Public + $Private))
{
    Try
    {
        . $import.fullname
    }
    Catch
    {
        Write-Error -Message "Failed to import function $($import.fullname): $_"
    }
}
```

> Beginning in PowerShell 3.0, there is a new automatic variable available called $PSScriptRoot. This variable previously was only available within modules. It always points to the folder the current script is located in (so it only starts to be useful once you actually save a script before you run it).

> You can use $PSScriptRoot to load additional resources relative to your script location. For example, if you decide to place some functions in a separate "library" script that is located in the same folder, this would load the library script and import all of its functions

The following snipet handles PS v2 situation
```powershell
#handle PS2
    if(-not $PSScriptRoot)
    {
        $PSScriptRoot = Split-Path $MyInvocation.MyCommand.Path -Parent
    }
```




## Export functions
```Powershell
# $Public is an array imported above
Export-ModuleMember -Function $Public.Basename -Variable _PSSlackColorMap
```
Export using wild-cards

```Powershell
Export-ModuleMember -Function 'Get-*'
```
