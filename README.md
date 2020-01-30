![MIKES DATA WORK GIT REPO](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_01.png "Mikes Data Work")        


# Use SQL To Find When SSPI Context Error Started
**Post Date: November 22, 2017**        



## Contents    
- [About Process](##About-Process)  
- [SQL Logic](#SQL-Logic)  
- [Build Info](#Build-Info)  
- [Author](#Author)  
- [License](#License)       

## About-Process


![Find SSPI Error Time]( https://mikesdatawork.files.wordpress.com/2017/11/image002.png "Find SSPI Error Time Of Occurance")
 
<p>As most of us know already; getting the SSPI Context error is extremely annoying and it's infuriating that this is still happening on the latest servers.
In all most cases this is caused by the (Service Principal Name) issue. Unfortunately; while this is found based on 'connectivity' to a database server; the issue it's self is only resolved using non database centric operations such as managing Service Principal Names/DNS entries on the network security layer. Once this is corrected you'll need to restart the database service so the new SPN can be properly assigned.
To get started on finding out when this issue occurred I present to you the following SQL logic. As the image above shows; this will give you all the logs so you can review it and I've included a quick WHERE clause to help you get started in looking for the right bits of information. In this case you'll get results in blocks. On this example you'll see a red-block, and black-block. Each block represents when the database service was restarted. Typically when resolving this issue; you'll need to restart the services so often you'll see even one or more set of different blocks denoted by the time when the restarting service takes place.
With the logic I can quickly tell when the error started. I know it started after the previous reboot; where the services were restarted twice, and the last time was about 61 minutes after the Server came online.</p>      




## SQL-Logic
```SQL
use master;
set nocount on
 
-- check for kerberos config
select
    [net_transport]
,   [auth_scheme]
from
    sys.dm_exec_connections
where
    [session_id] = @@spid
 
/*********************************/
-- get boot info.
declare @start_os   datetime = (select [sqlserver_start_time] from sys.dm_os_sys_info)
declare @start_sql  datetime = (select [create_date] from sys.databases where [database_id] = 2)
 
select
    'start_os'      = left(@start_os, 19)
,   'start_sql'     = left(@start_sql, 19)
,   'notes'         = 'The SQL Service started aproximately (' + cast(datediff(minute, @start_os, @start_sql) as varchar) + ' minutes) after the OS boot.'
 
/*********************************/
-- create table to list all sql files
if object_id('tempdb..#errorlogfiles') is not null
    drop table #errorlogfiles
create table #errorlogfiles ([archive_num] int, [date] datetime, [text] nvarchar(max))
insert into #errorlogfiles exec xp_enumerrorlogs 
 
-- create table to hold errors events (from files above)
 
if object_id('tempdb..#errorlogs') is not null
    drop table #errorlogs
create table #errorlogs ([archive_num] int null, [logdate] datetime, [processinfo] varchar(255), [text] varchar(max))
 
declare @i      int = 0
declare @maxid  int = (select max([archive_num]) from #errorlogfiles)
 
while @i <= @maxid
begin
    insert into #errorlogs ([logdate], [processinfo], [text])
    exec master..sp_readerrorlog @i, 1;
    update #errorlogs set [archive_num] = @i where [archive_num] is null;
    set @i = @i + 1;
end
 
-- get error log info regarding spn events, and service start events.
select
    'logdate'       = left([logdate], 19)
,   [processinfo]
,   [text]
from
    #errorlogs
where
    [text] like '%started%'
    or [text] like '%spn%'
order by
    [logdate] desc

```

[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

[![Gist](https://img.shields.io/badge/Gist-MikesDataWork-<COLOR>.svg)](https://gist.github.com/mikesdatawork)
[![Twitter](https://img.shields.io/badge/Twitter-MikesDataWork-<COLOR>.svg)](https://twitter.com/mikesdatawork)
[![Wordpress](https://img.shields.io/badge/Wordpress-MikesDataWork-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

     
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Mikes Data Work](https://raw.githubusercontent.com/mikesdatawork/images/master/git_mikes_data_work_banner_02.png "Mikes Data Work")

