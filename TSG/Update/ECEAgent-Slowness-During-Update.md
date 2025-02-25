# ECEAgent Causing Slowness during Update

## Symptoms

The slowness can cause the update to take significantly longer than expected and possibly timeout. The exact symptom of this issue is that the interface 'FailoverECEService' takes a long time to complete. If this interface takes longer than 15 minutes to complete the stamp is likely experiencing this issue. 

### Issue Confirmation
1) Examine the progress xml for the update. 
2) The interface FailoverECEService has the status Inprogress or Pending
3) The time since the update was resumed **or** the start time of the interface is more than 15 minutes ago

## Mitigation Details  

Running the below command from one of the cluster nodes while the agent is stuck should clear up the slowness and the update should auto remediate its state. The script forces a restart of the ECEAgent on all of the nodes in the cluster

``` Powershell 
 $scriptBlock = {
            try {
                Trace-Execution "Running ECEAgent service restart on $(hostname)"
                $eceagent = Get-WmiObject -Class Win32_Service -Filter "Name='ECEAgent'"
                Stop-Process $eceagent.ProcessId -Force
                Start-Sleep -Seconds 5
                $eceagent = Get-WmiObject -Class Win32_Service -Filter "Name='ECEAgent'"
                
                $rety = 0
                $filepath = $eceagent.PathName.Replace("C:", $("\\" + $(hostname) + "\c$")).Replace('"',"")
                while ($eceagent.State -ne "Running")
                {
                    Start-Process -FilePath $filepath
                    $eceagent = Get-WmiObject -Class Win32_Service -Filter "Name='ECEAgent'"
                    if(++$retry -ge 5)
                    {
                        throw "Failed to start ECEAgent during agent restart on $(hostname)"
                    }
                    Start-Sleep -Seconds 1
                }
                Trace-Execution "ECE Agent service was restarted on $(hostname)"
            }
            catch {
                Trace-Execution "ECE Agent service does not exist. Unable to restart on $(hostname)"
            }
        }
    $nodes = Get-Clusternode | select -expandproperty Name

    
    
    Invoke-Command -ComputerName $nodes -ScriptBlock $scriptBlock -Verbose -ErrorAction 'Stop'
```