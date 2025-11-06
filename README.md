// Add this method to your BLL.cs in the Archiving project

public class BackfillResult
{
    public int TotalContainers { get; set; }
    public int SuccessCount { get; set; }
    public int FailureCount { get; set; }
    public List<string> FailedContainers { get; set; } = new();
}

public class BackfillContainerInfo
{
    public int Id { get; set; }
    public string Code { get; set; }
    public string CompanyCode { get; set; }
    public string Entity { get; set; }
    public string StatusCode { get; set; }
    public DateTime? ArchivingDate { get; set; }
    public DateTime CreatedDate { get; set; }
    public string CurrentLocation { get; set; }
}

public class BackfillFileInfo
{
    public int FileId { get; set; }
    public string FileName { get; set; }
    public int? CustomerId { get; set; }
    public DateTime? FromDate { get; set; }
    public DateTime? ToDate { get; set; }
    public string AdditionalInfo { get; set; }
    public string CompanyCode { get; set; }
    public int ArchivingPeriod { get; set; }
    public string FileTypeDescription { get; set; }
}

public BackfillResult BackfillMissingPDFsForLegacyContainers(string currentUser, DateTime? fromDate = null, DateTime? toDate = null)
{
    BackfillResult result = new();
    
    try
    {
        using (DAL.DAL iDAL = new())
        {
            // Get all containers without PDF using stored procedure
            DynamicParameters param = new();
            param.Add("FromDate", fromDate, DbType.DateTime, ParameterDirection.Input);
            param.Add("ToDate", toDate, DbType.DateTime, ParameterDirection.Input);

            string command = ConfigurationManager.AppSettings["Get_Containers_Without_PDF_SP"] ??
                           "usp_GetContainersWithoutPDF";

            var containersWithoutPDF = iDAL.ExecuteQuery<BackfillContainerInfo>(
                command, 
                param, 
                CommandType.StoredProcedure, 
                CommandDirection.Read
            );

            result.TotalContainers = containersWithoutPDF.Count();

            foreach (var container in containersWithoutPDF)
            {
                try
                {
                    // Get the actual user who sent the container using stored procedure
                    string sentByUser = GetContainerSentByUser(container.Id, iDAL);

                    // Get container files using stored procedure
                    DynamicParameters fileParam = new();
                    fileParam.Add("ContainerId", container.Id);

                    string fileCommand = ConfigurationManager.AppSettings["Get_Container_Files_For_Backfill_SP"] ??
                                       "usp_GetContainerFilesForBackfill";

                    var files = iDAL.ExecuteQuery<BackfillFileInfo>(
                        fileCommand,
                        fileParam,
                        CommandType.StoredProcedure,
                        CommandDirection.Read
                    ).ToList();

                    if (files.Count > 0)
                    {
                        var firstFile = files.First();
                        
                        bool unlimited = firstFile.ArchivingPeriod == -1;
                        DateTime archivePeriod = container.ArchivingDate ?? DateTime.Now;
                        archivePeriod = archivePeriod.AddYears(firstFile.ArchivingPeriod);

                        string entity = GetActiveEntity(container.Entity);
                        string destructionDate = unlimited ? "Unlimited" : $"{archivePeriod:dd/MM/yyyy}";
                        string creationDate = container.ArchivingDate.HasValue 
                            ? $"{container.ArchivingDate.Value:dd/MM/yyyy}" 
                            : $"{DateTime.Now:dd/MM/yyyy}";

                        // Determine document type and generate PDF
                        if (firstFile.CustomerId != null)
                        {
                            GenerateCustomerPDFForLegacyContainer(files, container.Code, entity, sentByUser, destructionDate, creationDate, iDAL);
                        }
                        else if (firstFile.CompanyCode.StartsWith("LB"))
                        {
                            GenerateBranchPDFForLegacyContainer(files, container.Code, entity, sentByUser, destructionDate, creationDate);
                        }
                        else if (firstFile.CompanyCode.StartsWith("ET"))
                        {
                            GenerateEntityPDFForLegacyContainer(files, container.Code, firstFile.CompanyCode, sentByUser, destructionDate, creationDate);
                        }

                        result.SuccessCount++;
                    }
                    else
                    {
                        result.FailureCount++;
                        result.FailedContainers.Add($"{container.Code}: No files found");
                    }
                }
                catch (Exception ex)
                {
                    result.FailureCount++;
                    result.FailedContainers.Add($"{container.Code}: {ex.Message}");
                    // Log the error
                    LogError($"Failed to generate PDF for container {container.Code}", new CorrelationInfo(), ex);
                }
            }
        }
    }
    catch (Exception ex)
    {
        throw new SGBLInternalServerException("Backfill process failed", ex);
    }

    return result;
}

private string GetContainerSentByUser(int containerId, DAL.DAL iDAL)
{
    try
    {
        DynamicParameters param = new();
        param.Add("ContainerId", containerId);

        string command = ConfigurationManager.AppSettings["Get_Container_Sent_By_User_SP"] ??
                        "usp_GetContainerSentByUser";

        var result = iDAL.ExecuteQuery<dynamic>(
            command, 
            param, 
            CommandType.StoredProcedure, 
            CommandDirection.Read
        ).FirstOrDefault();

        return result?.SentByUser ?? "System";
    }
    catch
    {
        return "System";
    }
}

private void GenerateEntityPDFForLegacyContainer(List<BackfillFileInfo> files, string containerCode, 
    string entity, string user, string destructionDate, string creationDate)
{
    EntityDocRequest entityDocRequest = new()
    {
        DestructionDate = destructionDate,
        ContainerID = containerCode,
        Entity = entity,
        User = user,
        EntityFiles = new(),
        CreationDate = creationDate
    };

    foreach (BackfillFileInfo item in files)
    {
        entityDocRequest.EntityFiles.Add(new()
        {
            DocumentType = item.FileName,
            DocumentDescription = item.AdditionalInfo ?? string.Empty
        });
    }

    CallPDFServiceAndStore(entityDocRequest, "GenerateEntityDocPDFForArchive");
}

private void GenerateBranchPDFForLegacyContainer(List<BackfillFileInfo> files, string containerCode, 
    string entity, string user, string destructionDate, string creationDate)
{
    BranchDocRequest branchDocRequest = new()
    {
        DestructionDate = destructionDate,
        ContainerID = containerCode,
        Entity = entity,
        User = user,
        BranchFiles = new(),
        CreationDate = creationDate
    };

    foreach (BackfillFileInfo item in files)
    {
        branchDocRequest.BranchFiles.Add(new()
        {
            DocumentType = item.FileName,
            FromDate = item.FromDate.HasValue ? $"{item.FromDate.Value:dd-MM-yyyy}" : "",
            ToDate = item.ToDate.HasValue ? $"{item.ToDate.Value:dd-MM-yyyy}" : ""
        });
    }

    CallPDFServiceAndStore(branchDocRequest, "GenerateBranchDocPDFForArchive");
}

private void GenerateCustomerPDFForLegacyContainer(List<BackfillFileInfo> files, string containerCode, 
    string entity, string user, string destructionDate, string creationDate, DAL.DAL iDAL)
{
    CustomerDocRequest customerDocRequest = new()
    {
        DestructionDate = destructionDate,
        ContainerID = containerCode,
        Entity = entity,
        User = user,
        CustomerFiles = new(),
        CreationDate = creationDate
    };

    // Get customer files grouped by document type using stored procedure
    DynamicParameters param = new();
    param.Add("ContainerCode", containerCode);

    string command = ConfigurationManager.AppSettings["Get_Customer_Files_By_Container_SP"] ??
                   "usp_GetCustomerFilesByContainer";

    var customerFiles = iDAL.ExecuteQuery<dynamic>(
        command, 
        param, 
        CommandType.StoredProcedure, 
        CommandDirection.Read
    ).ToList();

    // Group customer IDs by document type
    Dictionary<string, List<string>> fileDict = new();
    foreach (var file in customerFiles)
    {
        string docType = file.DocumentType;
        string customerId = file.CustomerIdString ?? "";

        if (!fileDict.ContainsKey(docType))
        {
            fileDict.Add(docType, new List<string> { customerId });
        }
        else if (!fileDict[docType].Contains(customerId))
        {
            fileDict[docType].Add(customerId);
        }
    }

    foreach (KeyValuePair<string, List<string>> dictEntry in fileDict)
    {
        customerDocRequest.CustomerFiles.Add(new() 
        { 
            DocumentType = dictEntry.Key, 
            Id = dictEntry.Value
        });
    }

    CallPDFServiceAndStore(customerDocRequest, "GenerateCustomerDocPDFForArchive");
}

private void CallPDFServiceAndStore(object request, string apiMethod)
{
    try
    {
        string data = JsonConvert.SerializeObject(request);
        HttpContent content = new StringContent(data, Encoding.UTF8, "application/json");
        HttpClient client = new();
        string pdfRequestBase = ConfigurationManager.AppSettings["PDFService"] 
            ?? throw new SGBLInternalServerException("PDF Service not initialized please Contact Support");

        Task<HttpResponseMessage> httpRequest = client.PostAsync($"{pdfRequestBase}{apiMethod}", content);
        httpRequest.Wait();
        
        Task<string> responseString = httpRequest.Result.Content.ReadAsStringAsync();
        responseString.Wait();

        if (string.IsNullOrEmpty(responseString.Result))
        {
            throw new SGBLInternalServerException("PDF Service returned empty response");
        }
    }
    catch (Exception ex)
    {
        throw new SGBLInternalServerException($"PDF Creation Failed for {apiMethod}", ex);
    }
}
