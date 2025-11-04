// Add this method to your BLL.cs in the Archiving project

public class BackfillResult
{
    public int TotalContainers { get; set; }
    public int SuccessCount { get; set; }
    public int FailureCount { get; set; }
    public List<string> FailedContainers { get; set; } = new();
}

public BackfillResult BackfillMissingPDFsForLegacyContainers(string currentUser)
{
    BackfillResult result = new();
    
    try
    {
        // Get all containers in SENT, RECEIVED, TOBEDESTR, DESTROYED status without PDF
        string query = @"
            SELECT DISTINCT c.Id, c.Code, c.CompanyCode, c.Entity, c.StatusCode, c.ArchivingDate
            FROM t_Container c
            WHERE c.StatusCode IN ('SENT', 'RECEIVED', 'TOBEDESTR', 'DESTROYED')
            AND c.isDeleted = 0
            AND NOT EXISTS (
                SELECT 1 
                FROM t_PDF p 
                WHERE p.Request LIKE '%' + c.Code + '%'
            )";

        using (DAL.DAL iDAL = new())
        {
            var containersWithoutPDF = iDAL.ExecuteQuery<Container>(query, null, CommandType.Text, CommandDirection.Read);
            result.TotalContainers = containersWithoutPDF.Count();

            foreach (var container in containersWithoutPDF)
            {
                try
                {
                    // Get the actual user who sent the container
                    string sentByUser = GetContainerSentByUser(container.Id, iDAL);

                    // Get container files
                    GetContainerFilesReq getContainerFilesReq = new()
                    {
                        BaseReq = new BaseRequest { CurrentUser = sentByUser },
                        ContainerId = container.Id
                    };

                    List<ArchivedFile> files = GetContainerFiles(getContainerFilesReq).Files;

                    if (files.Count > 0)
                    {
                        bool unlimited = false;
                        DateTime archivePeriod = container.ArchivingDate ?? DateTime.Now;
                        archivePeriod = archivePeriod.AddYears(files[0].ArchivingPeriod);

                        if (files[0].ArchivingPeriod == -1)
                        {
                            unlimited = true;
                        }

                        string entity = GetActiveEntity(container.Entity);
                        string destructionDate = unlimited ? "Unlimited" : $"{archivePeriod:dd/MM/yyyy}";
                        string creationDate = container.ArchivingDate.HasValue 
                            ? $"{container.ArchivingDate.Value:dd/MM/yyyy}" 
                            : $"{DateTime.Now:dd/MM/yyyy}";

                        // Determine document type and generate PDF
                        if (files[0].CustomerId != null)
                        {
                            GenerateCustomerPDFForLegacyContainer(files, container.Code, entity, sentByUser, destructionDate, creationDate);
                        }
                        else if (files[0].CompanyCode.StartsWith("LB"))
                        {
                            GenerateBranchPDFForLegacyContainer(files, container.Code, entity, sentByUser, destructionDate, creationDate);
                        }
                        else if (files[0].CompanyCode.StartsWith("ET"))
                        {
                            GenerateEntityPDFForLegacyContainer(files, container.Code, files[0].CompanyCode, sentByUser, destructionDate, creationDate);
                        }

                        result.SuccessCount++;
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

        var result = iDAL.ExecuteQuery<dynamic>(command, param, CommandType.StoredProcedure, CommandDirection.Read)
            .FirstOrDefault();

        return result?.SentByUser ?? "System";
    }
    catch
    {
        return "System";
    }
}

private void GenerateEntityPDFForLegacyContainer(List<ArchivedFile> files, string containerCode, 
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

    foreach (ArchivedFile item in files)
    {
        entityDocRequest.EntityFiles.Add(new()
        {
            DocumentType = item.Name,
            DocumentDescription = item.AdditionalInfo ?? string.Empty
        });
    }

    CallPDFServiceAndStore(entityDocRequest, "GenerateEntityDocPDFForArchive");
}

private void GenerateBranchPDFForLegacyContainer(List<ArchivedFile> files, string containerCode, 
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

    foreach (ArchivedFile item in files)
    {
        branchDocRequest.BranchFiles.Add(new()
        {
            DocumentType = item.Name,
            FromDate = $"{item.FromDate:dd-MM-yyyy}",
            ToDate = $"{item.ToDate:dd-MM-yyyy}"
        });
    }

    CallPDFServiceAndStore(branchDocRequest, "GenerateBranchDocPDFForArchive");
}

private void GenerateCustomerPDFForLegacyContainer(List<ArchivedFile> files, string containerCode, 
    string entity, string user, string destructionDate, string creationDate)
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

    Dictionary<string, List<string?>> fileDict = new();
    foreach (ArchivedFile item in files)
    {
        if (!fileDict.ContainsKey(item.Name))
        {
            fileDict.Add(item.Name, new List<string?> { item.CustomerId!.ToString() });
        }
        else
        {
            fileDict[item.Name].Add(item.CustomerId!.ToString());
        }
    }

    foreach (KeyValuePair<string, List<string?>> dictEntry in fileDict)
    {
        customerDocRequest.CustomerFiles.Add(new() 
        { 
            DocumentType = dictEntry.Key, 
            Id = dictEntry.Value! 
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
