# Powershell functions notes
<!-- TOC -->

- [Powershell functions notes](#powershell-functions-notes)
    - [ParameterSet and CmdletBinding](#parameterset-and-cmdletbinding)
    - [Use Invoke-RestMethod to get GitHubOAuth Token](#use-invoke-restmethod-to-get-githuboauth-token)

<!-- /TOC -->

## ParameterSet and CmdletBinding

**Notable points:**

-  DefaultParameterSetName
-  [Parameter(Mandatory = $false, Position=0,  ParameterSetName='repo')]
-  [ValidatePattern('^\*$|^none$|^.+$')]
-  [ValidateSet('open', 'closed')]

```Powershell
function Get-GitHubIssues
{
  [CmdletBinding(DefaultParameterSetName='repo')]
  param(
    [Parameter(Mandatory = $false, Position=0,  ParameterSetName='repo')]
    [string]
    $Owner = $null,

    [Parameter(Mandatory = $false, Position=1, ParameterSetName='repo')]
    [string]
    $Repository = $null,

    [Parameter(Mandatory = $false, ParameterSetName='user')]
    [switch]
    $ForUser,

    [Parameter(Mandatory = $false)]
    [ValidateSet('open', 'closed')]
    $State = 'open',

    [Parameter(Mandatory = $false, ParameterSetName='user')]
    [ValidateSet('assigned', 'created', 'mentioned', 'subscribed')]
    $Filter = 'assigned',

    [Parameter(Mandatory = $false, ParameterSetName='repo')]
    [ValidatePattern('^\*$|^none$|^\d+$')]
    $Milestone,

    [Parameter(Mandatory = $false, ParameterSetName='repo')]
    [ValidatePattern('^\*$|^none$|^.+$')]
    $Assignee,

    [Parameter(Mandatory = $false, ParameterSetName='repo')]
    [string]
    $Creator,

    [Parameter(Mandatory = $false, ParameterSetName='repo')]
    [string]
    $Mentioned,

    [Parameter(Mandatory = $false)]
    [string[]]
    $Labels = @(),

    [Parameter(Mandatory = $false)]
    [ValidateSet('created', 'updated', 'comments')]
    $Sort = 'created',

    [Parameter(Mandatory = $false)]
    [ValidateSet('asc', 'desc')]
    $Direction = 'desc',

    [Parameter(Mandatory = $false)]
    [DateTime]
    #Optional string of a timestamp in ISO 8601 format: YYYY-MM-DDTHH:MM:SSZ
    $Since
  )
```

## Use Invoke-RestMethod to get GitHubOAuth Token

it's using basic auth, use `GetBytes` to convert string to bytes
then use `ToBase64String` to convert them to base64 string

```Powershell
function Get-GitHubOAuthTokens
{
  [CmdletBinding()]
  param(
    [Parameter(Mandatory = $true)]
    [string]
    $UserName,

    [Parameter(Mandatory = $true)]
    [string]
    $Password
  )

  try
  {
    $params = @{
      Uri = 'https://api.github.com/authorizations';
      Headers = @{
        Authorization = 'Basic ' + [Convert]::ToBase64String(
          [Text.Encoding]::ASCII.GetBytes("$($userName):$($password)"));
      }
    }
    $global:GITHUB_API_OUTPUT = Invoke-RestMethod @params
    #Write-Verbose $global:GITHUB_API_OUTPUT

    $global:GITHUB_API_OUTPUT |
      % {
        $date = [DateTime]::Parse($_.created_at).ToString('g')
        Write-Host "`n$($_.app.name) - Created $date"
        Write-Host "`t$($_.token)`n`t$($_.app.url)"
      }
  }
  catch
  {
    Write-Error "An unexpected error occurred (bad user/password?) $($Error[0])"
  }
}
```

