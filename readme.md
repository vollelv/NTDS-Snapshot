# NTDS SNAPSHOT

Vi legger inn shadowcopy av NTDS databasen.

## SETT STØRRELSE FOR MAX BRUK TIL SHADOWCOPY

Kjør i cmd.exe/powershell:
```
vssadmin resize shadowstorage /on=c: /for=c: /maxsize=20%
```

Hvis kommandoen over feiler, kjør denne istedenfor:

```
vssadmin add shadowstorage /on=c: /for=c: /maxsize=20%
```

Lagre først scriptet under her som filnavn "Start-NTDSSnapshot.ps1" i mappen C:\Powershell (c:\Powershell\Start-NTDSSnapshot.ps1).

## LEGGE INN SCHEDULED TASK

Deretter kjører vi scriptet under for å legge inn Scheduled Task "NTDS Snapshot" med scriptet vi lagret over:
```
function Get-RandomTimeBetween {
    <#
    .EXAMPLE
    Get-RandomTimeBetween -StartTime "08:30" -EndTime "16:30"
    #>
    [Cmdletbinding()]
    param(
        [parameter(Mandatory=$True)][string]$StartTime,
        [parameter(Mandatory=$True)][string]$EndTime
        )
    begin{
        $minuteTimeArray =@("00","01","02","03","04","05","06","07","08","09","10","11","12","13","14","15","16","17","18","19","20","21","22","23","24","25","26","27","28","29","30","31","32","33","34","35","36","37","38","39","40","41","42","43","44","45","46","47","48","49","50","51","52","53","54","55","56","57","58","59")
    }   
    process{
        $rangeHours = @($StartTime.Split(":")[0],$EndTime.Split(":")[0])
        $hourTime = Get-Random -Minimum $rangeHours[0] -Maximum $rangeHours[1]
        $minuteTime = "00"
        if($hourTime -ne $rangeHours[0] -and $hourTime -ne $rangeHours[1]){
            $minuteTime = Get-Random $minuteTimeArray
            return "${hourTime}:${minuteTime}"
        }
        elseif ($hourTime -eq $rangeHours[0]) { # hour is the same as the start time so we ensure the minute time is higher
            $minuteTime = $minuteTimeArray | ?{ [int]$_ -ge [int]$StartTime.Split(":")[1] } | Get-Random # Pick the next quarter
            #If there is no quarter available (eg 09:50) we jump to the next hour (10:00)
            return (.{If(-not $minuteTime){ "${[int]hourTime+1}:00" }else{ "${hourTime}:${minuteTime}" }})              
            
        }
        else { # hour is the same as the end time
            #By sorting the array, 00 will be pick if no close hour quarter is found
            $minuteTime = $minuteTimeArray | Sort-Object -Descending | ?{ [int]$_ -le [int]$EndTime.Split(":")[1] } | Get-Random
            return "${hourTime}:${minuteTime}"
        }
    }
}
$TaskTime = Get-RandomTimeBetween -StartTime "00:00" -EndTime "23:59" #Setter tilfeldig tidspunkt for scheduled task

# Scheduled Task
$Action = New-ScheduledTaskAction -execute "powershell.exe" -argument "C:\Powershell\Start-NTDSSnapshot.ps1"
$taskSettings = New-ScheduledTaskSettingsSet -Compatibility Win8 -ExecutionTimeLimit 00:15:00
$trigger = New-ScheduledTaskTrigger -At $TaskTime -Once -RepetitionInterval (New-TimeSpan -Days 7) #-RepetitionDuration ([System.TimeSpan]::MaxValue)
$principal = New-ScheduledTaskPrincipal -UserID SYSTEM -LogonType Interactive -RunLevel Highest
Register-ScheduledTask "NTDS Snapshot" –Action $action –Trigger $trigger –Principal $principal -Settings $taskSettings
```