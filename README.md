 OLEDB Exception: 
System.Data.OleDb.OleDbException: SQL0901: SQL system error.
Cause . . . . . :   An SQL system error has occurred.  The current SQL statement cannot be completed successfully.  The error will not prevent other SQL statements from being processed. Previous messages may indicate that there is a problem with the SQL statement and SQL did not correctly diagnose the error. The previous message identifier was CPF4204. Internal error type 3107 has occurred. If precompiling, processing will not continue beyond this statement. Recovery  . . . :   See the previous messages to determine if there is a problem with the SQL statement. To view the messages, use the DSPJOBLOG command if running interactively, or the WRKJOB command to view the output of a precompile.  An application program receiving this return code may attempt further SQL statements.  Correct any errors and try the request again.
   at System.Data.OleDb.OleDbCommand.ExecuteCommandTextErrorHandling(OleDbHResult hr)
   at System.Data.OleDb.OleDbCommand.ExecuteCommandTextForSingleResult(tagDBPARAMS dbParams, Object& executeResult)
   at System.Data.OleDb.OleDbCommand.ExecuteCommandText(Object& executeResult)
   at System.Data.OleDb.OleDbCommand.ExecuteReaderInternal(CommandBehavior behavior, String method)
   at System.Data.OleDb.OleDbCommand.ExecuteReader(CommandBehavior behavior)
   at System.Data.OleDb.OleDbCommand.System.Data.IDbCommand.ExecuteReader(CommandBehavior behavior)
   at System.Data.Common.DbDataAdapter.FillInternal(DataSet dataset, DataTable[] datatables, Int32 startRecord, Int32 maxRecords, String srcTable, IDbCommand command, CommandBehavior behavior)
   at System.Data.Common.DbDataAdapter.Fill(DataTable[] dataTables, Int32 startRecord, Int32 maxRecords, IDbCommand command, CommandBehavior behavior)
   at System.Data.Common.DbDataAdapter.Fill(DataTable dataTable)
   at AuditProOM.Adapters.OLEDBAdapter.ExecuteSelectRequest(String Command, DataTable& Result, OleDbConnection Sc)

  Call Stack: 
   at AuditProOM.Adapters.OLEDBAdapter.ExecuteSelectRequest(String Command, DataTable& Result, OleDbConnection Sc)
   at AuditProOM.Program.ProcessRichelieuGCCDailyTransactionControl()
   at AuditProOM.Program.Main(String[] args)


   Message: SQL0901: SQL system error.
Cause . . . . . :   An SQL system error has occurred.  The current SQL statement cannot be completed successfully.  The error will not prevent other SQL statements from being processed. Previous messages may indicate that there is a problem with the SQL statement and SQL did not correctly diagnose the error. The previous message identifier was CPF4204. Internal error type 3107 has occurred. If precompiling, processing will not continue beyond this statement. Recovery  . . . :   See the previous messages to determine if there is a problem with the SQL statement. To view the messages, use the DSPJOBLOG command if running interactively, or the WRKJOB command to view the output of a precompile.  An application program receiving this return code may attempt further SQL statements.  Correct any errors and try the request again.
   Native Error: -901
   Source: IBMDA400 Command

