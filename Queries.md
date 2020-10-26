# KustoQueries
## Sample Kusto Queries

This is a work in progress. Queries are provided as sample code, "as is" with no support

### SCOM Alert Queries
#### * Expose alerts with a specific criterion

Alert
| where AlertName containscs "CPU Utilization" and AlertSeverity == "Error"
| project TimeGenerated, AlertSeverity, SourceDisplayName, AlertName 
| sort by SourceDisplayName desc

### Service Health
//Sample Alerts from SCOM for a specific computername (dc)
Alert
| where AlertSeverity == "Error" and SourceDisplayName contains "dc"
| project TimeGenerated, AlertSeverity, SourceDisplayName, AlertName
| sort by SourceDisplayName desc


//Sample Service Health Queries
Event
| where EventLog == "System" and EventID == 7036 and Source == "Service Control Manager" and Computer contains "lms-dc" 
| parse kind=relaxed EventData with * '<Data Name="param1">' Windows_Service_Name '</Data><Data Name="param2">' Windows_Service_State '</Data>' *
| sort by TimeGenerated desc
| project Computer, Windows_Service_Name, Windows_Service_State, TimeGenerated

### Configuration Data
ConfigurationData
| where Computer contains "lms-dc-01" and SvcName =~ "spooler"
| project SvcName, SvcDisplayName, SvcState, TimeGenerated
//| where SvcState != "Running"


### Performance Queries
#### -	Top 10 processor utilization, you can rename the Y axis as needed

Perf 
| where ObjectName == "Processor"
| summarize Average_CPU = avg(CounterValue) by Computer, CounterName 
| where Average_CPU > 1
| render barchart

#### -	Similar query for disk latency

Perf 
| where CounterName == "Avg. Disk sec/Read" 
| summarize Average_Latency = avg(CounterValue) by Computer, CounterName 
| sort by Average_Latency desc


#### -	Overall Performance Data for the environment

Perf
| where TimeGenerated >=ago (7d)
| where ObjectName == "Processor"
| where CounterName == "% Processor Time"
| summarize avg(CounterValue) by bin(TimeGenerated, 1h)
| render timechart

#### - Show me the top 10 computers with less than 1GB RAM available?
 
 Perf
| where CounterName == "Available MBytes" and  CounterValue <= 1024
| project TimeGenerated, Computer, CounterName, ObjectName, CounterValue 
| summarize arg_max(TimeGenerated, *) by Computer
| top 10 by CounterValue asc
 
You could drill down once you find the one using most of the resources and expose data quickly

#### -	Perf data for all our the computers we have

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

#### -	Compare perf data between computer groups

Perf
| where ObjectName == "Processor"
| where CounterName == "% Processor Time"
| extend group = case(Computer startswith "apm01", "Application Performance Monitoring", Computer startswith "dc", "dc", "Domain Controllers")
| summarize avg(CounterValue) by bin(TimeGenerated, 1h), group
| render timechart

 
You can see perf data per groups, however, I need to work on this one, I can see both groups, but need to work on the case statement

#### -	show me the top 5 logical drives by free space? (optional filter by drive)
let PercentSpace = 90;
Perf
| where ObjectName == "LogicalDisk" and CounterName == "% Free Space"
| summarize FreeSpace = min(CounterValue) by Computer, InstanceName
//| where InstanceName == "C:"
| where InstanceName contains ":"
| where FreeSpace < PercentSpace
| sort by FreeSpace asc
| take 5

### Event Queries

#### -	List of all security events
SecurityEvent
| project  Activity
| parse Activity with activityID " - " activityDesc
| summarize count() by activityID

#### -	When was the last time a specific computer was rebooted?

Event 
| where Computer containscs "clt" and  EventID == 6005 and EventLog == "System" and Source == "EventLog"
| project Computer, TimeGenerated 
| sort by Computer

### Updates Queries

#### -	Updates for a specific computer

UpdateSummary
| project Computer, WindowsUpdateSetting  
 | where Computer  like 'test' 
 | render table

#### -	Updates for a computer and WSUS Server 

UpdateSummary
| project Computer, ManagementGroupName , WindowsUpdateAgentVersion, WSUSServer  | render table
| sort by ManagementGroupName asc 

#### -	Is there a specific update installed?

Update 
| where KBID == 3173424
| project Computer, KBID, UpdateState 
| render table

#### -	Show me updates from different types, and tell me which SCOM environment they are talking to 

ConfigurationChange 
| where ConfigChangeType == "Software" and SoftwareType == "Update" 
| project Computer, ConfigChangeType , SoftwareType, TimeGenerated , SoftwareName, ManagementGroupName
| sort by ManagementGroupName desc


 #### - Show me Updates from a specific Computer
 
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

 #### - What are the top 100 updates for the last 30 days on my machines

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



### Security

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


### SLA for IIS Logs

W3CIISLog
| where TimeGenerated >= ago(2d) 
| summarize SLABreached=count(TimeTaken >10000), SLAMet=count(TimeTaken < 10000), totalCount=count() by bin(TimeGenerated, 15m)  
| extend pctSLAIndex = SLAMet * 100.0/totalCount 
| extend SLA = 99
| project TimeGenerated, pctSLAIndex , SLA 
| render timechart 


### Azure AD

#### Show me login info and location

SigninLogs
| extend LocationAndState= strcat(tostring(LocationDetails["state"]), ", ",  (LocationDetails["countryOrRegion"])) 
| project TimeGenerated ,UserDisplayName , ConditionalAccessStatus ,  SourceSystem , OperationName, LocationAndState, IPAddress

#### Which IPs are accesing my storage accounts? (Thanks Dustin Paulson!)

AzureActivity
| where Type == "AzureActivity" 
| where OperationName == "List Storage Account Keys" 
| where ResourceProvider == "Microsoft.Storage" 
| extend method_ = tostring(parse_json(HTTPRequest).method) 
| project Caller, CallerIpAddress, method_, Resource, ResourceGroup 


### Windows Virtual Desktop
#### Connection status, review CorrelationID, State

WVDConnections
| where TimeGenerated > ago(1d)
| where UserName contains "Smith"
| sort by TimeGenerated, CorrelationId asc 

#### No Connections by day for UPN / this will show the pattern of connections

WVDConnections
| where TimeGenerated >ago(1d)
| where UserName contains "Smith"
| sort by TimeGenerated asc, CorrelationId
| summarize dcount(CorrelationId) by bin (TimeGenerated, 1d)

#### Session lenght for User UPN / Show Connection Lenght for Application, that could show an issue

let Events = WVDConnections | where UserName contains "Smith";
Events
| where State == "Connected"
| where TimeGenerated > ago (1d)
| project CorrelationId, UserName, ResourceAlias, StartTime=TimeGenerated
| join (Events
| where State== "Completed"
| where TimeGenerated > ago(1d)
| project EndTime=TimeGenerated, CorrelationId)
on CorrelationId
| project Duration=EndTime-StartTime,ResourceAlias

#### Show Errors / we see Time Stamp, CorrelationId (to help debug issues), ActivityType (to debug issue on specific activity types), Message, ServiceError can help expose that Microsoft support must be engaged (i.e. service outage!)

WVDErrors
| where TimeGenerated > ago(1d)
| where UserName contains "Smith"
| take 100

#### Show Errors / Show aggregated data of different errors and render a chart, or review by Results as well by checking teh ServiceError Column!

WVDErrors
| where TimeGenerated > ago(1d)
| where UserName =="SMITHB@vincyman.com"
| summarize CorrelationIDCount = count(CorrelationId) by CodeSymbolic,ServiceError
| sort by CorrelationIDCount desc 
| render columnchart 

#### Search by ErrorCode / You can see UserName column to see if multiple users are hitting a specific issue specified in CodeSymbolic

WVDErrors
| where TimeGenerated > ago(1d)
| where CodeSymbolic contains "failed" 
| summarize count(UserName) by CodeSymbolic, ServiceError

#### Search by ErrorCode / you can see Results, to expose and identify  if multiple users are hitting a specific issue specified in CodeSymbolic

WVDErrors
| where TimeGenerated > ago(1d)
| where ServiceError == "false"
| summarize usercount = count(UserName) by CodeSymbolic
| sort by usercount desc 
| render barchart 

#### What did our admins activities were done recently?

WVDManagement
| where TimeGenerated > ago(1d)
| where ObjectsUpdated != 0
| project TimeGenerated, UserName, _ResourceId

#### Track Resource Usage of applications! 

WVDConnections
| where TimeGenerated > ago(1d)
| summarize dcount(UserName) by ResourceAlias
| render columnchart 



#### Sample SCOM

//////////////////////////////////////////////////////////////////
//Sample Alerts from SCOM for a specific computername (dc)
Alert
| where AlertSeverity == "Error" and SourceDisplayName contains "dc"
| project TimeGenerated, AlertSeverity, SourceDisplayName, AlertName
| sort by SourceDisplayName desc

//Sample Service Health Queries
Event
| where EventLog == "System" and EventID == 7036 and Source == "Service Control Manager" and Computer contains "lms-dc"
| parse kind=relaxed EventData with * '<Data Name="param1">' Windows_Service_Name '</Data><Data Name="param2">' Windows_Service_State '</Data>' *
| sort by TimeGenerated desc
| project Computer, Windows_Service_Name, Windows_Service_State, TimeGenerated

ConfigurationData
| where Computer contains "lms-dc-01" and SvcName =~ "spooler"
| project SvcName, SvcDisplayName, SvcState, TimeGenerated
//| where SvcState != "Running"

// SQL Server Recommendations
SQLAssessmentRecommendation
| where TimeGenerated >=ago (30d)
| where RecommendationResult == "Failed"
//and DatabaseName == "OperationsManager"
| project  TimeGenerated, Recommendation, DatabaseName
| sort by DatabaseName desc

// Update Settings for a specific computer
UpdateSummary
| project Computer, WindowsUpdateSetting
| where Computer like 'dc'
| render table

// WSUS Settings
UpdateSummary
| project Computer, ManagementGroupName , WindowsUpdateAgentVersion, WSUSServer
| render table
| sort by ManagementGroupName asc

// Show me updates from different types, and tell me which SCOM environment they are talking to
ConfigurationChange
| where ConfigChangeType == "Software" and SoftwareType == "Update"
| project Computer, ConfigChangeType , SoftwareType, TimeGenerated , SoftwareName, ManagementGroupName
| sort by ManagementGroupName desc

//Show me Updates needed from a specific Computer
Update
| where  SourceComputerId in ((Heartbeat
| where  notempty(Computer)
| summarize arg_max(TimeGenerated, Solutions) by SourceComputerId
| where Solutions has "updates"
| distinct SourceComputerId))
| summarize hint.strategy=partitioned arg_max(TimeGenerated, *) by Computer, SourceComputerId, Product, ProductArch
| where UpdateState=~"Needed" and Computer contains "dc-01"
| render table
| summarize count() by Classification

//Graph Performance usage
Perf
| where TimeGenerated > ago(7d)
| where Computer contains "dc"
| where CounterName == @"% Processor Time"
| summarize avg(CounterValue) by Computer, bin(TimeGenerated, 15m)
| render timechart

//security account log on
SecurityEvent
| where TimeGenerated >= ago(1d)
| where Process != ""
| where Process != "-"
| project TimeGenerated, Process , Computer, Account
| summarize count() by TimeGenerated, Process, Computer, Account

//Security incorrect log on
SecurityEvent
| where EventID between (529 .. 537) or EventID==539 or (EventID==4625 and Status=="0xc000006d") and TimeGenerated >= ago(1m)
| project TargetAccount, IpAddress, Computer, LogonProcessName, AuthenticationPackageName, LogonTypeName

//Security event log cleared
SecurityEvent
| where EventID==517 or EventID==1102
| project Activity, Computer, TimeGenerated, EventData

//Security Account Management: Passwords Change Attempts by Non-owner
SecurityEvent
| where EventID==4723 or EventID==4724 or EventID between (627 .. 628) and SubjectAccount != "ANONYMOUS LOGON" and TargetAccount!=SubjectAccount
| project TimeGenerated, Computer, TargetAccount, ChangedBy=SubjectAccount

//SigninLogs AAD
SigninLogs
| extend LocationAndState= strcat(tostring(LocationDetails["state"]), ", ", (LocationDetails["countryOrRegion"]))
| project TimeGenerated ,UserDisplayName , ConditionalAccessStatus , SourceSystem , OperationName, LocationAndState, IPAddress

// ALA Query
// Shows cost per data source in ALA
// == Variables start ==
let CostPerGB = 2.18;
let Currency = '$';
let StartTime = ago(9d);
let EndTime = ago(1hr);
// == Variables end ==
let ChargeableTables = Usage
| where TimeGenerated between (startofday(StartTime) .. endofday(EndTime))
| where IsBillable == true
//| where DataType != 'SecurityEvent'
| summarize by tostring(pack(DataType, bin(TimeGenerated, 1d)));
union withsource=SourceTable *
| where TimeGenerated between (startofday(StartTime) .. endofday(EndTime))
| summarize TotalDailyUsagePerTable=sum(_BilledSize) by SourceTable, TimeGenerated=(bin(TimeGenerated, 1d))
| where tostring(pack(SourceTable, bin(TimeGenerated, 1d))) in (ChargeableTables)
| summarize DailyUsageMbNonSecurity=(sum(TotalDailyUsagePerTable)/1024)/1024, DataSources=make_set(SourceTable) by Date=format_datetime(TimeGenerated, 'dd-MM-yyy')
| extend TotalDailyPriceNonSecurity = strcat(Currency, round((DailyUsageMbNonSecurity/1024) * CostPerGB, 2))



//security account log on
SecurityEvent
| where TimeGenerated >= ago(1d)
| where Process != ""
| where Process != "-"
| project TimeGenerated, Process , Computer, Account
| summarize count() by TimeGenerated, Process, Computer, Account

//Security incorrect log on
SecurityEvent
| where EventID between (529 .. 537) or EventID==539 or (EventID==4625 and Status=="0xc000006d") and TimeGenerated >= ago(1m)
| project TargetAccount, IpAddress, Computer, LogonProcessName, AuthenticationPackageName, LogonTypeName

//Security event log cleared
SecurityEvent
| where EventID==517 or EventID==1102
| project Activity, Computer, TimeGenerated, EventData

//Security Account Management: Passwords Change Attempts by Non-owner
SecurityEvent
| where EventID==4723 or EventID==4724 or EventID between (627 .. 628) and SubjectAccount != "ANONYMOUS LOGON" and TargetAccount!=SubjectAccount
| project TimeGenerated, Computer, TargetAccount, ChangedBy=SubjectAccount



#### Key Vault Queries


### Show me Operations and Source
AzureDiagnostics | where ResourceProvider =="MICROSOFT.KEYVAULT" | summarize count() by OperationName, CallerIPAddress, Resource | order by count_ desc

### Details on Categories for a specific Key Vault and renaming the Any(Category) column, into Category
AzureDiagnostics | where ResourceProvider =="MICROSOFT.KEYVAULT" and Resource == "CH-MIGRATEKV-J5ZR1T" | summarize Category=any(Category) by OperationName


### Details on failures reported in metrics

AzureDiagnostics | where ResourceProvider =="MICROSOFT.KEYVAULT" and httpStatusCode_d > 200 | summarize count() by Resource, CallerIPAddress, httpStatusCode_d | order by httpStatusCode_d desc


