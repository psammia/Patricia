// Replace the existing RedownloadDocPDFForArchive method in PDFGenerator BLL.cs

public byte[] RedownloadDocPDFForArchive(RedownloadDocPDFForArchiveRequest redownloadDocPDFForArchiveRequest)
{
    byte[] retRes = [];
    byte[] pdfInDb = [];

    DynamicParameters dynamicParameters = new();
    dynamicParameters.Add("BoxReference", redownloadDocPDFForArchiveRequest.ContainerID);

    var originalRequestInJsonFormat = string.Empty;

    using (DAL.DAL dal = new(Catalog_Archive, out var res))
    {
        var command = "";
        switch (redownloadDocPDFForArchiveRequest.DocumentType)
        {
            case ArchivingDocumentType.ENTITY_PDF:
                command = ConfigurationManager.AppSettings["Get_PDF_Var_Binary_By_Box_Reference_SP"] ??
                          "usp_GetPDFVarBinaryByBoxReference";
                pdfInDb = dal.ExecuteQuery<byte[]>(command, dynamicParameters).DefaultIfEmpty([]).First();

                if (pdfInDb.Length > 5)
                {
                    retRes = pdfInDb;
                }
                else
                {
                    // Try to get the original request from t_PDF
                    command = ConfigurationManager.AppSettings["Get_PDF_Request_By_Box_Reference_SP"] ??
                              "usp_GetPDFRequestByBoxReference";
                    originalRequestInJsonFormat = dal.ExecuteQuery<string>(command, dynamicParameters)
                        .DefaultIfEmpty(string.Empty).First();

                    if (!string.IsNullOrEmpty(originalRequestInJsonFormat))
                    {
                        // PDF exists but varbinary is empty, regenerate from request
                        retRes = GetByteArrayForEntityDocPDFForArchive(
                            JsonConvert.DeserializeObject<EntityDocRequest>(originalRequestInJsonFormat)!);
                    }
                    else
                    {
                        // PDF doesn't exist at all - this is a legacy container
                        // Generate PDF from container data
                        retRes = GeneratePDFFromContainerData(redownloadDocPDFForArchiveRequest.ContainerID, dal);
                    }
                }

                break;
                
            case ArchivingDocumentType.BRANCH_PDF:
                // Similar logic for branch documents
                command = ConfigurationManager.AppSettings["Get_PDF_Var_Binary_By_Box_Reference_SP"] ??
                          "usp_GetPDFVarBinaryByBoxReference";
                pdfInDb = dal.ExecuteQuery<byte[]>(command, dynamicParameters).DefaultIfEmpty([]).First();

                if (pdfInDb.Length > 5)
                {
                    retRes = pdfInDb;
                }
                else
                {
                    command = ConfigurationManager.AppSettings["Get_PDF_Request_By_Box_Reference_SP"] ??
                              "usp_GetPDFRequestByBoxReference";
                    originalRequestInJsonFormat = dal.ExecuteQuery<string>(command, dynamicParameters)
                        .DefaultIfEmpty(string.Empty).First();

                    if (!string.IsNullOrEmpty(originalRequestInJsonFormat))
                    {
                        retRes = GetByteArrayForBranchDocPDFForArchive(
                            JsonConvert.DeserializeObject<BranchDocRequest>(originalRequestInJsonFormat)!);
                    }
                    else
                    {
                        retRes = GeneratePDFFromContainerData(redownloadDocPDFForArchiveRequest.ContainerID, dal);
                    }
                }
                break;

            case ArchivingDocumentType.CUSTOMER_PDF:
                // Similar logic for customer documents
                command = ConfigurationManager.AppSettings["Get_PDF_Var_Binary_By_Box_Reference_SP"] ??
                          "usp_GetPDFVarBinaryByBoxReference";
                pdfInDb = dal.ExecuteQuery<byte[]>(command, dynamicParameters).DefaultIfEmpty([]).First();

                if (pdfInDb.Length > 5)
                {
                    retRes = pdfInDb;
                }
                else
                {
                    command = ConfigurationManager.AppSettings["Get_PDF_Request_By_Box_Reference_SP"] ??
                              "usp_GetPDFRequestByBoxReference";
                    originalRequestInJsonFormat = dal.ExecuteQuery<string>(command, dynamicParameters)
                        .DefaultIfEmpty(string.Empty).First();

                    if (!string.IsNullOrEmpty(originalRequestInJsonFormat))
                    {
                        retRes = GetByteArrayForCustomerDocPDFForArchive(
                            JsonConvert.DeserializeObject<CustomerDocRequest>(originalRequestInJsonFormat)!);
                    }
                    else
                    {
                        retRes = GeneratePDFFromContainerData(redownloadDocPDFForArchiveRequest.ContainerID, dal);
                    }
                }
                break;
        }
    }

    return retRes;
}

private byte[] GeneratePDFFromContainerData(string containerCode, DAL.DAL dal)
{
    try
    {
        // Query to get container and its files
        string query = @"
            SELECT 
                c.Id, c.Code, c.CompanyCode, c.Entity, c.ArchivingDate,
                f.Id as FileId, f.Name, f.CustomerId, f.FromDate, f.ToDate, 
                f.AdditionalInfo, ft.ArchivingPeriod
            FROM t_Container c
            INNER JOIN t_CurrentContainerFileRelationship ccfr ON c.Id = ccfr.ContainerId
            INNER JOIN t_File f ON ccfr.FileId = f.Id
            INNER JOIN lkp_FileType ft ON f.FileTypeCode = ft.Code
            WHERE c.Code = @ContainerCode AND c.isDeleted = 0 AND f.isDeleted = 0";

        DynamicParameters param = new();
        param.Add("ContainerCode", containerCode);

        var containerData = dal.ExecuteQuery<dynamic>(query, param, CommandType.Text, CommandDirection.Read).ToList();

        if (!containerData.Any())
        {
            throw new Exception($"No data found for container: {containerCode}");
        }

        var firstRow = containerData.First();
        string companyCode = firstRow.CompanyCode;
        DateTime? archivingDate = firstRow.ArchivingDate;
        int archivingPeriod = firstRow.ArchivingPeriod;

        bool unlimited = archivingPeriod == -1;
        DateTime archivePeriodDate = (archivingDate ?? DateTime.Now).AddYears(archivingPeriod);
        string destructionDate = unlimited ? "Unlimited" : $"{archivePeriodDate:dd/MM/yyyy}";
        string creationDate = archivingDate.HasValue ? $"{archivingDate.Value:dd/MM/yyyy}" : $"{DateTime.Now:dd/MM/yyyy}";

        // Determine entity
        string entity = firstRow.Entity ?? companyCode;
        
        // Check document type and generate appropriate PDF
        if (firstRow.CustomerId != null)
        {
            // Customer document
            CustomerDocRequest customerDocRequest = new()
            {
                DestructionDate = destructionDate,
                ContainerID = containerCode,
                Entity = entity,
                User = "System",
                CustomerFiles = new(),
                CreationDate = creationDate
            };

            Dictionary<string, List<string>> fileDict = new();
            foreach (var row in containerData)
            {
                string docType = row.Name;
                string customerId = row.CustomerId?.ToString() ?? "";
                
                if (!fileDict.ContainsKey(docType))
                {
                    fileDict.Add(docType, new List<string> { customerId });
                }
                else
                {
                    fileDict[docType].Add(customerId);
                }
            }

            foreach (var dictEntry in fileDict)
            {
                customerDocRequest.CustomerFiles.Add(new() 
                { 
                    DocumentType = dictEntry.Key, 
                    Id = dictEntry.Value 
                });
            }

            var pdfBytes = GetByteArrayForCustomerDocPDFForArchive(customerDocRequest);
            SavePDFToDatabase(pdfBytes, customerDocRequest, "GenerateCustomerDocPDFForArchive", entity, "System");
            return pdfBytes;
        }
        else if (companyCode.StartsWith("LB"))
        {
            // Branch document
            BranchDocRequest branchDocRequest = new()
            {
                DestructionDate = destructionDate,
                ContainerID = containerCode,
                Entity = entity,
                User = "System",
                BranchFiles = new(),
                CreationDate = creationDate
            };

            foreach (var row in containerData)
            {
                branchDocRequest.BranchFiles.Add(new()
                {
                    DocumentType = row.Name,
                    FromDate = $"{((DateTime)row.FromDate):dd-MM-yyyy}",
                    ToDate = $"{((DateTime)row.ToDate):dd-MM-yyyy}"
                });
            }

            var pdfBytes = GetByteArrayForBranchDocPDFForArchive(branchDocRequest);
            SavePDFToDatabase(pdfBytes, branchDocRequest, "GenerateBranchDocPDFForArchive", entity, "System");
            return pdfBytes;
        }
        else if (companyCode.StartsWith("ET"))
        {
            // Entity document
            EntityDocRequest entityDocRequest = new()
            {
                DestructionDate = destructionDate,
                ContainerID = containerCode,
                Entity = companyCode,
                User = "System",
                EntityFiles = new(),
                CreationDate = creationDate
            };

            foreach (var row in containerData)
            {
                entityDocRequest.EntityFiles.Add(new()
                {
                    DocumentType = row.Name,
                    DocumentDescription = row.AdditionalInfo ?? string.Empty
                });
            }

            var pdfBytes = GetByteArrayForEntityDocPDFForArchive(entityDocRequest);
            SavePDFToDatabase(pdfBytes, entityDocRequest, "GenerateEntityDocPDFForArchive", entity, "System");
            return pdfBytes;
        }

        throw new Exception($"Unable to determine document type for container: {containerCode}");
    }
    catch (Exception ex)
    {
        throw new Exception($"Failed to generate PDF from container data: {ex.Message}", ex);
    }
}

private void SavePDFToDatabase(byte[] pdfBytes, object request, string apiMethod, string entity, string user)
{
    try
    {
        DynamicParameters dynamicParameters = new();
        dynamicParameters.Add("PDF", pdfBytes, DbType.Binary, ParameterDirection.Input);
        dynamicParameters.Add("Request", JsonConvert.SerializeObject(request, Formatting.Indented),
            DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("ApiMethod", apiMethod, DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("BranchList", "N/A", DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("Entity", entity, DbType.String, ParameterDirection.Input);
        dynamicParameters.Add("User", user, DbType.String, ParameterDirection.Input);

        using (DAL.DAL dal = new(Catalog_Archive, out var res))
        {
            var command = ConfigurationManager.AppSettings["Insert_PDF_SP"] ?? "usp_InsertPDF";
            dal.ExecuteQuery(command, dynamicParameters);
        }
    }
    catch (Exception ex)
    {
        // Log but don't fail - PDF was generated successfully
        // Logging code here
    }
}
