# Table of Contents

- [Status of DBCC commands running in the background](#status-of-dbcc-commands-running-in-the-background)
- [Most expensive queries](#most-expensive-queries)
- [Jobs owned by a specific user](#jobs-owned-by-a-specific-user)
- [Table storage](#table-storage)
- [Delete duplicate rows from a table](#delete-duplicate-rows-from-a-table)
- [Buffer Pool](#buffer-pool)
- [Unable to shrink transaction log file in SQL Server](#unable-to-shrink-transaction-log-file-in-sql-server)
- [Delete user](#delete-user)
- [Get drives and free space percent](#get-drives-and-free-space-percent)
- [Get backup info where a backup is missing](#get-backup-info-where-a-backup-is-missing)
- [Partition information](#partition-information)
- [Get fragmented indexes](#get-fragmented-indexes)
- [Objects referencing a table](#objects-referencing-a-table)
- [Get information about log usage](#get-information-about-log-usage)
- [Information about your virtual logs inside your transaction log](#information-about-your-virtual-logs-inside-your-transaction-log)
- [Run SSMS as different user](#run-ssms-as-different-user)
- [Shrink tempdb without restarting the server](#shrink-tempdb-without-restarting-the-server)
- [Stored procedure to delete records using cursor](#stored-procedure-to-delete-records-using-cursor)
- [Restore database](#restore-database)
- [Get all rights of users](#get-all-rights-of-users)
- [Granting right to run jobs](#granting-right-to-run-jobs)
- [User permissions](#user-permissions)
- [Getting list of linked servers](#getting-list-of-linked-servers)
- [Resume suspended data movement on the specified secondary database](#resume-suspended-data-movement-on-the-specified-secondary-database)
- [Getting jobs with daily and weekly schedule](#getting-jobs-with-daily-and-weekly-schedule)
- [Generating script to disable all jobs](#generating-script-to-disable-all-jobs)
- [Scripting Out the Logins Server Role Assignments and Server Permissions](#scripting-out-the-logins-server-role-assignments-and-server-permissions)
- [Scripting users and right on database level](#scripting-users-and-right-on-database-level)
- [SQL Server Indexes on Computed Columns](#sql-server-indexes-on-computed-columns)
- [Kill all sessions of the user and drop the login](#kill-all-sessions-of-the-user-and-drop-the-login)
- [Searching all triggers on a server for a specific string](#searching-all-triggers-on-a-server-for-a-specific-string)
- [Exporting and importing SSIS packages](#exporting-and-importing-SSIS-packages)
- [Get missing indexes from DMV](#get-missing-indexes-from-dmv)
- [Get unused indexes from DMV](#get-unused-indexes-from-dmv)
- [Copy login with same password and SID](#copy-login-with-same-password-and-sid)
- [Set database compatibility level](#set-database-compatibility-level)
- [Tracking the progress of the CREATE INDEX command](#tracking-the-progress-of-the-create-index-command)
- [Reason for not backup up the log](#reason-for-not-backup-up-the-log)
- [Trace to discover locking](#trace-to-discover-locking)
- [Making database to read only – changing database to read_write](#making-database-to-read-only–changing-database-to-read_write)
- [Find outdated statistics](#find-outdated-statistics)
- [Get empty space in each database file](#get-empty-space-in-each-database-file)
- [Memory consumption](#memory-consumption)

# Status of DBCC commands running in the background
```SQL
select 
  [status],
  start_time,
  convert(varchar,(total_elapsed_time/(1000))/60) + 'M ' + convert(varchar,(total_elapsed_time/(1000))%60) + 'S' AS [Elapsed],
  convert(varchar,(estimated_completion_time/(1000))/60) + 'M ' + convert(varchar,(estimated_completion_time/(1000))%60) + 'S' as [ETA],
  command,
  -- [sql_handle],
  -- connection_id,
  -- database_id,
  d.name database_name,
  blocking_session_id,
  percent_complete,
  CONVERT(VARCHAR(1000),
		(SELECT 
			SUBSTRING(text,r.statement_start_offset/2,
			CASE WHEN r.statement_end_offset = -1 THEN 1000 ELSE (r.statement_end_offset-r.statement_start_offset)/2 END)
			FROM sys.dm_exec_sql_text([sql_handle]))
		) AS [SQL]
from  sys.dm_exec_requests r
inner join sys.databases d on r.database_id = d.database_id
where estimated_completion_time > 1
order by total_elapsed_time desc
```

# Most expensive queries
```SQL
SELECT TOP 10 SUBSTRING(qt.TEXT, (qs.statement_start_offset/2)+1,
  ((CASE qs.statement_end_offset
  WHEN -1 THEN DATALENGTH(qt.TEXT)
  ELSE qs.statement_end_offset
  END - qs.statement_start_offset)/2)+1),
  qs.execution_count,
  qs.total_logical_reads, qs.last_logical_reads,
  qs.total_logical_writes, qs.last_logical_writes,
  qs.total_worker_time,
  qs.last_worker_time,
  qs.total_elapsed_time/1000000 total_elapsed_time_in_S,
  qs.last_elapsed_time/1000000 last_elapsed_time_in_S,
  qs.last_execution_time,
  qp.query_plan
FROM sys.dm_exec_query_stats qs
CROSS APPLY sys.dm_exec_sql_text(qs.sql_handle) qt
CROSS APPLY sys.dm_exec_query_plan(qs.plan_handle) qp
ORDER BY qs.total_logical_reads DESC -- logical reads
-- ORDER BY qs.total_logical_writes DESC -- logical writes
-- ORDER BY qs.total_worker_time DESC -- CPU time
```

# Jobs owned by a specific user

```SQL
SELECT 
  J.name AS Job_Name
  ,L.name AS Job_Owner
FROM msdb.dbo.sysjobs_view J
INNER JOIN master.dbo.syslogins L
ON J.owner_sid = L.sid
WHERE L.name = '<user name>';
```

# Table storage
```SQL
USE <database>
GO
SELECT
  s.Name AS SchemaName,
  t.Name AS TableName,
  p.rows AS RowCounts,
  CAST(ROUND((SUM(a.used_pages) / 128.00), 2) AS NUMERIC(36, 2)) AS Used_MB,
  CAST(ROUND((SUM(a.total_pages) - SUM(a.used_pages)) / 128.00, 2) AS NUMERIC(36, 2)) AS Unused_MB,
  CAST(ROUND((SUM(a.total_pages) / 128.00), 2) AS NUMERIC(36, 2)) AS Total_MB
FROM sys.tables t
INNER JOIN sys.indexes i ON t.OBJECT_ID = i.object_id
INNER JOIN sys.partitions p ON i.object_id = p.OBJECT_ID AND i.index_id = p.index_id
INNER JOIN sys.allocation_units a ON p.partition_id = a.container_id
INNER JOIN sys.schemas s ON t.schema_id = s.schema_id
GROUP BY t.Name, s.Name, p.Rows
ORDER BY Used_MB desc
GO
```

# Delete duplicate rows from a table

```SQL
WITH cte AS (
    SELECT 
      column1
      ,column2,
      ,column3
      , ROW_NUMBER() OVER (
            PARTITION BY 
      column1
      ,column2,
      ,column3
            ORDER BY 
      column1
      ,column2,
      ,column3
        ) row_num
     FROM 
        <table>
)
DELETE FROM cte
WHERE row_num > 1;
```


# Buffer Pool


## find out how big buffer pool is and determine percentage used by each database

```SQL
DECLARE @total_buffer INT;

SELECT @total_buffer = cntr_value   FROM sys.dm_os_performance_counters
WHERE RTRIM([object_name]) LIKE '%Buffer Manager'   AND counter_name = 'Total Pages';

;WITH src AS(   
	SELECT        database_id, db_buffer_pages = COUNT_BIG(*)
	FROM sys.dm_os_buffer_descriptors  
	GROUP BY database_id)
SELECT  
	[db_name] = CASE [database_id] WHEN 32767 THEN 'Resource DB' ELSE DB_NAME([database_id]) END,   
	db_buffer_pages,   
	db_buffer_MB = db_buffer_pages / 128,   
	db_buffer_percent = CONVERT(DECIMAL(6,3),        
	db_buffer_pages * 100.0 / @total_buffer)
FROM src
ORDER BY db_buffer_MB DESC;
```


## then drill down into memory used by objects in database of your choice
```SQL
USE <database>;
WITH src AS(   
	SELECT       
		[Object] = o.name,       
		[Type] = o.type_desc,       
		[Index] = COALESCE(i.name, ''),       
		[Index_Type] = i.type_desc,       
		p.[object_id],       
		p.index_id,       
		au.allocation_unit_id
FROM  sys.partitions AS p   
	INNER JOIN  sys.allocation_units AS au ON p.hobt_id = au.container_id   
	INNER JOIN  sys.objects AS o ON p.[object_id] = o.[object_id]   
	INNER JOIN  sys.indexes AS i ON o.[object_id] = i.[object_id] AND p.index_id = i.index_id   
WHERE  au.[type] IN (1,2,3)       
	AND o.is_ms_shipped = 0
)
SELECT   
	src.[Object],   
	src.[Type],   
	src.[Index],   
	src.Index_Type,   
	buffer_pages = COUNT_BIG(b.page_id),   
	buffer_mb = COUNT_BIG(b.page_id) / 128
FROM   src
INNER JOIN   sys.dm_os_buffer_descriptors AS b
	ON src.allocation_unit_id = b.allocation_unit_id
WHERE   b.database_id = DB_ID()
GROUP BY   src.[Object],   src.[Type],   src.[Index],   src.Index_Type
ORDER BY   buffer_pages DESC;
```

# Unable to shrink transaction log file in SQL Server


1) Switch to Simple Recovery Model
2) Run a CHECKPOINT and DBCC DROPCLEANBUFFERS (just in case)
3) Shrink the log file
4) Switch back to Full Recovery Model
5) Grow the log in reasonable increments (I use 4000MB increments)
6) Set autogrowth to a value your I/O subsystem can handle
7) Run a full or differential backup to restart the log backup chain.

# Delete user 

## Drop the schema's. They WILL NOT get dropped if schema had objects
```SQL
EXEC sp_MSForEachDB 'USE [?];
        IF  EXISTS (SELECT * FROM sys.schemas WHERE name = N'<USERNAME>')
        DROP SCHEMA [<USERNAME>]; '
```        

## Drop the database users
```SQL

EXEC sp_MSForEachDB 'USE [?];
        IF  EXISTS (SELECT * FROM sys.database_principals WHERE name = N'<USERNAME>')
        DROP USER [<USERNAME>]; '
``` 
## Drop the login
```SQL
DROP LOGIN [<USERNAME>];
```

# Get drives and free space percent
```SQL
SELECT
   [Drive] = volume_mount_point
  ,[PercentFree] = CONVERT(INT,CONVERT(DECIMAL(15,2),available_bytes) / total_bytes * 100)
  FROM sys.master_files mf
  CROSS APPLY sys.dm_os_volume_stats(mf.database_id,mf.file_id)
  --Optional where clause filters drives with more than 20% free space
WHERE CONVERT(INT,CONVERT(DECIMAL(15,2),available_bytes) / total_bytes * 100) < 20
GROUP BY
  volume_mount_point
  ,total_bytes/1024/1024 --/1024
  ,available_bytes/1024/1024 --/1024
  ,CONVERT(INT,CONVERT(DECIMAL(15,2),available_bytes) / total_bytes * 100)
ORDER BY [Drive]
```

# Get backup info where a backup is missing

```SQL
SELECT  name ,
recovery_model_desc AS RecoveryModel,
d AS 'LastFullBackup' ,
i AS 'LastDiffBackup' ,
l AS 'LastLogBackup'
FROM (
            SELECT  name ,
	            recovery_model_desc AS RecoveryModel,
	            state_desc ,
	            d AS 'LastFullBackup' ,
	            i AS 'LastDiffBackup' ,
	            l AS 'LastLogBackup'
            FROM    ( SELECT    db.name ,
			            db.state_desc ,
			            db.recovery_model_desc ,
			            type ,
			            backup_finish_date
	               FROM      master.sys.databases db
  	               LEFT OUTER JOIN msdb.dbo.backupset a ON a.database_name = db.name
			 WHERE db.name <> 'tempdb'
			 ) AS Sourcetable 
	     PIVOT ( MAX(backup_finish_date) FOR type IN ( D, I, L ) ) AS MostRecentBackup
            ) d
WHERE LastFullBackup IS NULL
      OR (DATEDIFF(DAY, LastFullBackup, GETDATE()) > 7)
      OR (DATEDIFF(DAY, LastFullBackup, GETDATE()) > 1 AND ((LastDiffBackup IS NULL) OR (DATEDIFF(DAY, LastDiffBackup, GETDATE()) > 1)))
      OR ((RecoveryModel = 'FULL') AND ((LastLogBackup IS NULL) OR (DATEDIFF(MINUTE, LastLogBackup, GETDATE()) > 30)))
```

# Partition information

```SQL
select
  t.name as TableName
  , ps.name as PartitionScheme
  , pf.name as PartitionFunction
  , p.partition_number
  , p.rows
  , case
    when pf.boundary_value_on_right=1 then 'RIGHT'
    else 'LEFT'
    end [range_type]
  , prv.value [boundary]
from sys.tables t
    join sys.indexes i on t.object_id = i.object_id
    join sys.partition_schemes ps on i.data_space_id = ps.data_space_id
    join sys.partition_functions pf on ps.function_id = pf.function_id
    join sys.partitions p on i.object_id = p.object_id and i.index_id = p.index_id
    join sys.partition_range_values prv on pf.function_id = prv.function_id and p.partition_number = prv.boundary_id
where i.index_id < 2  --So we're only looking at a clustered index or heap, which the table is partitioned on
order by p.partition_number
```

# Get fragmented indexes

```SQL
select TableName, 
	   IndexName, 
	   avg_fragmentation_in_percent,
	   case when  avg_fragmentation_in_percent <= 30 
		then 'ALTER INDEX [' + IndexName + '] ON [' + TableName + '] REORGANIZE;' 
		else 'ALTER INDEX [' + IndexName + '] ON [' + TableName + '] REBUILD WITH (ONLINE = ON);' 
		end as operation
INTO fragmented
from (
	SELECT a.object_id, object_name(a.object_id) AS TableName,
		a.index_id, name AS IndexName, 
		avg_fragmentation_in_percent
	FROM sys.dm_db_index_physical_stats
		(DB_ID (N'<database>')
			, NULL
			, NULL
			, NULL
			, NULL) AS a
	INNER JOIN sys.indexes AS b
		ON a.object_id = b.object_id
		AND a.index_id = b.index_id
) a
where avg_fragmentation_in_percent > 5
AND IndexName is not null
order by avg_fragmentation_in_percent desc;

GO
```

# Objects referencing a table

```SQL
SELECT
  referencing_schema_name = SCHEMA_NAME(o.SCHEMA_ID),
  referencing_object_name = o.name,
  referencing_object_type_desc = o.type_desc,
  referenced_schema_name,
  referenced_object_name = referenced_entity_name,
  referenced_object_type_desc = o1.type_desc,
  referenced_server_name, referenced_database_name
  --,sed.* -- Uncomment for all the columns
FROM
  sys.sql_expression_dependencies sed
INNER JOIN
  sys.objects o ON sed.referencing_id = o.[object_id]
LEFT OUTER JOIN
  sys.objects o1 ON sed.referenced_id = o1.[object_id]
WHERE
  referenced_entity_name = '<table>'
ORDER BY referencing_object_name
```

# Get information about log usage

Why each database’s log file isn’t clearing out

```SQL
SELECT log_reuse_wait_desc
FROM sys.databases
WHERE name = '<database>'
```

https://www.brentozar.com/archive/2016/03/my-favorite-system-column-log_reuse_wait_desc/

Possible reasons include:
*	Log backup needs to be run (or if you could lose a day’s worth of data, throw this little fella in simple recovery mode)
*	Active backup running – because the full backup needs the transaction log to be able to restore to a specific point in time
*	Active transaction – somebody typed BEGIN TRAN and locked their workstation for the weekend
*	Database mirroring, replication, or AlwaysOn Availability Groups – because these features need to hang onto transaction log data to send to another replica



# Information about your virtual logs inside your transaction log
```SQL
USE <database>;
DBCC loginfo;
```

## How much of the transaction log is being used
```SQL
DBCC SQLPerf(logspace)
```

# Run SSMS as different user
```
runas /user:<username> "C:\Program Files (x86)\Microsoft SQL Server Management Studio 18\Common7\IDE\Ssms.exe"
```

# Shrink tempdb without restarting the server

```SQL
CHECKPOINT;
GO
DBCC DROPCLEANBUFFERS;
GO

DBCC FREEPROCCACHE;
GO

DBCC FREESYSTEMCACHE ('ALL');
GO

DBCC FREESESSIONCACHE;
GO


USE [tempdb]
GO
DBCC SHRINKFILE (N'<tempdb file name>' , 1000)
GO
```

# Stored procedure to delete records using cursor

```SQL
USE <database>
GO


CREATE Procedure [dbo].[PROC_Delete....]
As

BEGIN

	DECLARE @Id INT

	select Id, ....
	into #temp
	from <table>
	where ....
	)

	DECLARE DeleteCrsr CURSOR FOR
		select Id
		from #temp
		where ....


	OPEN DeleteCrsr

	FETCH NEXT FROM DeleteCrsr
	INTO @Id

	WHILE @@FETCH_STATUS=0
	BEGIN

		delete from <table> where Id = @Id

		IF(@@ROWCOUNT=0)
		BEGIN
			PRINT 'Failed to delete ' + CAST(@Id AS varchar)
		END

		waitfor delay '00:00:02'

		FETCH NEXT FROM DeleteCrsr
		INTO @DId

	END

	CLOSE DeleteCrsr

	DEALLOCATE DeleteCrsr

END

GO
```

# Restore database

```SQL
USE [master]
GO

alter database <database> set offline with rollback immediate
GO

RESTORE DATABASE <database> FROM  DISK = N'<backup file>, REPLACE, RECOVERY
GO
```

# Get all rights of users
```SQL
SELECT  
    [UserName] = ulogin.[name],
    [UserType] = CASE princ.[type]
                    WHEN 'S' THEN 'SQL User'
                    WHEN 'U' THEN 'Windows User'
                    WHEN 'G' THEN 'Windows Group'
                 END,  
    [DatabaseUserName] = princ.[name],       
    [Role] = null,      
    [PermissionType] = perm.[permission_name],       
    [PermissionState] = perm.[state_desc],       
    [ObjectType] = CASE perm.[class] 
                        WHEN 1 THEN obj.type_desc               -- Schema-contained objects
                        ELSE perm.[class_desc]                  -- Higher-level objects
                   END,       
    [ObjectName] = CASE perm.[class] 
                        WHEN 1 THEN OBJECT_NAME(perm.major_id)  -- General objects
                        WHEN 3 THEN schem.[name]                -- Schemas
                        WHEN 4 THEN imp.[name]                  -- Impersonations
                   END,
    [ColumnName] = col.[name]
FROM    
    --database user
    sys.database_principals princ  
LEFT JOIN
    --Login accounts
    sys.server_principals ulogin on princ.[sid] = ulogin.[sid]
LEFT JOIN        
    --Permissions
    sys.database_permissions perm ON perm.[grantee_principal_id] = princ.[principal_id]
LEFT JOIN
    --Table columns
    sys.columns col ON col.[object_id] = perm.major_id 
                    AND col.[column_id] = perm.[minor_id]
LEFT JOIN
    sys.objects obj ON perm.[major_id] = obj.[object_id]
LEFT JOIN
    sys.schemas schem ON schem.[schema_id] = perm.[major_id]
LEFT JOIN
    sys.database_principals imp ON imp.[principal_id] = perm.[major_id]
WHERE 
    princ.[type] IN ('S','U','G') AND
    -- No need for these system accounts
    princ.[name] NOT IN ('sys', 'INFORMATION_SCHEMA')
```

# Granting right to run jobs

```SQL
USE [msdb] GO ALTER ROLE [SQLAgentOperatorRole] ADD MEMBER [user1] GO
```

# User permissions

```SQL
use <database>
go

select  princ.name
,       princ.type_desc
,       perm.permission_name
,       perm.state_desc
,       perm.class_desc
,       object_name(perm.major_id)
from    sys.database_principals princ
left join
        sys.database_permissions perm
on      perm.grantee_principal_id = princ.principal_id
where princ.name = '<username>'
```

# Getting list of linked servers

```SQL
-- drop table tempdb..#t

create table #t(
SRV_NAME sysname NULL,
SRV_PROVIDERNAME nvarchar(128) NULL,
SRV_PRODUCT nvarchar(128) NULL,
SRV_DATASOURCE nvarchar(4000) NULL,
SRV_PROVIDERSTRING nvarchar(4000) NULL,
SRV_LOCATION nvarchar(4000) NULL,
SRV_CAT sysname NULL
)

insert into #t
exec sp_linkedservers  

select srv_name, srv_datasource from #t where srv_name
```

# Resume suspended data movement on the specified secondary database

```SQL
ALTER DATABASE <database> SET HADR RESUME;
```

# Getting jobs with daily and weekly schedule

```SQL
select
   sysjobs.name job_name
  ,sysjobs.enabled job_enabled
  ,sysschedules.name schedule_name
  ,sysschedules.freq_recurrence_factor
  ,case
   when freq_type = 4 then 'Daily'
  end frequency
  ,
  'every ' + cast (freq_interval as varchar(3)) + ' day(s)'  Days
  ,
  case
   when freq_subday_type = 2 then ' every ' + cast(freq_subday_interval as varchar(7)) 
   + ' seconds' + ' starting at '
   + stuff(stuff(RIGHT(replicate('0', 6) +  cast(active_start_time as varchar(6)), 6), 3, 0, ':'), 6, 0, ':')
   when freq_subday_type = 4 then ' every ' + cast(freq_subday_interval as varchar(7)) 
   + ' minutes' + ' starting at '
   + stuff(stuff(RIGHT(replicate('0', 6) +  cast(active_start_time as varchar(6)), 6), 3, 0, ':'), 6, 0, ':')
   when freq_subday_type = 8 then ' every ' + cast(freq_subday_interval as varchar(7)) 
   + ' hours'   + ' starting at '
   + stuff(stuff(RIGHT(replicate('0', 6) +  cast(active_start_time as varchar(6)), 6), 3, 0, ':'), 6, 0, ':')
   else ' starting at ' 
   +stuff(stuff(RIGHT(replicate('0', 6) +  cast(active_start_time as varchar(6)), 6), 3, 0, ':'), 6, 0, ':')
  end time
from msdb.dbo.sysjobs
inner join msdb.dbo.sysjobschedules on sysjobs.job_id = sysjobschedules.job_id
inner join msdb.dbo.sysschedules on sysjobschedules.schedule_id = sysschedules.schedule_id
where freq_type = 4

union

-- jobs with a weekly schedule
select
   sysjobs.name job_name
  ,sysjobs.enabled job_enabled
  ,sysschedules.name schedule_name
  ,sysschedules.freq_recurrence_factor
  ,case
   when freq_type = 8 then 'Weekly'
  end frequency
  ,
  replace
  (
     CASE WHEN freq_interval&1 = 1 THEN 'Sunday, ' ELSE '' END
    +CASE WHEN freq_interval&2 = 2 THEN 'Monday, ' ELSE '' END
    +CASE WHEN freq_interval&4 = 4 THEN 'Tuesday, ' ELSE '' END
    +CASE WHEN freq_interval&8 = 8 THEN 'Wednesday, ' ELSE '' END
    +CASE WHEN freq_interval&16 = 16 THEN 'Thursday, ' ELSE '' END
    +CASE WHEN freq_interval&32 = 32 THEN 'Friday, ' ELSE '' END
    +CASE WHEN freq_interval&64 = 64 THEN 'Saturday, ' ELSE '' END
    ,', '
    ,''
  ) Days
  ,
  case
   when freq_subday_type = 2 then ' every ' + cast(freq_subday_interval as varchar(7)) 
   + ' seconds' + ' starting at '
   + stuff(stuff(RIGHT(replicate('0', 6) +  cast(active_start_time as varchar(6)), 6), 3, 0, ':'), 6, 0, ':') 
   when freq_subday_type = 4 then ' every ' + cast(freq_subday_interval as varchar(7)) 
   + ' minutes' + ' starting at '
   + stuff(stuff(RIGHT(replicate('0', 6) +  cast(active_start_time as varchar(6)), 6), 3, 0, ':'), 6, 0, ':')
   when freq_subday_type = 8 then ' every ' + cast(freq_subday_interval as varchar(7)) 
   + ' hours'   + ' starting at '
   + stuff(stuff(RIGHT(replicate('0', 6) +  cast(active_start_time as varchar(6)), 6), 3, 0, ':'), 6, 0, ':')
   else ' starting at ' 
   + stuff(stuff(RIGHT(replicate('0', 6) +  cast(active_start_time as varchar(6)), 6), 3, 0, ':'), 6, 0, ':')
  end time
from msdb.dbo.sysjobs
inner join msdb.dbo.sysjobschedules on sysjobs.job_id = sysjobschedules.job_id
inner join msdb.dbo.sysschedules on sysjobschedules.schedule_id = sysschedules.schedule_id
where freq_type = 8
order by job_enabled desc
```


# Generating script to disable all jobs

select 'exec msdb..sp_update_job @job_name = ''' + name + ''', @enabled = 0;'  from msdb.dbo.sysjobs

# Scripting Out the Logins Server Role Assignments and Server Permissions
```SQL
--https://www.datavail.com/blog/scripting-out-the-logins-server-role-assignments-and-server-permissions/
/********************************************************************************************************************/
-- Scripting Out the Logins, Server Role Assignments, and Server Permissions
/********************************************************************************************************************/
SET NOCOUNT ON
-- Scripting Out the Logins To Be Created
SELECT 'IF (SUSER_ID('+QUOTENAME(SP.name,'''')+') IS NULL) BEGIN CREATE LOGIN ' +QUOTENAME(SP.name)+
			   CASE 
					WHEN SP.type_desc = 'SQL_LOGIN' THEN ' WITH PASSWORD = ' +CONVERT(NVARCHAR(MAX),SL.password_hash,1)+ ' HASHED, CHECK_EXPIRATION = ' 
						+ CASE WHEN SL.is_expiration_checked = 1 THEN 'ON' ELSE 'OFF' END +', CHECK_POLICY = ' +CASE WHEN SL.is_policy_checked = 1 THEN 'ON,' ELSE 'OFF,' END
					ELSE ' FROM WINDOWS WITH'
				END 
	   +' DEFAULT_DATABASE=[' +SP.default_database_name+ '], DEFAULT_LANGUAGE=[' +SP.default_language_name+ '] END;' COLLATE SQL_Latin1_General_CP1_CI_AS AS [-- Logins To Be Created --]
FROM sys.server_principals AS SP LEFT JOIN sys.sql_logins AS SL
		ON SP.principal_id = SL.principal_id
WHERE SP.type IN ('S','G','U')
		AND SP.name NOT LIKE '##%##'
		AND SP.name NOT LIKE 'NT AUTHORITY%'
		AND SP.name NOT LIKE 'NT SERVICE%'
		AND SP.name <> ('sa');

-- Scripting Out the Role Membership to Be Added
SELECT 
'EXEC master..sp_addsrvrolemember @loginame = N''' + SL.name + ''', @rolename = N''' + SR.name + '''
' AS [-- Server Roles the Logins Need to be Added --]
FROM master.sys.server_role_members SRM
	JOIN master.sys.server_principals SR ON SR.principal_id = SRM.role_principal_id
	JOIN master.sys.server_principals SL ON SL.principal_id = SRM.member_principal_id
WHERE SL.type IN ('S','G','U')
		AND SL.name NOT LIKE '##%##'
		AND SL.name NOT LIKE 'NT AUTHORITY%'
		AND SL.name NOT LIKE 'NT SERVICE%'
		AND SL.name <> ('sa');


-- Scripting out the Permissions to Be Granted
SELECT 
	CASE WHEN SrvPerm.state_desc <> 'GRANT_WITH_GRANT_OPTION' 
		THEN SrvPerm.state_desc 
		ELSE 'GRANT' 
	END
    + ' ' + SrvPerm.permission_name + ' TO [' + SP.name + ']' + 
	CASE WHEN SrvPerm.state_desc <> 'GRANT_WITH_GRANT_OPTION' 
		THEN '' 
		ELSE ' WITH GRANT OPTION' 
	END collate database_default AS [-- Server Level Permissions to Be Granted --] 
FROM sys.server_permissions AS SrvPerm 
	JOIN sys.server_principals AS SP ON SrvPerm.grantee_principal_id = SP.principal_id 
WHERE   SP.type IN ( 'S', 'U', 'G' ) 
		AND SP.name NOT LIKE '##%##'
		AND SP.name NOT LIKE 'NT AUTHORITY%'
		AND SP.name NOT LIKE 'NT SERVICE%'
		AND SP.name <> ('sa');

SET NOCOUNT OFF
```


# Scripting users and right on database level

```SQL
USE Master  -- Use the required database name here
GO
SET NOCOUNT ON;

PRINT 'USE ['+DB_NAME()+']';
PRINT 'GO'

/********************************************************************************/
/**************** Create a new user and map it with login ***********************/
/********************************************************************************/

PRINT '/*************************************************************/'
PRINT '/************** Create User Script ***************************/'
PRINT '/*************************************************************/'

SELECT 'CREATE USER [' + NAME + '] FOR LOGIN [' + NAME + ']' 
FROM sys.database_principals
WHERE	[Type] IN ('U','S')
		AND 
		[NAME] NOT IN ('dbo','guest','sys','INFORMATION_SCHEMA')

GO
-- Troubleshooting User creation issues
PRINT '/***'+CHAR(10)+
'--Error 15023: User or role <XXXX> is already exists in the database.'+CHAR(10)+
'--Then Execute the below code can fix the issue'+CHAR(10)+
'EXEC sp_change_users_login ''Auto_Fix'',''<Failed User>'''+CHAR(10)+
'GO **/'

/************************************************************************/
/************  Script the User Role Information *************************/
/************************************************************************/

PRINT '/**********************************************************/'
PRINT '/************** Create User-Role Script *******************/'
PRINT '/**********************************************************/'

SELECT 'EXEC sp_AddRoleMember ''' + DBRole.NAME + ''', ''' + DBP.NAME + '''' 
FROM sys.database_principals DBP
INNER JOIN sys.database_role_members DBM ON DBM.member_principal_id = DBP.principal_id
INNER JOIN sys.database_principals DBRole ON DBRole.principal_id = DBM.role_principal_id
WHERE DBP.NAME <> 'dbo'

GO

/***************************************************************************/
/************  Script Database Level Permission ****************************/
/***************************************************************************/

PRINT '/*************************************************************/'
PRINT '/************** Database Level Permission ********************/'
PRINT '/*************************************************************/'

SELECT	CASE WHEN DBP.state <> 'W' THEN DBP.state_desc ELSE 'GRANT' END
		+ SPACE(1) + DBP.permission_name + SPACE(1)
		+ SPACE(1) + 'TO' + SPACE(1) + QUOTENAME(USR.name) COLLATE database_default
		+ CASE WHEN DBP.state <> 'W' THEN SPACE(0) ELSE SPACE(1) + 'WITH GRANT OPTION' END + ';' 
FROM	sys.database_permissions AS DBP
		INNER JOIN	sys.database_principals AS USR	ON DBP.grantee_principal_id = USR.principal_id
WHERE	DBP.major_id = 0 and USR.name <> 'dbo'
ORDER BY DBP.permission_name ASC, DBP.state_desc ASC


/***************************************************************************/
/************  Script Object Level Permission ******************************/
/***************************************************************************/

PRINT '/*************************************************************/'
PRINT '/************** Object Level Permission **********************/'
PRINT '/*************************************************************/'

SELECT	CASE WHEN DBP.state <> 'W' THEN DBP.state_desc ELSE 'GRANT' END
		+ SPACE(1) + DBP.permission_name + SPACE(1) + 'ON ' + QUOTENAME(USER_NAME(OBJ.schema_id)) + '.' + QUOTENAME(OBJ.name) 
		+ CASE WHEN CL.column_id IS NULL THEN SPACE(0) ELSE '(' + QUOTENAME(CL.name) + ')' END
		+ SPACE(1) + 'TO' + SPACE(1) + QUOTENAME(USR.name) COLLATE database_default
		+ CASE WHEN DBP.state <> 'W' THEN SPACE(0) ELSE SPACE(1) + 'WITH GRANT OPTION' END + ';' 
FROM	sys.database_permissions AS DBP
		INNER JOIN	sys.objects AS OBJ	ON DBP.major_id = OBJ.[object_id]
		INNER JOIN	sys.database_principals AS USR	ON DBP.grantee_principal_id = USR.principal_id
		LEFT JOIN	sys.columns AS CL	ON CL.column_id = DBP.minor_id AND CL.[object_id] = DBP.major_id
ORDER BY DBP.permission_name ASC, DBP.state_desc ASC



SET NOCOUNT OFF;
```


# SQL Server Indexes on Computed Columns

Adding computed column to the table
```SQL
ALTER TABLE sales.customers
ADD 
    email_local_part AS 
        SUBSTRING(email, 
            0, 
            CHARINDEX('@', email, 0)
        );
```

Creating index on the computed column

```SQL
CREATE INDEX ix_cust_email_local_part
ON sales.customers(email_local_part);
```

# Kill all sessions of the user and drop the login

```SQL
DECLARE @loginNameToDrop sysname
SET @loginNameToDrop = '<username>';

DECLARE sessionsToKill CURSOR FAST_FORWARD FOR
    SELECT session_id
    FROM sys.dm_exec_sessions
    WHERE login_name = @loginNameToDrop
OPEN sessionsToKill

DECLARE @sessionId INT
DECLARE @statement NVARCHAR(200)

FETCH NEXT FROM sessionsToKill INTO @sessionId

WHILE @@FETCH_STATUS = 0
BEGIN
    PRINT 'Killing session ' + CAST(@sessionId AS NVARCHAR(20)) + ' for login ' + @loginNameToDrop

    SET @statement = 'KILL ' + CAST(@sessionId AS NVARCHAR(20))
    EXEC sp_executesql @statement

    FETCH NEXT FROM sessionsToKill INTO @sessionId
END

CLOSE sessionsToKill
DEALLOCATE sessionsToKill

PRINT 'Dropping login ' + @loginNameToDrop
SET @statement = 'DROP LOGIN [' + @loginNameToDrop + ']'
EXEC sp_executesql @statement
```


# Searching all triggers on a server for a specific string
```SQL
DECLARE @command varchar(1000) 
SELECT @command = 'USE [?] 
SELECT     
    DB_NAME() AS DataBaseName,  
    S.Name AS SchemaName,               
    T.name AS TableName,
    dbo.SysObjects.Name AS TriggerName,
    dbo.sysComments.Text AS SqlContent
FROM dbo.SysObjects 
INNER JOIN dbo.sysComments ON dbo.SysObjects.ID = dbo.sysComments.ID
INNER JOIN sys.tables AS T ON sysobjects.parent_obj = t.object_id 
INNER JOIN sys.schemas AS S ON t.schema_id = s.schema_id 
WHERE dbo.SysObjects.xType = ''TR''
and dbo.sysComments.Text like ''%sp_send_dbmail%'''

EXEC sp_MSforeachdb @command 
```

# Exporting and importing SSIS packages

```SQL
;
WITH FOLDERS AS
(
    -- Capture root node
    SELECT
        cast(PF.foldername AS varchar(max)) AS FolderPath
    ,   PF.folderid
    ,   PF.parentfolderid
    ,   PF.foldername
    FROM
        msdb.dbo.sysssispackagefolders PF
    WHERE
        PF.parentfolderid IS NULL

    -- build recursive hierarchy
    UNION ALL
    SELECT
        cast(F.FolderPath + '\' + PF.foldername AS varchar(max)) AS FolderPath
    ,   PF.folderid
    ,   PF.parentfolderid
    ,   PF.foldername
    FROM
        msdb.dbo.sysssispackagefolders PF
        INNER JOIN
            FOLDERS F
            ON F.folderid = PF.parentfolderid
)
,   PACKAGES AS
(
    -- pull information about stored SSIS packages
    SELECT
        P.name AS PackageName
    ,   P.id AS PackageId
    ,   P.description as PackageDescription
    ,   P.folderid
    ,   P.packageFormat
    ,   P.packageType
    ,   P.vermajor
    ,   P.verminor
    ,   P.verbuild
    ,   suser_sname(P.ownersid) AS ownername
    FROM
        msdb.dbo.sysssispackages P
)
SELECT 
    -- assumes default instance and localhost
    -- use serverproperty('servername') and serverproperty('instancename') 
    -- if you need to really make this generic
    -- *******************************************
    -- uncomment the following line for exporting
    -- **********************************************
    -- '"C:\Program Files\Microsoft SQL Server\120\DTS\Binn\dtutil" /sourceserver ' + @@SERVERNAME + ' /SQL "'+ F.FolderPath + '\' + P.PackageName + '" /ENCRYPT file;".' +  F.FolderPath + '\' + P.PackageName +'.dtsx";1' AS cmd
    -- *******************************************
    -- uncomment the following line for importing
    -- **********************************************
    -- '"C:\Program Files\Microsoft SQL Server\120\DTS\Binn\dtutil" /Destserver <destination server> /FILE ".' + F.FolderPath + '\' + P.PackageName+'.dtsx"' + ' /EN SQL;"' + F.FolderPath + '\' + P.PackageName + '";5'
FROM 
    FOLDERS F
    INNER JOIN
        PACKAGES P
        ON P.folderid = F.folderid
-- uncomment this if you want to filter out the 
-- native Data Collector packages
WHERE
     F.FolderPath <> '\Data Collector'
```

# Get missing indexes from DMV
```SQL
SELECT 
	DB_NAME(mid.database_id) AS DatabaseName,
	OBJECT_SCHEMA_NAME(mid.object_id, mid.database_id) AS SchemaName,
	OBJECT_NAME(mid.object_id, mid.database_id) AS ObjectName,
	migs.avg_user_impact,
	mid.equality_columns,
	mid.included_columns
FROM sys.dm_db_missing_index_groups mig
	INNER JOIN sys.dm_db_missing_index_group_stats migs
		ON migs.group_handle = mig.index_group_handle
	INNER JOIN sys.dm_db_missing_index_details mid
		ON mig.index_handle = mid.index_handle
ORDER BY DatabaseName, ObjectName, equality_columns;
```

# Get unused indexes from DMV
```SQL
SELECT 
	OBJECT_NAME(i.object_id) AS TableName,
	i.index_id,
	i.is_unique,
	ISNULL(user_seeks, 0) AS UserSeeks,
	ISNULL(user_scans, 0) AS UserScans,
	ISNULL(user_lookups, 0) AS UserLookups,
	ISNULL(user_updates, 0) AS UserUpdates
FROM sys.indexes i
	LEFT OUTER JOIN sys.dm_db_index_usage_stats ius
		ON ius.object_id = i.object_id AND ius.index_id = i.index_id
WHERE OBJECTPROPERTY(i.object_id, 'IsMSShipped') = 0
```

# Copy login with same password and SID
```SQL
USE [master]
SELECT LOGINPROPERTY('testlogin','PASSWORDHASH')
GO

SELECT SUSER_SID ('testlogin')
GO


CREATE LOGIN [testlogin] WITH PASSWORD = 0x020019814E12C5DCBE7D55C803E46D9CD7E349C9F9000BC392759E0CFD0CF98AA5A3D88B1A725F660A82FE7CAEAECA34E49AC5F08C188F5EF5DB99B06EC1E290EBFF4DF10EF1 HASHED, 
SID = 0xE7F3C36B478F5A4A96F179210CFF39C5, 
DEFAULT_DATABASE = [master],
DEFAULT_LANGUAGE=[us_english], 
CHECK_EXPIRATION = ON, 
CHECK_POLICY = ON
```

# Set database compatibility level
```SQL
ALTER DATABASE CURRENT SET COMPATIBILITY_LEVEL = 140
```
# Tracking the progress of the CREATE INDEX command
```SQL
;WITH agg AS
(
     SELECT SUM(qp.[row_count]) AS [RowsProcessed],
            SUM(qp.[estimate_row_count]) AS [TotalRows],
            MAX(qp.last_active_time) - MIN(qp.first_active_time) AS [ElapsedMS],
            MAX(IIF(qp.[close_time] = 0 AND qp.[first_row_time] > 0,
                    [physical_operator_name],
                    N'<Transition>')) AS [CurrentStep]
     FROM sys.dm_exec_query_profiles qp
     WHERE qp.[physical_operator_name] IN (N'Table Scan', N'Clustered Index Scan',
                                           N'Index Scan',  N'Sort')
     AND   qp.[session_id] IN (SELECT session_id from sys.dm_exec_requests where command IN ( 'CREATE INDEX','ALTER INDEX','ALTER TABLE') )
), comp AS
(
     SELECT *,
            ([TotalRows] - [RowsProcessed]) AS [RowsLeft],
            ([ElapsedMS] / 1000.0) AS [ElapsedSeconds]
     FROM   agg
)
SELECT [CurrentStep],
       [TotalRows],
       [RowsProcessed],
       [RowsLeft],
       CONVERT(DECIMAL(5, 2),
               (([RowsProcessed] * 1.0) / [TotalRows]) * 100) AS [PercentComplete],
       [ElapsedSeconds],
       (([ElapsedSeconds] / [RowsProcessed]) * [RowsLeft]) AS [EstimatedSecondsLeft],
       DATEADD(SECOND,
               (([ElapsedSeconds] / [RowsProcessed]) * [RowsLeft]),
               GETDATE()) AS [EstimatedCompletionTime]
FROM   comp;
```

# Reason for not backup up the log
```SQL
SELECT log_reuse_wait_desc
FROM sys.databases
WHERE name = 'DBName'
```

# Trace to discover locking

https://www.red-gate.com/simple-talk/sql/sql-tools/how-to-identify-blocking-problems-with-sql-profiler/
https://www.mssqltips.com/sqlservertip/1035/sql-server-performance-statistics-using-a-server-side-trace/

```SQL
/****************************************************/
/* Created by: SQL Server 2019 Profiler          */
/* Date: 25/11/2020  15:17:11         */
/****************************************************/


-- Create a Queue
declare @rc int
declare @TraceID int
declare @maxfilesize bigint
declare @DateTime datetime

set @DateTime = '2020-12-02 16:12:38.000'
set @maxfilesize = 5 

-- Please replace the text InsertFileNameHere, with an appropriate
-- filename prefixed by a path, e.g., c:\MyFolder\MyTrace. The .trc extension
-- will be appended to the filename automatically. If you are writing from
-- remote server to local drive, please use UNC path and make sure server has
-- write access to your network share

exec @rc = sp_trace_create @TraceID output, 0, N'E:\Traces\AF-BlockedProcesses', @maxfilesize, @Datetime
if (@rc != 0) goto error

-- Client side File and Table cannot be scripted

-- Set the events
declare @on bit
set @on = 1
exec sp_trace_setevent @TraceID, 137, 1, @on
exec sp_trace_setevent @TraceID, 137, 13, @on
exec sp_trace_setevent @TraceID, 137, 14, @on
exec sp_trace_setevent @TraceID, 137, 22, @on
exec sp_trace_setevent @TraceID, 137, 15, @on
exec sp_trace_setevent @TraceID, 137, 26, @on
exec sp_trace_setevent @TraceID, 137, 32, @on
exec sp_trace_setevent @TraceID, 137, 41, @on


-- Set the Filters
declare @intfilter int
declare @bigintfilter bigint

-- Set the trace status to start
exec sp_trace_setstatus @TraceID, 1

-- display trace id for future references
select TraceID=@TraceID
goto finish

error: 
select ErrorCode=@rc

finish: 
go


sp_trace_setstatus 2, 1

SELECT * FROM :: fn_trace_getinfo(2)

SELECT * 
FROM fn_trace_gettable('E:\Traces\AF-BlockedProcesses.trc', default)
GO

```

# Making database to read only – changing database to read_write

https://blog.sqlauthority.com/2011/04/16/sql-server-making-database-to-read-only-changing-database-to-readwrite/

```SQL
USE [master]
GO
ALTER DATABASE [TESTDB] SET READ_ONLY WITH NO_WAIT
GO

USE [master]
GO
ALTER DATABASE [TESTDB] SET READ_WRITE WITH NO_WAIT
GO
```

# Find outdated statistics
```SQL
SELECT DISTINCT
	OBJECT_SCHEMA_NAME(s.[object_id]) AS SchemaName,
	OBJECT_NAME(s.[object_id]) AS TableName,
	c.name AS ColumnName,
	s.name AS StatName,
	STATS_DATE(s.[object_id], s.stats_id) AS LastUpdated,
	DATEDIFF(d, STATS_DATE(s.[object_id], s.stats_id), getdate()) DaysOld,
	dsp.modification_counter,
	s.auto_created,
	s.user_created,
	s.no_recompute,
	s.[object_id],
	s.stats_id,
	sc.stats_column_id,
	sc.column_id
FROM sys.stats s
	JOIN sys.stats_columns sc
	ON sc.[object_id] = s.[object_id] AND sc.stats_id = s.stats_id
	JOIN sys.columns c ON c.[object_id] = sc.[object_id] AND c.column_id = sc.column_id
	JOIN sys.partitions par ON par.[object_id] = s.[object_id]
	JOIN sys.objects obj ON par.[object_id] = obj.[object_id]
	CROSS APPLY sys.dm_db_stats_properties(sc.[object_id], s.stats_id) AS dsp
WHERE OBJECTPROPERTY(s.object_id, 'IsUserTable') = 1
-- AND (s.auto_create = 1 OR s.user_created = 1) -- filter out stats for indexes
ORDER BY DaysOld;

-- exec sp_updatestats; if necessary
```

# Get empty space in each database file

```SQL
Create Table ##temp
(
    DatabaseName sysname,
    Name sysname,
    physical_name nvarchar(500),
    size decimal (18,2),
    FreeSpace decimal (18,2)
)   
Exec sp_msforeachdb '
Use [?];
Insert Into ##temp (DatabaseName, Name, physical_name, Size, FreeSpace)
    Select DB_NAME() AS [DatabaseName], Name,  physical_name,
    Cast(Cast(Round(cast(size as decimal) * 8.0/1024.0,2) as decimal(18,2)) as nvarchar) Size,
    Cast(Cast(Round(cast(size as decimal) * 8.0/1024.0,2) as decimal(18,2)) -
        Cast(FILEPROPERTY(name, ''SpaceUsed'') * 8.0/1024.0 as decimal(18,2)) as nvarchar) As FreeSpace
    From sys.database_files
'
Select * From ##temp
drop table ##temp

```
# Memory consumption

```SQL
SET NOCOUNT ON;
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SET LOCK_TIMEOUT 10000;

DECLARE @ServiceName nvarchar(100);
SET @ServiceName =
                  CASE
                    WHEN @@SERVICENAME = 'MSSQLSERVER' THEN 'SQLServer:'
                    ELSE 'MSSQL$' + @@SERVICENAME + ':'
                  END;

DECLARE @Perf TABLE (
  object_name nvarchar(20),
  counter_name nvarchar(128),
  instance_name nvarchar(128),
  cntr_value bigint,
  formatted_value numeric(20, 2),
  shortname nvarchar(20)
);
INSERT INTO @Perf (object_name, counter_name, instance_name, cntr_value, formatted_value, shortname)
  SELECT
    CASE
      WHEN CHARINDEX('Memory Manager', object_name) > 0 THEN 'Memory Manager'
      WHEN CHARINDEX('Buffer Manager', object_name) > 0 THEN 'Buffer Manager'
      WHEN CHARINDEX('Plan Cache', object_name) > 0 THEN 'Plan Cache'
      WHEN CHARINDEX('Buffer Node', object_name) > 0 THEN 'Buffer Node' -- 2008
      WHEN CHARINDEX('Memory Node', object_name) > 0 THEN 'Memory Node' -- 2012
      WHEN CHARINDEX('Cursor', object_name) > 0 THEN 'Cursor'
      ELSE NULL
    END AS object_name,
    CAST(RTRIM(counter_name) AS nvarchar(100)) AS counter_name,
    RTRIM(instance_name) AS instance_name,
    cntr_value,
    CAST(NULL AS decimal(20, 2)) AS formatted_value,
    SUBSTRING(counter_name, 1, PATINDEX('% %', counter_name)) shortname
  FROM sys.dm_os_performance_counters
  WHERE (object_name LIKE @ServiceName + 'Buffer Node%'     -- LIKE is faster than =. I have no idea why
  OR object_name LIKE @ServiceName + 'Buffer Manager%'
  OR object_name LIKE @ServiceName + 'Memory Node%'
  OR object_name LIKE @ServiceName + 'Plan Cache%')
  AND (counter_name LIKE '%pages %'
  OR counter_name LIKE '%Node Memory (KB)%'
  OR counter_name = 'Page life expectancy'
  )
  OR (object_name = @ServiceName + 'Memory Manager'
  AND counter_name IN ('Granted Workspace Memory (KB)', 'Maximum Workspace Memory (KB)',
  'Memory Grants Outstanding', 'Memory Grants Pending',
  'Target Server Memory (KB)', 'Total Server Memory (KB)',
  'Connection Memory (KB)', 'Lock Memory (KB)',
  'Optimizer Memory (KB)', 'SQL Cache Memory (KB)',
  -- for 2012
  'Free Memory (KB)', 'Reserved Server Memory (KB)',
  'Database Cache Memory (KB)', 'Stolen Server Memory (KB)')
  )
  OR (object_name LIKE @ServiceName + 'Cursor Manager by Type%'
  AND counter_name = 'Cursor memory usage'
  AND instance_name = '_Total'
  );

-- Add unit to 'Cursor memory usage'
UPDATE @Perf
SET counter_name = counter_name + ' (KB)'
WHERE counter_name = 'Cursor memory usage';

-- Convert values from pages and KB to MB and rename counters accordingly
UPDATE @Perf
SET counter_name = REPLACE(REPLACE(REPLACE(counter_name, ' pages', ''), ' (KB)', ''), ' (MB)', ''),
    formatted_value =
                     CASE
                       WHEN counter_name LIKE '%pages' THEN cntr_value / 128.
                       WHEN counter_name LIKE '%(KB)' THEN cntr_value / 1024.
                       ELSE cntr_value
                     END;

-- Delete some pre 2012 counters for 2012 in order to remove duplicates
DELETE P2008
  FROM @Perf P2008
  INNER JOIN @Perf P2012
    ON REPLACE(P2008.object_name, 'Buffer', 'Memory') = P2012.object_name
    AND P2008.shortname = P2012.shortname
WHERE P2008.object_name IN ('Buffer Manager', 'Buffer Node');

-- Update counter/object names so they look like in 2012
UPDATE PC
SET object_name = REPLACE(object_name, 'Buffer', 'Memory'),
    counter_name = ISNULL(M.NewName, counter_name)
FROM @Perf PC
LEFT JOIN (SELECT
  'Free' AS OldName,
  'Free Memory' AS NewName
UNION ALL
SELECT
  'Database',
  'Database Cache Memory'
UNION ALL
SELECT
  'Stolen',
  'Stolen Server Memory'
UNION ALL
SELECT
  'Reserved',
  'Reserved Server Memory'
UNION ALL
SELECT
  'Foreign',
  'Foreign Node Memory') M
  ON M.OldName = PC.counter_name
  AND NewName NOT IN (SELECT
    counter_name
  FROM @Perf
  WHERE object_name = 'Memory Manager')
WHERE object_name IN ('Buffer Manager', 'Buffer Node');


-- Build Memory Tree
DECLARE @MemTree TABLE (
  Id int,
  ParentId int,
  counter_name nvarchar(128),
  formatted_value numeric(20, 2),
  shortname nvarchar(20)
);

-- Level 5
INSERT @MemTree (Id, ParentId, counter_name, formatted_value, shortname)
  SELECT
    Id = 1226,
    ParentId = 1225,
    instance_name AS counter_name,
    formatted_value,
    shortname
  FROM @Perf
  WHERE object_name = 'Plan Cache'
  AND counter_name IN ('Cache')
  AND instance_name <> '_Total';

-- Level 4
INSERT @MemTree (Id, ParentId, counter_name, formatted_value, shortname)
  SELECT
    Id = 1225,
    ParentId = 1220,
    'Plan ' + counter_name AS counter_name,
    formatted_value,
    shortname
  FROM @Perf
  WHERE object_name = 'Plan Cache'
  AND counter_name IN ('Cache')
  AND instance_name = '_Total'

  UNION ALL

  SELECT
    Id = 1222,
    ParentId = 1220,
    counter_name,
    formatted_value,
    shortname
  FROM @Perf
  WHERE object_name = 'Cursor'
  OR (object_name = 'Memory Manager'
  AND shortname IN ('Connection', 'Lock', 'Optimizer', 'SQL'))

  UNION ALL

  SELECT
    Id = 1112,
    ParentId = 1110,
    counter_name,
    formatted_value,
    shortname
  FROM @Perf
  WHERE object_name = 'Memory Manager'
  AND shortname IN ('Reserved')
  UNION ALL
  SELECT
    Id = P.ParentID + 1,
    ParentID = P.ParentID,
    'Used Workspace Memory' AS counter_name,
    SUM(used_memory_kb) / 1024. AS formatted_value,
    NULL AS shortname
  FROM sys.dm_exec_query_resource_semaphores
  CROSS JOIN (SELECT
    1220 AS ParentID
  UNION ALL
  SELECT
    1110) P
  GROUP BY P.ParentID;

-- Level 3
INSERT @MemTree (Id, ParentId, counter_name, formatted_value, shortname)
  SELECT
    Id =
        CASE counter_name
          WHEN 'Granted Workspace Memory' THEN 1110
          WHEN 'Stolen Server Memory' THEN 1220
          ELSE 1210
        END,
    ParentId =
              CASE counter_name
                WHEN 'Granted Workspace Memory' THEN 1100
                ELSE 1200
              END,
    counter_name,
    formatted_value,
    shortname
  FROM @Perf
  WHERE object_name = 'Memory Manager'
  AND counter_name IN ('Stolen Server Memory', 'Database Cache Memory', 'Free Memory', 'Granted Workspace Memory');

-- Level 2
INSERT @MemTree (Id, ParentId, counter_name, formatted_value, shortname)
  SELECT
    Id =
        CASE
          WHEN counter_name = 'Maximum Workspace Memory' THEN 1100
          ELSE 1200
        END,
    ParentId = 1000,
    counter_name,
    formatted_value,
    shortname
  FROM @Perf
  WHERE object_name = 'Memory Manager'
  AND counter_name IN ('Total Server Memory', 'Maximum Workspace Memory');

-- Level 1
INSERT @MemTree (Id, ParentId, counter_name, formatted_value, shortname)
  SELECT
    Id = 1000,
    ParentId = NULL,
    counter_name,
    formatted_value,
    shortname
  FROM @Perf
  WHERE object_name = 'Memory Manager'
  AND counter_name IN ('Target Server Memory');

-- Level 4 -- 'Other Stolen Server Memory' = 'Stolen Server Memory' - SUM(Children of 'Stolen Server Memory')
INSERT @MemTree (Id, ParentId, counter_name, formatted_value, shortname)
  SELECT
    Id = 1222,
    ParentId = 1220,
    counter_name = '<Other Memory Clerks>',
    formatted_value = (SELECT
      SSM.formatted_value
    FROM @MemTree SSM
    WHERE Id = 1220)
    - SUM(formatted_value),
    shortname = 'Other Stolen'
  FROM @MemTree
  WHERE ParentId = 1220;

-- Results:

-- PLE and Memory Grants
SELECT
  [Counter Name] = P.counter_name + ISNULL(' (Node: ' + NULLIF(P.instance_name, '') + ')', ''),
  cntr_value AS Value,
  RecommendedMinimum =
                      CASE
                        WHEN P.counter_name = 'Page life expectancy' AND
                          R.Value <= 300 -- no less than 300
                        THEN 300
                        WHEN P.counter_name = 'Page life expectancy' AND
                          R.Value > 300 THEN R.Value
                        ELSE NULL
                      END
FROM @Perf P
LEFT JOIN -- Recommended PLE calculations
(SELECT
  object_name,
  counter_name,
  instance_name,
  CEILING(formatted_value / 4096. * 5) * 60 AS Value -- 300 per every 4GB of Buffer Pool memory or around 60 seconds (1 minute) per every 819MB
FROM @Perf PD
WHERE counter_name = 'Database Cache Memory') R
  ON R.object_name = P.object_name
  AND R.instance_name = P.instance_name
WHERE (P.object_name = 'Memory Manager'
AND P.counter_name IN ('Memory Grants Outstanding', 'Memory Grants Pending', 'Page life expectancy')
)
OR -- For NUMA
(
P.object_name = 'Memory Node'
AND P.counter_name = 'Page life expectancy'
AND (SELECT
  COUNT(DISTINCT instance_name)
FROM @Perf
WHERE object_name = 'Memory Node')
> 1
)
ORDER BY P.counter_name DESC, P.instance_name;

-- Get physical memory
-- You can also extract this information from sys.dm_os_sys_info but the column names have changed starting from 2012
IF OBJECT_ID('tempdb..#msver') IS NOT NULL
  DROP TABLE #msver
CREATE TABLE #msver (
  ID int,
  Name sysname,
  Internal_Value int,
  Value nvarchar(512)
);
INSERT #msver EXEC master.dbo.xp_msver 'PhysicalMemory';

-- Physical memory, config parameters and Target memory
SELECT
  min_server_mb = (SELECT
    CAST(value_in_use AS decimal(20, 2))
  FROM sys.configurations
  WHERE name = 'min server memory (MB)'),
  max_server_mb = (SELECT
    CAST(value_in_use AS decimal(20, 2))
  FROM sys.configurations
  WHERE name = 'max server memory (MB)'),
  target_mb = (SELECT
    formatted_value
  FROM @Perf
  WHERE object_name = 'Memory Manager'
  AND counter_name IN ('Target Server Memory')),
  physical_mb = CAST(Internal_Value AS decimal(20, 2))
FROM #msver;

-- Memory tree
;
WITH CTE
AS (SELECT
  0 AS lvl,
  counter_name,
  formatted_value,
  Id,
  NULL AS ParentId,
  shortname,
  formatted_value AS TargetServerMemory,
  CAST(NULL AS decimal(20, 4)) AS Perc,
  CAST(NULL AS decimal(20, 4)) AS PercOfTarget
FROM @MemTree
WHERE ParentId IS NULL
UNION ALL
SELECT
  CTE.lvl + 1,
  CAST(REPLICATE(' ', 6 * (CTE.lvl)) + NCHAR(124) + REPLICATE(NCHAR(183), 3) + MT.counter_name AS nvarchar(128)),
  MT.formatted_value,
  MT.Id,
  MT.ParentId,
  MT.shortname,
  CTE.TargetServerMemory,
  CAST(ISNULL(1.0 * MT.formatted_value / NULLIF(CTE.formatted_value, 0), 0) AS decimal(20, 4)) AS Perc,
  CAST(ISNULL(1.0 * MT.formatted_value / NULLIF(CTE.TargetServerMemory, 0), 0) AS decimal(20, 4)) AS PercOfTarget
FROM @MemTree MT
INNER JOIN CTE
  ON MT.ParentId = CTE.Id)
SELECT
  counter_name AS [Counter Name],
  CASE
    WHEN formatted_value > 0 THEN formatted_value
    ELSE NULL
  END AS [Memory MB],
  Perc AS [% of Parent],
  CASE
    WHEN lvl >= 2 THEN PercOfTarget
    ELSE NULL
  END AS [% of Target]
FROM CTE
ORDER BY ISNULL(Id, 10000), formatted_value DESC;
```
