# Monitor Azure VM (without running VM Insight)

To keep costs down, some customers only want the basic monitoring without running VMInishgt
With Azure Monitor Agent DCR, we can now customize collection to only specific "performance counters" and "events" to reduce data size and save money
Below are the basic monitors on which the rules must be built.


//Heartbeat: 

Heartbeat
| summarize max(TimeGenerated) by Computer
| where max_TimeGenerated < ago(5m)
// Dimension by Computer


//CPU: for Windows & Linux 
Perf 
| where ObjectName == "Processor" and CounterName == "% Processor Time" or 
ObjectName == "Processor Information" and CounterName == "% Processor Time"
| summarize AggregatedValue = avg(CounterValue) by bin(TimeGenerated, 5m), Computer 
| where AggregatedValue > 90


//Memory: Avail MB for Windows & Linux 
Perf
| where ObjectName == "Memory" 
| where CounterName == "Available MBytes Memory" or CounterName == "Available Bytes"
| summarize GBFree=avg(CounterValue) by Computer,bin(TimeGenerated, 5m)
| summarize arg_max(TimeGenerated, *) by Computer
| where GBFree < 3000

// % Mem Usage – Windows Only 
Perf
//| where Computer startswith "DC02" 
// add other computers here
| where ObjectName == "Memory" 
| where CounterName == "Available MBytes Memory" or CounterName == "Available Bytes"
| summarize GBFree=avg(CounterValue) by Computer,bin(TimeGenerated, 5m)
| summarize arg_max(TimeGenerated, *) by Computer
|join kind= inner
(
    Perf
    | where ObjectName == "Memory" 
    | where CounterName == "% Committed Bytes In Use"
    | summarize PctFree=avg(CounterValue) by Computer,bin(TimeGenerated, 5m)
    | summarize arg_max(TimeGenerated, *) by Computer
)
on Computer 
| project   TotalSizeGB=round(GBFree*100/PctFree,0), 
            round(PctFree,2),
            round(GBFree,2), 
            Computer
| summarize FreePCT=avg(PctFree) by Computer,
            TotalSizeGB,
            FreeGB = round(GBFree / 1024,2)
| where FreePCT < 10

//Free space Logical Disk –
// add DCR '\LogicalDisk(*)\% Free Space' to collect all disks free space

let _minValue = 90; // Set the minValue according to your needs
Perf
| where ObjectName == "LogicalDisk" and CounterName == "% Free Space"  and InstanceName != "_Total" and InstanceName != "HarddiskVolume1" 
  or ObjectName == "Logical Disk" and CounterName == "% Used Space"
| where TimeGenerated >= ago(30m) // choose time to observe 
| where CounterValue > _minValue
| summarize avg(CounterValue) by bin(TimeGenerated, 10m), Computer, InstanceName

//Process not running – 
//Add collection of Process(*)% Processor Time and run the query on process disappear

//Services status: Add System event collection or specific event System!*[System[EventID=7036]] for monitor //Service stopped
Event
| where EventLog == "System" and EventID == 7036 and Source == "Service Control Manager"
| parse kind=relaxed EventData with * "<Data Name="param1">" Windows_Service_Name "</Data><Data Name="param2">" Windows_Service_State "</Data>" *
| sort by TimeGenerated desc
| project Computer, Windows_Service_Name, Windows_Service_State, TimeGenerated

//Log file:
Custom monitor by AMA custom log wizard

//VM Shut down:
AzureActivity
| where TimeGenerated > ago(10m)
| where OperationName == "Deallocate Virtual Machine" and ActivityStatus == "Succeeded"

//Syslog: Find Linux kernel events
Syslog
| where ProcessName == "kernel" and SyslogMessage contains "Killed process"

To Deploy AMA or by AMA DCR configuration or by Az Policy:
Policy > Definition > Configure Windows machines to run Azure Monitor Agent and associate them to a Data Collection Rule, and add the DCR Resource Id
 

![image](https://user-images.githubusercontent.com/24368496/188887658-05d61d9b-98ff-4ee7-8d5a-6943badcd49c.png)
