#region Insert User Application With Files

public async Task<Application> Insert_UserApplication_WithFiles(Insert_UserApplication_WithFiles_Request request)
{
    if (request == null)
        throw new ArgumentNullException(nameof(request));

    // Process files: convert base64 to byte array and extract metadata
    var processedFiles = ProcessFilesList(request.app_FilesList);

    using (var dal = new DapperDal(_globalSettings.ConnString))
    {
        var parameters = new DynamicParameters();
        parameters.Add("P__CorrelationId", request.CorrelationId);
        parameters.Add("P__External_Id", request.External_Id);
        parameters.Add("P__TVP_Files", GetAppFilesDt(processedFiles).AsTableValuedParameter());
        
        var result = await dal.ExecuteQueryAsync<Application>(
            "usp_InsertUserApplicationWithFiles",
            parameters,
            CommandType.StoredProcedure,
            DapperDal.CommandDirection.Update);

        return result?.FirstOrDefault() 
            ?? throw new InvalidOperationException("Failed to create application");
    }
}

private List<App_Files> ProcessFilesList(List<App_Files> app_FilesList)
{
    if (app_FilesList == null || !app_FilesList.Any())
        return new List<App_Files>();

    var processedFiles = new List<App_Files>();

    foreach (var file in app_FilesList)
    {
        try
        {
            var processedFile = new App_Files
            {
                File_Name = file.File_Name,
                File_Type = file.File_Type
            };

            // Convert base64 to byte array
            if (!string.IsNullOrWhiteSpace(file.File_Data_Base64))
            {
                // Remove data URL prefix if present (e.g., "data:image/png;base64,")
                string base64String = file.File_Data_Base64;
                if (base64String.Contains(","))
                {
                    base64String = base64String.Substring(base64String.IndexOf(",") + 1);
                }

                // Convert to byte array
                processedFile.File_Data = Convert.FromBase64String(base64String);
                
                // Calculate file size in bytes
                processedFile.File_Size = processedFile.File_Data.Length;

                // Extract file name from base64 data URL if not provided
                if (string.IsNullOrWhiteSpace(processedFile.File_Name))
                {
                    processedFile.File_Name = ExtractFileNameFromDataUrl(file.File_Data_Base64);
                }

                // Extract file type (MIME type) if not provided
                if (string.IsNullOrWhiteSpace(processedFile.File_Type))
                {
                    processedFile.File_Type = ExtractMimeTypeFromDataUrl(file.File_Data_Base64);
                }
            }
            else if (file.File_Data != null)
            {
                // If File_Data is already provided as byte array
                processedFile.File_Data = file.File_Data;
                processedFile.File_Size = file.File_Data.Length;
            }

            processedFiles.Add(processedFile);
        }
        catch (FormatException ex)
        {
            throw new ArgumentException($"Invalid base64 format for file: {file.File_Name}", ex);
        }
    }

    return processedFiles;
}

private string ExtractFileNameFromDataUrl(string dataUrl)
{
    // If no name can be extracted, generate a default one
    if (string.IsNullOrWhiteSpace(dataUrl))
        return $"file_{Guid.NewGuid()}.dat";

    string mimeType = ExtractMimeTypeFromDataUrl(dataUrl);
    string extension = GetFileExtensionFromMimeType(mimeType);
    
    return $"file_{DateTime.Now:yyyyMMddHHmmss}{extension}";
}

private string ExtractMimeTypeFromDataUrl(string dataUrl)
{
    if (string.IsNullOrWhiteSpace(dataUrl) || !dataUrl.StartsWith("data:"))
        return "application/octet-stream";

    try
    {
        // Extract MIME type from data URL (e.g., "data:image/png;base64,...")
        int colonIndex = dataUrl.IndexOf(":");
        int semicolonIndex = dataUrl.IndexOf(";");
        
        if (colonIndex > -1 && semicolonIndex > colonIndex)
        {
            return dataUrl.Substring(colonIndex + 1, semicolonIndex - colonIndex - 1);
        }
    }
    catch
    {
        // Fall back to default
    }

    return "application/octet-stream";
}

private string GetFileExtensionFromMimeType(string mimeType)
{
    var mimeTypeMap = new Dictionary<string, string>(StringComparer.OrdinalIgnoreCase)
    {
        // Images
        { "image/jpeg", ".jpg" },
        { "image/jpg", ".jpg" },
        { "image/png", ".png" },
        { "image/gif", ".gif" },
        { "image/bmp", ".bmp" },
        { "image/webp", ".webp" },
        { "image/svg+xml", ".svg" },
        
        // Documents
        { "application/pdf", ".pdf" },
        { "application/msword", ".doc" },
        { "application/vnd.openxmlformats-officedocument.wordprocessingml.document", ".docx" },
        { "application/vnd.ms-excel", ".xls" },
        { "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet", ".xlsx" },
        { "application/vnd.ms-powerpoint", ".ppt" },
        { "application/vnd.openxmlformats-officedocument.presentationml.presentation", ".pptx" },
        { "text/plain", ".txt" },
        { "text/csv", ".csv" },
        
        // Archives
        { "application/zip", ".zip" },
        { "application/x-rar-compressed", ".rar" },
        { "application/x-7z-compressed", ".7z" },
        
        // Other
        { "application/json", ".json" },
        { "application/xml", ".xml" },
        { "text/xml", ".xml" }
    };

    return mimeTypeMap.TryGetValue(mimeType, out string extension) 
        ? extension 
        : ".dat";
}

private DataTable GetAppFilesDt(List<App_Files> app_Files)
{
    var dt = new DataTable("TVP_Files");
    dt.Columns.Add("File_Name", typeof(string));
    dt.Columns.Add("File_Type", typeof(string));
    dt.Columns.Add("File_Size", typeof(long));
    dt.Columns.Add("File_Data", typeof(byte[]));

    if (app_Files != null && app_Files.Any())
    {
        foreach (var appFile in app_Files)
        {
            var dr = dt.NewRow();
            dr["File_Name"] = appFile.File_Name ?? (object)DBNull.Value;
            dr["File_Type"] = appFile.File_Type ?? (object)DBNull.Value;
            dr["File_Size"] = appFile.File_Size > 0 ? (object)appFile.File_Size : DBNull.Value;
            dr["File_Data"] = appFile.File_Data ?? (object)DBNull.Value;
            dt.Rows.Add(dr);
        }
    }

    return dt;
}

#endregion
