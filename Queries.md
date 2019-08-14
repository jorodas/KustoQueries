# KustoQueries
Sample Kusto Queries

SCOM Alert Queries
-	Expose alerts with a specific criterion

Alert 
| where AlertName containscs "CPU Utilization" and AlertSeverity == "Error"
| project TimeGenerated, AlertSeverity, SourceDisplayName, AlertName 
| sort by SourceDisplayName desc

Performance Queries
-	Top 10 processor utilization, you can rename the Y axis as needed

Perf 
| where ObjectName == "Processor"
| summarize Average_CPU = avg(CounterValue) by Computer, CounterName 
| where Average_CPU > 1
| render barchart

-	Similar query for disk latency

Perf 
| where CounterName == "Avg. Disk sec/Read" 
| summarize Average_Latency = avg(CounterValue) by Computer, CounterName 
| sort by Average_Latency desc


-	Overall Performance Data for the environment

Perf
| where TimeGenerated >=ago (7d)
| where ObjectName == "Processor"
| where CounterName == "% Processor Time"
| summarize avg(CounterValue) by bin(TimeGenerated, 1h)
| render timechart

 
You could drill down once you find the one using most of the resources and expose data quickly

-	Perf data for all our the computers we have

let endTime=now();
let timerange =1d;
let startTime=now() - timerange;
let mInterval=4;
let mAvgParm= repeat(1, mInterval);
Perf
| where ObjectName == "Processor"
| where CounterName == "% Processor Time"
| make-series avgCpu=avg(CounterValue)  default=0 on TimeGenerated in range(startTime, endTime, 15m) by Computer
| extend moving_avgCpu = series_fir(avgCpu, mAvgParm) 
| render timechart

 
You could highlight the one you see using most data

-	Compare perf data between computer groups

Perf
| where ObjectName == "Processor"
| where CounterName == "% Processor Time"
| extend group = case(Computer startswith "apm01", "Application Performance Monitoring", Computer startswith "dc", "dc", "Domain Controllers")
| summarize avg(CounterValue) by bin(TimeGenerated, 1h), group
| render timechart


 
You can see perf data per groups, however, I need to work on this one, I can see both groups, but need to work on the case statement

-	show me the top 5 logical drives by free space? (optional filter by drive)
let PercentSpace = 90;
Perf
| where ObjectName == "LogicalDisk" and CounterName == "% Free Space"
| summarize FreeSpace = min(CounterValue) by Computer, InstanceName
//| where InstanceName == "C:"
| where InstanceName contains ":"
| where FreeSpace < PercentSpace
| sort by FreeSpace asc
| take 5

Event Queries
-	List of all security events
SecurityEvent
| project  Activity
| parse Activity with activityID " - " activityDesc
| summarize count() by activityID

-	When was the last time a specific computer was rebooted?

Event 
| where Computer containscs "clt" and  EventID == 6005 and EventLog == "System" and Source == "EventLog"
| project Computer, TimeGenerated 
| sort by Computer


Updates Queries
-	Updates for a specific computer

UpdateSummary
| project Computer, WindowsUpdateSetting  
 | where Computer  like 'test' 
 | render table

-	Updates for a computer and WSUS Server 

UpdateSummary
| project Computer, ManagementGroupName , WindowsUpdateAgentVersion, WSUSServer  | render table
| sort by ManagementGroupName asc 


-	Is there a specific update installed?

Update 
| where KBID == 3173424
| project Computer, KBID, UpdateState 
| render table

-	Show me updates from different types, and tell me which SCOM environment they are talking to 

ConfigurationChange 
| where ConfigChangeType == "Software" and SoftwareType == "Update" 
| project Computer, ConfigChangeType , SoftwareType, TimeGenerated , SoftwareName, ManagementGroupName
| sort by ManagementGroupName desc


 - Show me Updates from a specific Computer
 
 Update
| where TimeGenerated>ago(5h) and OSType=="Linux" and SourceComputerId in ((Heartbeat
| where TimeGenerated>ago(12h) and OSType=="Linux" and notempty(Computer)
| summarize arg_max(TimeGenerated, Solutions) by SourceComputerId
| where Solutions has "updates"
| distinct SourceComputerId))
| summarize hint.strategy=partitioned arg_max(TimeGenerated, *) by Computer, SourceComputerId, Product, ProductArch
| where UpdateState=~"Needed" and Computer=="centos-02"
| render table
| summarize count() by Classification

 - What are the top 100 updates for the last 30 days on my machines

Update
| where TimeGenerated>ago(30d) and OSType!="Linux" and (Optional==false or Classification has "Critical" or Classification has "Security") and SourceComputerId in ((Heartbeat
| where TimeGenerated>ago(30d) and OSType=~"Windows" and notempty(Computer)
| summarize arg_max(TimeGenerated, Solutions) by SourceComputerId
| where Solutions has "updates"
| distinct SourceComputerId))
| summarize hint.strategy=partitioned arg_max(TimeGenerated, *) by Computer, SourceComputerId, UpdateID
| where UpdateState=~"Needed" and Approved!=false
| project Computer  , TimeGenerated  , PublishedDate  , KBID  , Product  , Title  , UpdateState  , Optional  , RebootBehavior
| take 100
| sort by TimeGenerated desc



Security

SecurityEvent
| where TimeGenerated >= ago(1d) 
| where Process != "" 
| where Process != "-" 
| where Process !contains "\\windows\\system" 
| where Process !contains "\\Program Files\\Microsoft" 
| where Process !contains "\\Program Files\\Microsoft Monitoring Agent" 
| where Process !contains "\\ProgramData" 
| project TimeGenerated, Process , Computer, Account 
| summarize count() by TimeGenerated, Process, Computer, Account 


SLA

W3CIISLog
| where TimeGenerated >= ago(2d) 
| summarize SLABreached=count(TimeTaken >10000), SLAMet=count(TimeTaken < 10000), totalCount=count() by bin(TimeGenerated, 15m)  
| extend pctSLAIndex = SLAMet * 100.0/totalCount 
| extend SLA = 99
| project TimeGenerated, pctSLAIndex , SLA 
| render timechart 


Azure AD

show me login info and location

SigninLogs
| extend LocationAndState= strcat(tostring(LocationDetails["state"]), ", ",  (LocationDetails["countryOrRegion"])) 
| project TimeGenerated ,UserDisplayName , ConditionalAccessStatus ,  SourceSystem , OperationName, LocationAndState, IPAddress


