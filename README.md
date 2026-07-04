# DBMonitor
Database monitor for D365FO
A good part of the logic in this extension relies on the the infrastructure of setup classes in D365FO (SysSetup and SysSetupAsync). If you want to test the logic in a Tier1 box, you need to run a 'Full Monty' database sync with the following command:
X:\AosService\PackagesLocalDirectory\bin\Microsoft.Dynamics.AX.Deployment.Setup.exe -bindir "X:\AosService\PackagesLocalDirectory" -metadatadir "X:\AosService\PackagesLocalDirectory" -sqluser axdbadmin -sqlserver localhost -sqldatabase axdb -setupmode sync -syncmode fullall -isazuresql false -sqlpwd  <axdbadmin_password>

## Lock monitor
First of all, in the menu *System administration->Inqueries->DatabaseMonitor->Database sessions* we have information about all current connections to the database. The logic of fhe lock monitor itself is based on a batch that wakes up periodically and scans the database view behind the form.The form contains the following fields:
- Root SPID - SPID of the database session that is currently creating a lock chain.
- Level  - nesting level of the current sessions
- Current SPID - SPID of the current database session
- Blocking SPID - the SPID of the session directly blocking the current session
- Current SQL - the SQL statement that is currently being executed in the session
- Cursor SQL - the SQL Statement that is or was executed in the last cursor open in the session
- Recent SQL - the last SQL Statemnent executed in the session
- User context - the context saved in the session via the SQL Function context_info()
- Login datetime - the login date and time of the current session
- Start time - the start time of the currently executing SQL Statement (if any)
- Host name - name of the client computer from where the session is run (in most cases - name of the AOS)
- Program name - name of the program executing the session
- Resource description - description of the locked resource (if any)
- Resource type - type of the locked resource (if any)
- Object name - name of the locked object (usually - a table name)
- Index name - name of the locked index (if any)
- User id - D365FO user name of the session (it is calculated via heuristic that works for 95-98% of times, do not expect 100% reliability)
- Session Id - Session number in D365FO (the one you can see in the online users form)
- Batch job id, job description and class name - are calculated from the session id. Again - it is a heuristical, do not expect 100% reliability
I should also mention that the kernel of D365FO reuses database sessions. All browsing from UI and form navigation is run from a single readonly database session shared by multiple user connections. When the user starts a database transaction, the kernel allocates a session to this user session and stores the user/session info in the context_info(). When the database transaction is ended, the session is returned to a pool and the context_info is not cleared. If you see a session from a specific user, it is quite possible that this user logged out and left office, so do not suspect that an inactive session with a certain user name actually corresponds an active user. From the other side, if the session is locking someone else, it is quite obviously active and the user/session info is correct
To start the lock monitor, you need to run the menu item *System administration->Periodic operations->DBM Lock monitor*. The menuItem spawns a batch that is scheduled to be run every 5 minutes. Normally, if the lock monitor is enabled, it startes, but never ends (so - keep in mind that it always consumes one batch thread of your batch servers).
The configuration of the lock monitor is specified on the new tab *Lock monitor* in the standard form *System job parameters*. The configuration has three parameters *Activate lock monitor*, *Monitoring frequency* and *Lock re-scan frequency*. If the first parameter is not set, the batch job simply ends immediately after every activation. If it is set, the job keeps running permanently. Every <Monitoring frequency> seconds it wakes up and check whether the Database sessions view has any locks. If no locks is found it sleeps for another <Monitoring frequency> seconds. If locks are found, it goes into more active scanning mode and re-checks Database session views every <Lock re-scan frequency> seconds. If on the next re-scan the picture is improved (some of the root blocking session ended transactions), the lock monitor dumps the last saved state of the Database session view into a table. If the lock is resolved after just 1 re-scan, it does not do anything. (We do not want to save blocking info for blocks that lasted less than <Lock re-scan frequency> seconds). If all locks are resolved, the lock monitor returns to the standard mode and rescans the session info every <Monitoring frequency> seconds.
The dumped lock info can be viewed via the menuItem *System administration->Inqueries->DatabaseMonitor->Locking history*
To clean up locking history, one can use the menuItem *System administration->Periodic operations->Clean up DB locking history*
**DO NOT TRY TO SET MONITORING AND RE-SCAN FREQUENCIES TO LESS THAN 15 AND 5 SECONDS** The process of fetching this info itself creates considerable load on the SQL Server, if you do it too often, you can kill your performance on attempt to improve it.
## DB Resource usage monitor
 
Azure SQL has a new standard DMV that keeps information about the database resource usage for the last 60 minutes: [sys.dm_db_resource_stats](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-resource-stats-azure-sql-database?view=fabric-sqldb)
If you click on the menuItem *System->Periodic operations->DBM Database resource usage monitor batch*, the system spawns the necessary batch. It activates every 20 minutes and copies the data from the DMV to the form System->Inquiries->Database resource usage. The form contains an exact copy of the data from the DMV and it means that the data granularity is 15 seconds. (and in one week we will have 40K new records). I added into the batch also an archival function that calculates an average/min/max of every utilization parameter and saves it into the form System->Inquiries->DB Usage archive. To configure the data archival and retention, I added three new parameters to the new tab *DB Resource usage monitoring options* of the *System Job Parameters* form : *Archival unit* specifies how many minutes of the utilization history we need to group into one record in the database usage archive. *Number of days to keep* specifies the number of days to keep in the Database resource usage table/form (do not set it to more than 7, I do not see any point in keeping high-frequency utilization info for a longer period). *Number of days to keep in archive specifies* the retention time in the archive.
If you try to run the job on Tier1 instance (where we do not have the DMV), the batch just does nothing. (It starts every 20 minutes, realizes that it is not an Azure SQL and quits). So, if you try to run the Database usage monitor on non-Azure SQL Server, it does nothing.
The current database resource usage log is available under *System administration->Inqueries->DatabaseMonitor->Database resource usage*
The archived database resource usage log is available under *System administration->Inqueries->DatabaseMonitor->DB Resource usage archive*

## Active queries 
The form shows currently active queries and their execution stats.
The list of the currently executed queries is available under *System administration->Inqueries->DatabaseMonitor->Active queries* <Description is to be improved>

## SQL Statement cache
The form shows content of the SQL Server statement cache
The form is available under *System administration->Inqueries->DatabaseMonitor->SQL Statement cache* <Description is to be improved>

## Query store
The form shows content of the SQL Server query store
The form is available under *System administration->Inqueries->DatabaseMonitor->Query store* <Description is to be improved>

## Tuning recommendations
The form shows content of SQL Server DMV [sys.dm_db_tuning_recommendations](https://learn.microsoft.com/en-us/sql/relational-databases/system-dynamic-management-views/sys-dm-db-tuning-recommendations-transact-sql?view=sql-server-ver17)
The form is available under *System administration->Inqueries->DatabaseMonitor->Tuning recommendations* 
<Description is to be improved>


