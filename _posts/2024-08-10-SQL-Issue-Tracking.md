---
layout: post
title:  Tracking Issues with SQL when nobody is looking
date:   2024-08-10 08:21:59 -0500
categories: SQL
---

Using sp_whoisactive can be super handy when there is a conflict occuring while you are watching the system. However, sometimes the issues happen in the middle of the night or other random times. What I have done on several Windows Servers running SQL is to create a special database to hold a limited amount of regular sp_whoisactive results that I can inspect later. 



The High Level Process: 

1. Install sp_whoisactive on the SQL Server 
2. Create a Database to hold the results
3. Create a Table to hold the results
4. Create a SQL Agent Job to run sp_whoisactive on a regular basis
5. Use the data collected to find issues 



Step 1: Install sp_whoisactive. 

This is actually really easy. sp_whoisactive is maintained at https://github.com/amachanic/sp_whoisactive. Go get the official script and run it against the master DB on the server.



Step 2: Create a Database to hold the results. 

I am sure I could create a script to do this, but it is often easier to just use SSMS and create a database following the Microsoft best practices. Simple recovery is best since the data does not often have long term usefulness. For the sake of this example, the DB will be called 'MonitorDB'



Step 3: Create a Table to hold the results

This is my example script that will create a table in 'MonitorDB'

```sql
USE [MonitorDB]
GO

SET ANSI_NULLS ON
GO

SET QUOTED_IDENTIFIER ON
GO

SET ANSI_PADDING ON
GO

CREATE TABLE [dbo].[WhoIsActive](
	[RowId] [bigint] IDENTITY(1,1) NOT NULL,
	[dd hh:mm:ss.mss] [varchar](32) NULL,
	[session_id] [smallint] NOT NULL,
	[sql_text] [xml] NULL,
	[login_name] [nvarchar](128) NOT NULL,
	[wait_info] [nvarchar](4000) NULL,
	[tran_log_writes] [nvarchar](4000) NULL,
	[CPU] [varchar](30) NULL,
	[tempdb_allocations] [varchar](30) NULL,
	[tempdb_current] [varchar](30) NULL,
	[blocking_session_id] [smallint] NULL,
	[reads] [varchar](30) NULL,
	[writes] [varchar](30) NULL,
	[physical_reads] [varchar](30) NULL,
	[query_plan] [xml] NULL,
	[used_memory] [varchar](30) NULL,
	[status] [varchar](30) NOT NULL,
	[tran_start_time] [datetime] NULL,
	[open_tran_count] [varchar](30) NULL,
	[percent_complete] [varchar](30) NULL,
	[host_name] [nvarchar](128) NULL,
	[database_name] [nvarchar](128) NULL,
	[program_name] [nvarchar](128) NULL,
	[start_time] [datetime] NOT NULL,
	[login_time] [datetime] NULL,
	[request_id] [int] NULL,
	[collection_time] [datetime] NOT NULL,
 CONSTRAINT [PK_dbo_WhoIsActive_RowId] PRIMARY KEY CLUSTERED 
(
	[RowId] ASC
)WITH (PAD_INDEX  = OFF, STATISTICS_NORECOMPUTE  = OFF, IGNORE_DUP_KEY = OFF, ALLOW_ROW_LOCKS  = ON, ALLOW_PAGE_LOCKS  = ON, FILLFACTOR = 90) ON [PRIMARY]
) ON [PRIMARY] TEXTIMAGE_ON [PRIMARY]

GO

SET ANSI_PADDING OFF
GO
```



Step 4: Create a SQL Agent Job to run sp_whoisactive

This is my example job that runs the stored procedure, saves the results to the table, and then purges all but the most recent 50,000 results. This is nice to make sure that the MonitorDB does not grow too large. 

```sql
USE [msdb]
GO


BEGIN TRANSACTION
DECLARE @ReturnCode INT
SELECT @ReturnCode = 0

IF NOT EXISTS (SELECT name FROM msdb.dbo.syscategories WHERE name=N'[Uncategorized (Local)]' AND category_class=1)
BEGIN
EXEC @ReturnCode = msdb.dbo.sp_add_category @class=N'JOB', @type=N'LOCAL', @name=N'[Uncategorized (Local)]'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback

END

DECLARE @jobId BINARY(16)
EXEC @ReturnCode =  msdb.dbo.sp_add_job @job_name=N'_ADMIN_WhoIsActive Collection', 
		@enabled=1, 
		@notify_level_eventlog=0, 
		@notify_level_email=0, 
		@notify_level_netsend=0, 
		@notify_level_page=0, 
		@delete_level=0, 
		@description=N'No description available.', 
		@category_name=N'[Uncategorized (Local)]', 
		@owner_login_name=N'sa', @job_id = @jobId OUTPUT
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback

EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'WhoIsActive Collection', 
		@step_id=1, 
		@cmdexec_success_code=0, 
		@on_success_action=3, 
		@on_success_step_id=0, 
		@on_fail_action=2, 
		@on_fail_step_id=0, 
		@retry_attempts=0, 
		@retry_interval=0, 
		@os_run_priority=0, @subsystem=N'TSQL', 
		@command=N'USE [MonitorDB]
GO

IF OBJECT_ID (''dbo.WhoIsActive_Temp'') IS NOT NULL
	BEGIN
		DROP TABLE dbo.WhoIsActive_Temp;
	END

DECLARE
    @destination_table VARCHAR(4000) ,
    @msg NVARCHAR(1000) ;
SET @destination_table = ''WhoIsActive_Temp'';

DECLARE @schema VARCHAR(4000) ;
EXEC sp_WhoIsActive
@get_transaction_info = 1,
@get_plans = 1,
@return_schema = 1,
@schema = @schema OUTPUT ;

SET @schema = REPLACE(@schema, ''<table_name>'', @destination_table) ;

PRINT @schema
EXEC(@schema) ;


    DECLARE @numberOfRuns INT ;
    SET @numberOfRuns = 1 ;
    WHILE @numberOfRuns > 0
        BEGIN;
            EXEC dbo.sp_WhoIsActive @get_transaction_info = 1, @get_plans = 1,
                @destination_table = @destination_table ;
            SET @numberOfRuns = @numberOfRuns - 1 ;
            IF @numberOfRuns > 0
                BEGIN
                    SET @msg = CONVERT(CHAR(19), GETDATE(), 121) + '': '' +
                     ''Logged info. Waiting...''
                    RAISERROR(@msg,0,0) WITH nowait ;
                    WAITFOR DELAY ''00:00:05''
                END
            ELSE
                BEGIN
                    SET @msg = CONVERT(CHAR(19), GETDATE(), 121) + '': '' + ''Done.''
                    RAISERROR(@msg,0,0) WITH nowait ;
                END
        END ;
    GO

INSERT INTO [dbo].[WhoIsActive]
           ([dd hh:mm:ss.mss]
           ,[session_id]
           ,[sql_text]
           ,[login_name]
           ,[wait_info]
           ,[tran_log_writes]
           ,[CPU]
           ,[tempdb_allocations]
           ,[tempdb_current]
           ,[blocking_session_id]
           ,[reads]
           ,[writes]
           ,[physical_reads]
           ,[query_plan]
           ,[used_memory]
           ,[status]
           ,[tran_start_time]
           ,[open_tran_count]
           ,[percent_complete]
           ,[host_name]
           ,[database_name]
           ,[program_name]
           ,[start_time]
           ,[login_time]
           ,[request_id]
           ,[collection_time]
		   )
SELECT
            [dd hh:mm:ss.mss]
           ,[session_id]
           ,[sql_text]
           ,[login_name]
           ,[wait_info]
           ,[tran_log_writes]
           ,[CPU]
           ,[tempdb_allocations]
           ,[tempdb_current]
           ,[blocking_session_id]
           ,[reads]
           ,[writes]
           ,[physical_reads]
           ,[query_plan]
           ,[used_memory]
           ,[status]
           ,[tran_start_time]
           ,[open_tran_count]
           ,[percent_complete]
           ,[host_name]
           ,[database_name]
           ,[program_name]
           ,[start_time]
           ,[login_time]
           ,[request_id]
           ,[collection_time]
FROM	[dbo].[WhoIsActive_Temp];
DROP TABLE [dbo].[WhoIsActive_Temp];', 
		@database_name=N'master', 
		@flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback

EXEC @ReturnCode = msdb.dbo.sp_add_jobstep @job_id=@jobId, @step_name=N'Prune dbo.WhoisActive Table', 
		@step_id=2, 
		@cmdexec_success_code=0, 
		@on_success_action=1, 
		@on_success_step_id=0, 
		@on_fail_action=2, 
		@on_fail_step_id=0, 
		@retry_attempts=0, 
		@retry_interval=0, 
		@os_run_priority=0, @subsystem=N'TSQL', 
		@command=N'USE [MonitorDB]
GO
SET NOCOUNT ON;
DECLARE @ID BIGINT
	, @KeepLast BIGINT = 50000

-- dbo.WhoIsActive cleanup
SELECT	@ID = MAX(RowId)- @KeepLast
FROM	dbo.WhoIsActive WITH(NOLOCK);

DELETE FROM dbo.WhoIsActive
WHERE	RowId < @ID;
', 
		@database_name=N'master', 
		@flags=0
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_update_job @job_id = @jobId, @start_step_id = 1
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobschedule @job_id=@jobId, @name=N'WhoIsActive Collection', 
		@enabled=1, 
		@freq_type=4, 
		@freq_interval=1, 
		@freq_subday_type=4, 
		@freq_subday_interval=1, 
		@freq_relative_interval=0, 
		@freq_recurrence_factor=0, 
		@active_start_date=20160607, 
		@active_end_date=99991231, 
		@active_start_time=0, 
		@active_end_time=235959
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
EXEC @ReturnCode = msdb.dbo.sp_add_jobserver @job_id = @jobId, @server_name = N'(local)'
IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback
COMMIT TRANSACTION
GOTO EndSave
QuitWithRollback:
    IF (@@TRANCOUNT > 0) ROLLBACK TRANSACTION
EndSave:

GO
```



Step 5: Use the Data

There is lots of good data here that might help an investigation. Narrow the issues by date range using the 'collection_time'. Another hint would be looking at the 'blocking_session_id' to help identify conflicts. Lastly, looking at the query plan in SSMS might help identify missing indexes or problems areas.


