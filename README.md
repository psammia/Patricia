public async Task<Application> Insert_UserApplication_WithFiles(Insert_UserApplication_WithFiles_Request request)
{
    if (request == null)
        throw new ArgumentNullException(nameof(request));

    using (var dal = new DapperDal(_globalSettings.ConnString))
    {
        var parameters = new DynamicParameters();
        parameters.Add("P__CorrelationId", request.CorrelationId);
        parameters.Add("P__External_Id", request.External_Id);
        parameters.Add("P__TVP_Files", GetAppFilesDt(request.app_FilesList).AsTableValuedParameter());
        
        var result = await dal.ExecuteQueryAsync<Application>(
            "usp_InsertUserApplicationWithFiles",
            parameters,
            CommandType.StoredProcedure,
            DapperDal.CommandDirection.Update);

        return result?.FirstOrDefault() 
            ?? throw new InvalidOperationException("Failed to create application");
    }
}

private DataTable GetAppFilesDt(List<App_Files> app_Files)
{
    var dt = new DataTable("TVP_Files");
    dt.Columns.Add("File_Name", typeof(string));
    dt.Columns.Add("File_Type", typeof(string));
    dt.Columns.Add("File_Size", typeof(long));
    dt.Columns.Add("File_Data", typeof(byte[]));

    if (app_Files != null)
    {
        foreach (var appFile in app_Files)
        {
            var dr = dt.NewRow();
            dr["File_Name"] = appFile.File_Name ?? (object)DBNull.Value;
            dr["File_Type"] = appFile.File_Type ?? (object)DBNull.Value;
            dr["File_Size"] = appFile.File_Size;
            dr["File_Data"] = appFile.File_Data ?? (object)DBNull.Value;
            dt.Rows.Add(dr);
        }
    }

    return dt;
}
