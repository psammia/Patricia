using System;
using System.IO;
using System.Text;
using Newtonsoft.Json; // or System.Text.Json

// ====================================
// Example 1: Manual Object Creation
// ====================================
public Insert_UserApplication_WithFiles_Request CreateRequestManually()
{
    var request = new Insert_UserApplication_WithFiles_Request
    {
        BaseRequest = new BaseRequest
        {
            CorrelationId = Guid.NewGuid().ToString()
        },
        CorrelationId = Guid.NewGuid().ToString(),
        External_Id = "EXT-2024-001",
        app_FilesList = new List<App_Files>
        {
            new App_Files
            {
                File_Name = "invoice.pdf",
                File_Type = "application/pdf",
                File_Size = 0, // Will be calculated in BLL
                File_Data_Base64 = "JVBERi0xLjQKJeLjz9MKMyAwIG9iago8PC9UeXBlL1BhZ2UvUGFyZW50IDIgMCBSL01lZGlhQm94WzAgMCA2MTIgNzkyXS9Db250ZW50cyA0IDAgUj4+CmVuZG9iago0IDAgb2JqCjw8L0ZpbHRlci9GbGF0ZURlY29kZS9MZW5ndGggNDQ+PnN0cmVhbQp4nCvkMlAwULCx0XfOL0hNLlFQV0jMK0stKlFQBPFBIhYKPpm5qUUgHSAlphYKQFZqJVDKwQAAR3IQJgplbmRzdHJlYW0KZW5kb2JqCjEgMCBvYmoKPDwvVHlwZS9QYWdlcy9Db3VudCAxL0tpZHNbMyAwIFJdPj4KZW5kb2JqCjUgMCBvYmoKPDwvVHlwZS9DYXRhbG9nL1BhZ2VzIDEgMCBSPj4KZW5kb2JqCjYgMCBvYmoKPDwvUHJvZHVjZXIoaVRleHQm"
            },
            new App_Files
            {
                File_Name = "photo.jpg",
                File_Type = "image/jpeg",
                File_Size = 0,
                File_Data_Base64 = "/9j/4AAQSkZJRgABAQEAYABgAAD/2wBDAAgGBgcGBQgHBwcJCQgKDBQNDAsLDBkSEw8UHRofHh0aHBwgJC4nICIsIxwcKDcpLDAxNDQ0Hyc5PTgyPC4zNDL/wAALCAABAAEBAREA/8QAFAABAAAAAAAAAAAAAAAAAAAAA//EABQQAQAAAAAAAAAAAAAAAAAAAAD/2gAIAQEAAT8AKp//2Q=="
            }
        }
    };

    return request;
}

// ====================================
// Example 2: Load File from Disk
// ====================================
public Insert_UserApplication_WithFiles_Request CreateRequestFromFile(string filePath)
{
    // Read file from disk
    byte[] fileBytes = File.ReadAllBytes(filePath);
    string base64String = Convert.ToBase64String(fileBytes);
    
    // Get file info
    FileInfo fileInfo = new FileInfo(filePath);
    string fileName = fileInfo.Name;
    string mimeType = GetMimeType(fileName);

    var request = new Insert_UserApplication_WithFiles_Request
    {
        BaseRequest = new BaseRequest
        {
            CorrelationId = Guid.NewGuid().ToString()
        },
        CorrelationId = Guid.NewGuid().ToString(),
        External_Id = $"EXT-{DateTime.Now:yyyyMMddHHmmss}",
        app_FilesList = new List<App_Files>
        {
            new App_Files
            {
                File_Name = fileName,
                File_Type = mimeType,
                File_Size = fileBytes.Length,
                File_Data_Base64 = base64String
            }
        }
    };

    return request;
}

// ====================================
// Example 3: Multiple Files from Disk
// ====================================
public Insert_UserApplication_WithFiles_Request CreateRequestFromMultipleFiles(string[] filePaths)
{
    var filesList = new List<App_Files>();

    foreach (var filePath in filePaths)
    {
        if (File.Exists(filePath))
        {
            byte[] fileBytes = File.ReadAllBytes(filePath);
            FileInfo fileInfo = new FileInfo(filePath);

            filesList.Add(new App_Files
            {
                File_Name = fileInfo.Name,
                File_Type = GetMimeType(fileInfo.Name),
                File_Size = fileBytes.Length,
                File_Data_Base64 = Convert.ToBase64String(fileBytes)
            });
        }
    }

    var request = new Insert_UserApplication_WithFiles_Request
    {
        BaseRequest = new BaseRequest
        {
            CorrelationId = Guid.NewGuid().ToString()
        },
        CorrelationId = Guid.NewGuid().ToString(),
        External_Id = $"EXT-{DateTime.Now:yyyyMMddHHmmss}",
        app_FilesList = filesList
    };

    return request;
}

// ====================================
// Example 4: From Stream (e.g., Upload)
// ====================================
public Insert_UserApplication_WithFiles_Request CreateRequestFromStream(
    Stream fileStream, 
    string fileName, 
    string externalId)
{
    using (var memoryStream = new MemoryStream())
    {
        fileStream.CopyTo(memoryStream);
        byte[] fileBytes = memoryStream.ToArray();
        string base64String = Convert.ToBase64String(fileBytes);

        var request = new Insert_UserApplication_WithFiles_Request
        {
            BaseRequest = new BaseRequest
            {
                CorrelationId = Guid.NewGuid().ToString()
            },
            CorrelationId = Guid.NewGuid().ToString(),
            External_Id = externalId,
            app_FilesList = new List<App_Files>
            {
                new App_Files
                {
                    File_Name = fileName,
                    File_Type = GetMimeType(fileName),
                    File_Size = fileBytes.Length,
                    File_Data_Base64 = base64String
                }
            }
        };

        return request;
    }
}

// ====================================
// Example 5: From Web API Controller
// ====================================
[HttpPost("upload")]
public async Task<IActionResult> UploadFiles([FromForm] IFormFileCollection files, [FromForm] string externalId)
{
    var filesList = new List<App_Files>();

    foreach (var file in files)
    {
        if (file.Length > 0)
        {
            using (var memoryStream = new MemoryStream())
            {
                await file.CopyToAsync(memoryStream);
                byte[] fileBytes = memoryStream.ToArray();

                filesList.Add(new App_Files
                {
                    File_Name = file.FileName,
                    File_Type = file.ContentType,
                    File_Size = file.Length,
                    File_Data_Base64 = Convert.ToBase64String(fileBytes)
                });
            }
        }
    }

    var request = new Insert_UserApplication_WithFiles_Request
    {
        BaseRequest = new BaseRequest
        {
            CorrelationId = Guid.NewGuid().ToString()
        },
        CorrelationId = Guid.NewGuid().ToString(),
        External_Id = externalId,
        app_FilesList = filesList
    };

    // Call your BLL method
    var result = await _businessLayer.Insert_UserApplication_WithFiles(request);
    
    return Ok(result);
}

// ====================================
// Example 6: Serialize to JSON
// ====================================
public string CreateJsonRequest()
{
    var request = new Insert_UserApplication_WithFiles_Request
    {
        BaseRequest = new BaseRequest
        {
            CorrelationId = "550e8400-e29b-41d4-a716-446655440000"
        },
        CorrelationId = "550e8400-e29b-41d4-a716-446655440001",
        External_Id = "EXT-2024-12345",
        app_FilesList = new List<App_Files>
        {
            new App_Files
            {
                File_Name = "contract.pdf",
                File_Type = "application/pdf",
                File_Size = 2048,
                File_Data_Base64 = "JVBERi0xLjQKJeLjz9MK..."
            }
        }
    };

    // Using Newtonsoft.Json
    string json = JsonConvert.SerializeObject(request, Formatting.Indented);
    
    // Or using System.Text.Json
    // string json = JsonSerializer.Serialize(request, new JsonSerializerOptions { WriteIndented = true });
    
    return json;
}

// ====================================
// Example 7: Deserialize from JSON
// ====================================
public Insert_UserApplication_WithFiles_Request DeserializeJsonRequest(string json)
{
    // Using Newtonsoft.Json
    var request = JsonConvert.DeserializeObject<Insert_UserApplication_WithFiles_Request>(json);
    
    // Or using System.Text.Json
    // var request = JsonSerializer.Deserialize<Insert_UserApplication_WithFiles_Request>(json);
    
    return request;
}

// ====================================
// Helper Method: Get MIME Type
// ====================================
private string GetMimeType(string fileName)
{
    string extension = Path.GetExtension(fileName).ToLowerInvariant();
    
    var mimeTypes = new Dictionary<string, string>
    {
        { ".pdf", "application/pdf" },
        { ".doc", "application/msword" },
        { ".docx", "application/vnd.openxmlformats-officedocument.wordprocessingml.document" },
        { ".xls", "application/vnd.ms-excel" },
        { ".xlsx", "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet" },
        { ".png", "image/png" },
        { ".jpg", "image/jpeg" },
        { ".jpeg", "image/jpeg" },
        { ".gif", "image/gif" },
        { ".txt", "text/plain" },
        { ".csv", "text/csv" },
        { ".zip", "application/zip" }
    };

    return mimeTypes.ContainsKey(extension) 
        ? mimeTypes[extension] 
        : "application/octet-stream";
}

// ====================================
// Example 8: Sample JSON String
// ====================================
public string GetSampleJsonString()
{
    return @"{
  ""baseRequest"": {
    ""correlationId"": ""550e8400-e29b-41d4-a716-446655440000""
  },
  ""correlationId"": ""550e8400-e29b-41d4-a716-446655440001"",
  ""external_Id"": ""EXT-2024-12345"",
  ""app_FilesList"": [
    {
      ""file_Name"": ""invoice.pdf"",
      ""file_Type"": ""application/pdf"",
      ""file_Size"": 0,
      ""file_Data"": null,
      ""file_Data_Base64"": ""JVBERi0xLjQKJeLjz9MKMyAwIG9iago8PC9UeXBlL1BhZ2UvUGFyZW50IDIgMCBSL01lZGlhQm94WzAgMCA2MTIgNzkyXS9Db250ZW50cyA0IDAgUj4+CmVuZG9iago0IDAgb2JqCjw8L0ZpbHRlci9GbGF0ZURlY29kZS9MZW5ndGggNDQ+PnN0cmVhbQp4nCvkMlAwULCx0XfOL0hNLlFQV0jMK0stKlFQBPFBIhYKPpm5qUUgHSAlphYKQFZqJVDKwQAAR3IQJgplbmRzdHJlYW0KZW5kb2JqCjEgMCBvYmoKPDwvVHlwZS9QYWdlcy9Db3VudCAxL0tpZHNbMyAwIFJdPj4KZW5kb2JqCjUgMCBvYmoKPDwvVHlwZS9DYXRhbG9nL1BhZ2VzIDEgMCBSPj4KZW5kb2JqCjYgMCBvYmoKPDwvUHJvZHVjZXIoaVRleHQm""
    },
    {
      ""file_Name"": ""photo.jpg"",
      ""file_Type"": ""image/jpeg"",
      ""file_Size"": 0,
      ""file_Data"": null,
      ""file_Data_Base64"": ""/9j/4AAQSkZJRgABAQEAYABgAAD/2wBDAAgGBgcGBQgHBwcJCQgKDBQNDAsLDBkSEw8UHRofHh0aHBwgJC4nICIsIxwcKDcpLDAxNDQ0Hyc5PTgyPC4zNDL/wAALCAABAAEBAREA/8QAFAABAAAAAAAAAAAAAAAAAAAAA//EABQQAQAAAAAAAAAAAAAAAAAAAAD/2gAIAQEAAT8AKp//2Q==""
    }
  ]
}";
}

// ====================================
// Example 9: Usage in Console App
// ====================================
public static async Task Main(string[] args)
{
    // Example 1: From file
    string filePath = @"C:\Documents\invoice.pdf";
    var request1 = CreateRequestFromFile(filePath);
    
    // Example 2: Multiple files
    string[] files = new[] 
    { 
        @"C:\Documents\invoice.pdf",
        @"C:\Documents\contract.docx",
        @"C:\Images\photo.jpg"
    };
    var request2 = CreateRequestFromMultipleFiles(files);
    
    // Example 3: Manual creation
    var request3 = CreateRequestManually();
    
    // Send to API or BLL
    // var result = await businessLayer.Insert_UserApplication_WithFiles(request1);
}


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
