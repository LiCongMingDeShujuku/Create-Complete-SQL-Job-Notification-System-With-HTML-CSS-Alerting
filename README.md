![CLEVER DATA GIT REPO](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/0-clever-data-github.png "李聪明的数据库")

# 使用HTML CSS警报创建完整的SQL作业通知系统
#### Create Complete SQL Job Notification System With HTML CSS Alerting
**发布-日期: 2015年07月15日 (评论)**

## Contents

- [中文](#中文)
- [English](#English)
- [SQL Logic](#Logic)
- [Build Info](#Build-Info)
- [Author](#Author)
- [License](#License) 


## 中文
以下逻辑（logic）将配置SMTP电子邮件通知，创建作业以HTML和CSS格式发送通知，并在MSDB SysJobHistoryDatabase上创建触发器（trigger）。每次将“failure”条目插入表格时，触发器就会启动作业。以防这个逻辑（logic）
被博客自动化程序阻挡了，我在下面附上了一个包含所有逻辑（logic）的PDF。

## English
The following logic will configure your SMTP Email notification, Create a Job to send notifications out in an HTML & CSS format, and create a Trigger on the MSDB SysJobHistoryDatabase. Each time a ‘failure’ entry is inserted into the table, the trigger kicks off the Job.
In case this logic gets butchered by the blog auto-whatevers I’m attaching a PDF that has all the logic in it.


---
## Logic
```SQL
-- CREATE complete job notification sql逻辑（logic）-创建完整的作业通知 

USE [msdb]
go

BEGIN TRANSACTIONDECLARE 
	@ReturnCode INT 
SELECT @ReturnCode = 0 
IF NOT EXISTS
( 
       SELECT NAME 
       FROM   msdb.dbo.syscategories 
       WHERE  NAME=N'[Uncategorized (Local)]' 
       AND    category_class=1) 
BEGIN 
  EXEC @ReturnCode 
    = msdb.dbo.sp_add_category @class=N'JOB', 
    @type=N'LOCAL', 
    @name=N'[Uncategorized (Local)]' 
  IF (@@ERROR <> 0 
  OR 
  @ReturnCode <> 0) GOTO quitwithrollback
END
go ;

DECLARE @jobId BINARY(16)
EXEC @ReturnCode  			= msdb.dbo.Sp_add_job 
	@job_name 				=N'SEND SQL JOB ALERTS (beta)'
  , @enabled				=1
  , @notify_level_eventlog	=0
  , @notify_level_email		=0
  , @notify_level_netsend	=0
  , @notify_level_page		=0
  , @delete_level			=0
  , @description			=N'This is a new SQL Job notification.  It''s designed to send the a short error report about step failure within a job.'
  , @category_name			=N'[Uncategorized (Local)]'
  , @owner_login_name		=N'sa'
  , @job_id 				= @jobId outputIF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO quitwithrollback 

/************/

EXEC @ReturnCode 			= msdb.dbo.sp_add_jobstep @job_id=@jobId
  , @step_name				=N'Send email about step failure'
  , @step_id 				=1
  , @cmdexec_success_code	=0
  , @on_success_action		=1
  , @on_success_step_id		=0
  , @on_fail_action			=2
  , @on_fail_step_id		=0
  , @retry_attempts			=0
  , @retry_interval			=0
  , @os_run_priority		=0
  , @subsystem				=N'TSQL'
  , @command				=N'

  use msdb; 
  set nocount on 
  ---------------------------------------------------------------------- -
  - Configure SQL Database Mail if it''s not already configured. 配置SQL数据库邮件，如果尚未配置。

  if (select top 1 name from msdb..sysmail_profile) is null 
  	begin

  ---------------------------------------------------------------------- 
  --	Enable SQL Database Mail 启用SQL数据库邮件 
  
  exec master..sp_configure ''show advanced options'',1 reconfigure; exec master..sp_configure ''database mail xps'',1 reconfigure; 
 
  ---------------------------------------------------------------------- 
  --	Add a profile 添加文件 

  execute msdb.dbo.sysmail_add_profile_sp @profile_name 	= ''SQLDatabaseMailProfile'' 
  ,	@description = ''SQLDatabaseMail''; 

  ---------------------------------------------------------------------- 
  --	Add the account names you want to appear in the email message. 添加要在电子邮件中显示的帐户名称。 

  execute msdb.dbo.sysmail_add_account_sp 
  	@account_name 		= ''sqldatabasemail@mydomain.com'' 
  ,	@email_address 		= ''sqldatabasemail@mydomain.com'' 
  , @mailserver_name 	= ‘‘MySMTPServerName.MyDomain.com’’ 
  -- optional configs
  --,	@port           = ####  
  --,	@enable_ssl     = 1 
  --,	@username 		=''MySQLDatabaseMailProfile'' 
  --,	@password 		=''MyPassword'' 
  --	optional


  --	Adding the account to the profile 将帐户添加到文件 
  execute msdb.dbo.sysmail_add_profileaccount_sp 
  	@profile_name 		= ''SQLDatabaseMailProfile'' 
  ,	@account_name 		= ''sqldatabasemail@mydomain.com'' 
  , @sequence_number 	= 1; 

  --	Give access to new database mail profile (DatabaseMailUserRole) 授予对新数据库邮件文件的访问 
  execute msdb.dbo.sysmail_add_principalprofile_sp 
  	@profile_name 		= ''SQLDatabaseMailProfile'' 
  ,	@principal_id  		= 0 
  , @is_default			= 1; 
 
  ---------------------------------------------------------------------- 
  -- Get Server info for test message 设置测试消息的服务器信息 
  go ; 

  declare 	@get_basic_server_name 						varchar(255) 
  declare 	@get_basic_server_name_and_instance_name		varchar(255) 
  declare 	@basic_test_subject_message 					varchar(255) 
  declare 	@basic_test_body_message 						varchar(max) 
  set 		@get_basic_server_name 						= (select cast(serverproperty(''servername'') as varchar(255))) 
  set 		@get_basic_server_name_and_instance_name 	= (select  replace(cast(serverproperty(''servername'') as varchar(255)), ''\'', ''   SQL Instance: '')) 
  set 		@basic_test_subject_message                 = ''Test SMTP email from SQL Server: '' + @get_basic_server_name_and_instance_name 
  set 		@basic_test_body_message 					= ''This is a test SMTP email from SQL Server:  '' + @get_basic_server_name_and_instance_name + char(10) + char(10) + ''If you see this.  It''''s working perfectly :)''

  ---------------------------------------------------------------------- 
  -- Send quick email to confirm email is properly working.  发送快速电子邮件以确认电子邮件是否正常工作 go ; 

  EXEC msdb.dbo.sp_send_dbmail 
  	@profile_name 		= ''SQLDatabaseMailProfile'' 
  ,	@recipients 		= “MyEmailAddressGoesHere 
  , @subject 			= @basic_test_subject_message 
  , @body 				= @basic_test_body_message;


  --	Confirm message send 确认邮件发送 
  -- 	select * from msdb..sysmail_allitems end go ; 

  ---------------------------------------------------------------------- 
  -- get basic server info. 获取基本的服务器信息 go ; 

  declare @server_name_basic				varchar(255) 
  declare @server_name_instance_name 		varchar(255) 
  declare @server_time_zone 				varchar(255) 
  set     @server_name_basic 				= (select cast(serverproperty(''servername'') 			as varchar(255))) 
  set     @server_name_instance_name 		= (select  replace(cast(serverproperty(''servername'') 	as varchar(255)), ''\'', ''   SQL Instance: '')) 

  exec 		master.dbo.xp_regread 
  		''hkey_local_machine''
  , 	''system\currentcontrolset\control\timezoneinformation''
  ,		''timezonekeyname''
  , 	@server_time_zone out

  ---------------------------------------------------------------------- 
  --	set message subject. 设置消息对象 

  declare 	@message_subject	varchar(255) 
  set 		@message_subject 	= ''SQL Job failure found on Server:  '' + @server_name_instance_name go ; 

  ---------------------------------------------------------------------- 
  -- find most recent error step error in sysjobhistory and pull name based on instance_id from history table. 在sysjobhistory中查找最近的错误步骤，并根据历史记录表格中的instance_id拉取名称。

  declare 	@last_error 				varchar(255) 
  declare 	@last_error_job_name 		varchar(255) 
  set 		@last_error 				= ( select top 1 instance_id from sysjobhistory where message like ''%The step failed%'' order by run_date desc ) 
  set       @last_error_job_name  		= ( select sj.name from sysjobs sj join sysjobhistory sjh on sj.job_id = sjh.job_id where instance_id = @last_error )

  ---------------------------------------------------------------------- 
  --	create temp table to store error information. 创建临时表格保存错误信息

  if object_id(''tempdb..#agent_job_step_error_report'') is not null 
  	drop table #agent_job_step_error_report
  create table #agent_job_step_error_report 
( 
	[id] 				int identity (1,1) 
,	[server_name] 		varchar(255) 
,	[time_of_error]  	varchar(255) 
,	[job_name] 			varchar(255) 
,	[step_id]  			int not null 
,	[step_name] 		varchar(255) 
,	[duration]   		varchar(255) 
,	[error_message] 	varchar(max) 
)


  ---------------------------------------------------------------------- 
  --	get information from job system tables for job step error report. 为了作业步骤的错误报告，从作业系统表格中获取信息 go ; 

  insert into #agent_job_step_error_report ([server_name], [time_of_error], [job_name], [step_id], [step_name], [duration], [error_message]) 
  	select 
  			''server name'' 	= @@servername 
  		,	''time of error'' 	= datename(dw, msdb.dbo.agent_datetime(run_date, run_time) ) + '':  '' + convert(char, msdb.dbo.agent_datetime(run_date, run_time) , 9) 
  		,	''job name'' 		= sj.name 
  		, 	''step id'' 		= sjh.step_id 
  		, 	''step name'' 		= sjh.step_name 
  		,	''duration'' 		= CAST(sjh.run_duration/10000 as varchar)  + '':'' + CAST(sjh.run_duration/100%100 as varchar) + '':'' + CAST(sjh.run_duration%100 as varchar) 
  		,	''error message'' 	= sjh.message 
  from 
  	msdb..sysjobs sj join msdb..sysjobhistory sjh on sj.job_id = sjh.job_id 
  where instance_id = @last_error 
  order by 
  		sj.name
  , 	step_id asc 

  ---------------------------------------------------------------------- 
  -- create temp table to store job information 创建临时表格来保存作业信息 go ; 

  if object_id(''tempdb..#agent_job_information'') is not null 
  	drop table #agent_job_information go ; 

  	create table #agent_job_information 
  		( 
  				[id]			int identity (1,1) 
  			,   [job_name]		varchar(255) 
  			,	[step_id]		int not null 
  			,	[step_name]		varchar(255) 
  			,	[process_type]	varchar(255) 
  			,	[last_ran]		varchar(255)
  		) 

  ---------------------------------------------------------------------- 
  -- get all job, and step info for quick reference including the previous run duration and the last known run timestamp before the most previous error. 获取所有作业和步骤信息以便快速参考，包括之前的运行持续时间和最后一次错误之前的最后一个已知运行时间戳。 

  insert into #agent_job_information 
  		(
  				[job_name]
  			, 	[step_id]
  			, 	[step_name]
  			, 	[process_type]
  			, 	[last_ran]
  		) 
  select 
  		''job name''		= sj.name 
  	,	''step id''     	= sjs.step_id 
  	,   ''step name''		= sjs.step_name 
  	,   ''process type''	= sjs.subsystem 
  	,	''last ran''		= datename(dw, dateadd(millisecond, sjs.last_run_time,convert(datetime,cast(nullif(sjs.last_run_date,0) as nvarchar(10))))) + '':  '' + convert(char, dateadd(millisecond, sjs.last_run_time,convert(datetime,cast(nullif(sjs.last_run_date,0) as nvarchar(10)))), 9) 
  from 
  	msdb..sysjobs sj join msdb..sysjobsteps sjs on sj.job_id = sjs.job_id 
  where 
  	sj.name = @last_error_job_name 
  order by 
  	sj.name
  ,	sjs.step_id asc 

  ---------------------------------------------------------------------- 
  -- create conditions for html tables in top and mid sections of email. 在电子邮件的顶部和中间部分为html表格创造条件。 go ; 

  declare @xml_top	NVARCHAR(MAX) 
  declare @xml_mid 	NVARCHAR(MAX) 
  declare @body_top NVARCHAR(MAX) 
  declare @body_mid NVARCHAR(MAX) 

  ---------------------------------------------------------------------- 
  --	set xml top table td''s 设置xml顶部表格td' 
  --    CREATE html TABLE objectFOR: #agent_job_step_error_report 为#agent_job_step_error_report创建html表格对象

  SET @xml_top = cast( 
                      ( 
                      SELECT 
                      	[server_name]	AS ''td'' 
                      ,	'''' 
                      , [time_of_error]	AS ''td'' 
                      , '''' 
                      , [job_name]		AS ''td'' 
                      , '''' 
                      , [step_id]		AS ''td'' 
                      , '''' 
                      , [step_name]		AS ''td'' 
                      , '''' 
                      , [duration]		AS ''td'' 
                      , '''' 
                      , [error_message]	AS ''td'' 
                      FROM   
                      	#agent_job_step_error_report 
                             --order by rank 
                             --按排名排序 
                         FOR xml path(''tr'') 
                      ,elements) AS nvarchar(max) ) 
  go ; 

---------------------------------------------------------------------- 
-- set xml mid table td''s设置xml中间表格td''s 
-- CREATE html TABLE objectFOR: #agent_job_information 为#agent_job_information创建html表格对象

SET @xml_mid = cast( 
                      ( 
                      SELECT [job_name] AS ''td'' , 
                             '''' , 
                             [step_id] AS ''td'' , 
                             '''' , 
                             [step_name] AS ''td'' , 
                             '''' , 
                             [process_type] AS ''td'' , 
                             '''' , 
                             [last_ran] AS ''td'' go ;FROM #agent_job_information ORDER BY [job_name], 
                      [step_id] ASC FOR xml path(''tr'') , 
                      elements) AS nvarchar(max) ) go ; 

----------------------------------------------------------------------
go ;

SET @body_top = ''<html> <head> <style> 

	h1{ font-family: sans-serif; font -size: 110%; } 
	h3{ font -family: sans-serif; color: red; };

	TABLE, td, tr, 
	th { font-family: sans-serif; border: 1px solid black; border -collapse: collapse; };
	th { text-align: LEFT; background -color: gray; color: white; padding: 5px; };
	td { padding: 5px; } </style> </head> 

	<body> <h3>'' + @message_subject + ''</h3> <h1>NOTE: the server time IS operatingON: '' 
	+ @server_time_zone + '' 注意：服务器时间正在运行：''
	+ @server_time_zone + '' 
		<TABLE border = 1> 
			<tr> <th> server NAME </th> 
			<th> time OF error </th> 
			<th> job NAME </th> 
			<th> step id </th> 
			<th> step NAME </th> 
			<th> duration </th> 
			<th> error message </th></tr>'';

		SET @body_top = @body_top + @xml_top + ''
		</table>;

	<h1>quickREFERENCE: job info</h1> 快速参考：作业信息</h1>;
	<TABLE border = 1> 
		<tr> <th> job NAME </th> 
		<th> step id </th> 
		<th> step NAME </th> 
		<th> process type </th> 
		<th> last ran </th> </tr>''; 

		+ @xml_mid + ''
	</TABLE> 

	<h1>Go to the server BY pasting IN the following textUNDER: start-run, OR (  win + r ) </h1> 
	<h1>mstsc -V:' ' + @server_name_basic + ''</h1>'' 
	+ ''</body></html>'' 

----------------------------------------------------------------------
-- send email. 发送邮件 

EXEC msdb.dbo.sp_send_dbmail 
  		@profile_name 	= ''sqldatabasemailprofile'' 
 ,		@recipients 	= “myemailaddressgoeshere 
 , 		@subject 		= @message_subject 
 ,		@body 			= @body_top 
 ,		@body_format 	= ''html'';

  DROP TABLE #agent_job_step_error_report
  DROP TABLE #agent_job_information 
  , 	@database_name=N'master'
  , 	@flags=0 IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback 
  EXEC 	@ReturnCode = msdb.dbo.sp_update_job @job_id = @jobId, @start_step_id = 1 IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback EXEC @ReturnCode = msdb.dbo.sp_add_jobserver @job_id = @jobId, 	@server_name = N'(local)' IF (@@ERROR <> 0 OR @ReturnCode <> 0) GOTO QuitWithRollback COMMIT TRANSACTION GOTO EndSave QuitWithRollback: IF (@@TRANCOUNT > 0) ROLLBACK TRANSACTION 
  EndSave:   
  go ; 

  use msdb; 
  set nocount on 
  set ansi_nulls on 
  set quoted_identifier on 
  go 

  create trigger [dbo].[check_for_job_failure] on  msdb..sysjobhistory 
  	after insert as begin 
  	set nocount on 
  	declare @is_fail   int 
  	set     @is_fail   = (select case when [message] like '%the step failed%' then 1 else 0 end from msdb..sysjobhistory where instance_id in (select max(instance_id) from msdb..sysjobhistory)) 
  	if 		@is_fail   = 1 
  		begin 
  			exec msdb.dbo.sp_start_job @job_name = 'send sql job alerts (beta)' 
  		end 
  	end 
  	go ; <pre>

```



[![WorksEveryTime](https://forthebadge.com/images/badges/60-percent-of-the-time-works-every-time.svg)](https://shitday.de/)

## Build-Info

| Build Quality | Build History |
|--|--|
|<table><tr><td>[![Build-Status](https://ci.appveyor.com/api/projects/status/pjxh5g91jpbh7t84?svg?style=flat-square)](#)</td></tr><tr><td>[![Coverage](https://coveralls.io/repos/github/tygerbytes/ResourceFitness/badge.svg?style=flat-square)](#)</td></tr><tr><td>[![Nuget](https://img.shields.io/nuget/v/TW.Resfit.Core.svg?style=flat-square)](#)</td></tr></table>|<table><tr><td>[![Build history](https://buildstats.info/appveyor/chart/tygerbytes/resourcefitness)](#)</td></tr></table>|

## Author

- **李聪明的数据库 Lee's Clever Data**
- **Mike的数据库宝典 Mikes Database Collection**
- **李聪明的数据库** "Lee Songming"

[![Gist](https://img.shields.io/badge/Gist-李聪明的数据库-<COLOR>.svg)](https://gist.github.com/congmingshuju)
[![Twitter](https://img.shields.io/badge/Twitter-mike的数据库宝典-<COLOR>.svg)](https://twitter.com/mikesdatawork?lang=en)
[![Wordpress](https://img.shields.io/badge/Wordpress-mike的数据库宝典-<COLOR>.svg)](https://mikesdatawork.wordpress.com/)

---
## License
[![LicenseCCSA](https://img.shields.io/badge/License-CreativeCommonsSA-<COLOR>.svg)](https://creativecommons.org/share-your-work/licensing-types-examples/)

![Lee Songming](https://raw.githubusercontent.com/LiCongMingDeShujuku/git-resources/master/1-clever-data-github.png "李聪明的数据库")

