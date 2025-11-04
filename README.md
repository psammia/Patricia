using System.Configuration;
using System.Data;
using System.Net;
using Dapper;
using Newtonsoft.Json;
using QuestPDF;
using QuestPDF.Fluent;
using QuestPDF.Helpers;
using QuestPDF.Infrastructure;

namespace PDFGenerator.BLL
{
    public class BLL
    {
        private readonly string Catalog_Archive = ConfigurationManager.AppSettings["Catalog_Archive"] ?? "Alterna.Archive";

        #region RedownloadDocPDFForArchive - Enhanced with On-Demand Generation

        public byte[] RedownloadDocPDFForArchive(RedownloadDocPDFForArchiveRequest redownloadDocPDFForArchiveRequest)
        {
            byte[] retRes = [];

            try
            {
                using (DAL.DAL dal = new(Catalog_Archive, out var res))
                {
                    // Step 1: Try to get existing PDF binary
                    DynamicParameters dynamicParameters = new();
                    dynamicParameters.Add("BoxReference", redownloadDocPDFForArchiveRequest.ContainerID);

                    string command = ConfigurationManager.AppSettings["Get_PDF_Var_Binary_By_Box_Reference_SP"] ??
                                   "usp_GetPDFVarBinaryByBoxReference";
                    
                    byte[] pdfInDb = dal.ExecuteQuery<byte[]>(command, dynamicParameters, CommandType.StoredProcedure, DAL.CommandDirection.Read)
                        .DefaultIfEmpty([]).FirstOrDefault() ?? [];

                    // Step 2: If PDF binary exists and is valid, return it
                    if (pdfInDb.Length > 5)
                    {
                        return pdfInDb;
                    }

                    // Step 3: Try to get the original request from t_PDF
                    command = ConfigurationManager.AppSettings["Get_PDF_Request_By_Box_Reference_SP"] ??
                             "usp_GetPDFRequestByBoxReference";
                    
                    var requestData = dal.ExecuteQuery<dynamic>(command, dynamicParameters, CommandType.StoredProcedure, DAL.CommandDirection.Read)
                        .FirstOrDefault();

                    if (requestData != null && !string.IsNullOrEmpty(requestData.Request))
                    {
                        // PDF record exists but binary is empty - regenerate from stored request
                        string apiMethod = requestData.ApiMethod;
                        string originalRequestInJsonFormat = requestData.Request;

                        retRes = RegeneratePDFFromStoredRequest(originalRequestInJsonFormat, apiMethod);

                        // Update the PDF binary in database
                        if (retRes.Length > 0)
                        {
                            UpdatePDFBinaryInDatabase(retRes, originalRequestInJsonFormat, apiMethod, "System");
                        }

                        return retRes;
                    }

                    // Step 4: No PDF record exists - this is a legacy container
                    // Generate PDF from container data
                    retRes = GeneratePDFFromContainerData(redownloadDocPDFForArchiveRequest.ContainerID, dal);

                    return retRes;
                }
            }
            catch (Exception ex)
            {
                throw new Exception($"Failed to download/generate PDF for container {redownloadDocPDFForArchiveRequest.ContainerID}: {ex.Message}", ex);
            }
        }

        #endregion

        #region Generate PDF From Container Data (For Legacy Containers)

        private byte[] GeneratePDFFromContainerData(string containerCode, DAL.DAL dal)
        {
            try
            {
                // Get the user who sent the container
                string sentByUser = GetContainerSentByUser(containerCode, dal);

                // Get container data with files using stored procedure
                DynamicParameters param = new();
                param.Add("ContainerCode", containerCode);

                string command = ConfigurationManager.AppSettings["Get_Container_Data_For_PDF_Generation_SP"] ??
                               "usp_GetContainerDataForPDFGeneration";

                var containerData = dal.ExecuteQuery<ContainerFileData>(command, param, CommandType.StoredProcedure, DAL.CommandDirection.Read).ToList();

                if (!containerData.Any())
                {
                    throw new Exception($"No data found for container: {containerCode}");
                }

                var firstRow = containerData.First();
                string documentType = firstRow.DocumentType;
                string companyCode = firstRow.CompanyCode;
                DateTime? archivingDate = firstRow.ArchivingDate;
                int archivingPeriod = firstRow.ArchivingPeriod;

                // Calculate destruction date
                bool unlimited = archivingPeriod == -1;
                DateTime archivePeriodDate = (archivingDate ?? DateTime.Now).AddYears(archivingPeriod);
                string destructionDate = unlimited ? "Unlimited" : $"{archivePeriodDate:dd/MM/yyyy}";
                string creationDate = archivingDate.HasValue ? $"{archivingDate.Value:dd/MM/yyyy}" : $"{DateTime.Now:dd/MM/yyyy}";

                // Get entity
                string entity = GetEntityDescription(firstRow.Entity ?? companyCode, dal);

                // Generate PDF based on document type
                byte[] pdfBytes = [];

                switch (documentType)
                {
                    case "CUSTOMER":
                        pdfBytes = GenerateCustomerPDFFromContainerData(containerData, containerCode, entity, destructionDate, creationDate, sentByUser, dal);
                        break;

                    case "BRANCH":
                        pdfBytes = GenerateBranchPDFFromContainerData(containerData, containerCode, entity, destructionDate, creationDate, sentByUser);
                        break;

                    case "ENTITY":
                        pdfBytes = GenerateEntityPDFFromContainerData(containerData, containerCode, companyCode, destructionDate, creationDate, sentByUser);
                        break;

                    default:
                        throw new Exception($"Unknown document type: {documentType}");
                }

                return pdfBytes;
            }
            catch (Exception ex)
            {
                throw new Exception($"Failed to generate PDF from container data for {containerCode}: {ex.Message}", ex);
            }
        }

        private byte[] GenerateCustomerPDFFromContainerData(List<ContainerFileData> containerData, string containerCode, 
            string entity, string destructionDate, string creationDate, string user, DAL.DAL dal)
        {
            // Get customer files grouped by document type
            DynamicParameters param = new();
            param.Add("ContainerCode", containerCode);

            string command = ConfigurationManager.AppSettings["Get_Customer_Files_By_Container_SP"] ??
                           "usp_GetCustomerFilesByContainer";

            var customerFiles = dal.ExecuteQuery<dynamic>(command, param, CommandType.StoredProcedure, DAL.CommandDirection.Read).ToList();

            CustomerDocRequest customerDocRequest = new()
            {
                DestructionDate = destructionDate,
                ContainerID = containerCode,
                Entity = entity,
                User = user,
                CustomerFiles = new(),
                CreationDate = creationDate
            };

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

            foreach (var dictEntry in fileDict)
            {
                customerDocRequest.CustomerFiles.Add(new CustomerFile
                {
                    DocumentType = dictEntry.Key,
                    Id = dictEntry.Value
                });
            }

            var pdfBytes = GetByteArrayForCustomerDocPDFForArchive(customerDocRequest);
            
            // Save to database
            SavePDFToDatabase(pdfBytes, customerDocRequest, "GenerateCustomerDocPDFForArchive", entity, user);

            return pdfBytes;
        }

        private byte[] GenerateBranchPDFFromContainerData(List<ContainerFileData> containerData, string containerCode,
            string entity, string destructionDate, string creationDate, string user)
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

            foreach (var row in containerData)
            {
                branchDocRequest.BranchFiles.Add(new BranchFile
                {
                    DocumentType = row.FileName,
                    FromDate = row.FromDate.HasValue ? $"{row.FromDate.Value:dd-MM-yyyy}" : "",
                    ToDate = row.ToDate.HasValue ? $"{row.ToDate.Value:dd-MM-yyyy}" : ""
                });
            }

            var pdfBytes = GetByteArrayForBranchDocPDFForArchive(branchDocRequest);
            
            // Save to database
            SavePDFToDatabase(pdfBytes, branchDocRequest, "GenerateBranchDocPDFForArchive", entity, user);

            return pdfBytes;
        }

        private byte[] GenerateEntityPDFFromContainerData(List<ContainerFileData> containerData, string containerCode,
            string companyCode, string destructionDate, string creationDate, string user)
        {
            EntityDocRequest entityDocRequest = new()
            {
                DestructionDate = destructionDate,
                ContainerID = containerCode,
                Entity = companyCode,
                User = user,
                EntityFiles = new(),
                CreationDate = creationDate
            };

            foreach (var row in containerData)
            {
                entityDocRequest.EntityFiles.Add(new EntityFile
                {
                    DocumentType = row.FileName,
                    DocumentDescription = row.AdditionalInfo ?? string.Empty
                });
            }

            var pdfBytes = GetByteArrayForEntityDocPDFForArchive(entityDocRequest);
            
            // Save to database
            SavePDFToDatabase(pdfBytes, entityDocRequest, "GenerateEntityDocPDFForArchive", companyCode, user);

            return pdfBytes;
        }

        private string GetContainerSentByUser(string containerCode, DAL.DAL dal)
        {
            try
            {
                DynamicParameters param = new();
                param.Add("ContainerCode", containerCode);

                string command = ConfigurationManager.AppSettings["Get_Container_Sent_By_User_By_Code_SP"] ??
                               "usp_GetContainerSentByUserByCode";

                var result = dal.ExecuteQuery<dynamic>(command, param, CommandType.StoredProcedure, DAL.CommandDirection.Read)
                    .FirstOrDefault();

                return result?.SentByUser ?? "System";
            }
            catch
            {
                return "System";
            }
        }

        private string GetEntityDescription(string entityCode, DAL.DAL dal)
        {
            try
            {
                DynamicParameters param = new();
                param.Add("EntityCode", entityCode);

                string command = ConfigurationManager.AppSettings["Get_Entity_By_Code_SP"] ??
                               "usp_GetEntityByCode";

                var entity = dal.ExecuteQuery<dynamic>(command, param, CommandType.StoredProcedure, DAL.CommandDirection.Read)
                    .FirstOrDefault();

                return entity?.Description ?? entityCode;
            }
            catch
            {
                return entityCode;
            }
        }

        #endregion

        #region Regenerate PDF From Stored Request

        private byte[] RegeneratePDFFromStoredRequest(string requestJson, string apiMethod)
        {
            byte[] pdfBytes = [];

            switch (apiMethod)
            {
                case "GenerateEntityDocPDFForArchive":
                    var entityRequest = JsonConvert.DeserializeObject<EntityDocRequest>(requestJson);
                    if (entityRequest != null)
                    {
                        pdfBytes = GetByteArrayForEntityDocPDFForArchive(entityRequest);
                    }
                    break;

                case "GenerateBranchDocPDFForArchive":
                    var branchRequest = JsonConvert.DeserializeObject<BranchDocRequest>(requestJson);
                    if (branchRequest != null)
                    {
                        pdfBytes = GetByteArrayForBranchDocPDFForArchive(branchRequest);
                    }
                    break;

                case "GenerateCustomerDocPDFForArchive":
                    var customerRequest = JsonConvert.DeserializeObject<CustomerDocRequest>(requestJson);
                    if (customerRequest != null)
                    {
                        pdfBytes = GetByteArrayForCustomerDocPDFForArchive(customerRequest);
                    }
                    break;

                default:
                    throw new Exception($"Unknown API method: {apiMethod}");
            }

            return pdfBytes;
        }

        #endregion

        #region Save/Update PDF in Database

        private void SavePDFToDatabase(byte[] pdfBytes, object request, string apiMethod, string entity, string user)
        {
            try
            {
                string requestJson = JsonConvert.SerializeObject(request, Formatting.Indented);

                DynamicParameters dynamicParameters = new();
                dynamicParameters.Add("PDF", pdfBytes, DbType.Binary, ParameterDirection.Input);
                dynamicParameters.Add("Request", requestJson, DbType.String, ParameterDirection.Input);
                dynamicParameters.Add("ApiMethod", apiMethod, DbType.String, ParameterDirection.Input);
                dynamicParameters.Add("BranchList", "N/A", DbType.String, ParameterDirection.Input);
                dynamicParameters.Add("Entity", entity, DbType.String, ParameterDirection.Input);
                dynamicParameters.Add("User", user, DbType.String, ParameterDirection.Input);

                using (DAL.DAL dal = new(Catalog_Archive, out var res))
                {
                    var command = ConfigurationManager.AppSettings["Insert_PDF_SP"] ?? "usp_InsertPDF";
                    dal.ExecuteQuery(command, dynamicParameters, CommandType.StoredProcedure, DAL.CommandDirection.Update);
                }
            }
            catch (Exception ex)
            {
                // Log but don't throw - PDF was generated successfully
                Console.WriteLine($"Warning: Failed to save PDF to database: {ex.Message}");
            }
        }

        private void UpdatePDFBinaryInDatabase(byte[] pdfBytes, string requestJson, string apiMethod, string user)
        {
            try
            {
                // Extract entity and user from request if user is "System"
                dynamic requestObj = JsonConvert.DeserializeObject<dynamic>(requestJson);
                string entity = requestObj?.Entity ?? "Unknown";
                string requestUser = requestObj?.User ?? user;

                // If request has actual user, use that instead of "System"
                if (!string.IsNullOrEmpty(requestUser) && requestUser != "System")
                {
                    user = requestUser;
                }

                DynamicParameters dynamicParameters = new();
                dynamicParameters.Add("PDF", pdfBytes, DbType.Binary, ParameterDirection.Input);
                dynamicParameters.Add("Request", requestJson, DbType.String, ParameterDirection.Input);
                dynamicParameters.Add("ApiMethod", apiMethod, DbType.String, ParameterDirection.Input);
                dynamicParameters.Add("BranchList", "N/A", DbType.String, ParameterDirection.Input);
                dynamicParameters.Add("Entity", entity, DbType.String, ParameterDirection.Input);
                dynamicParameters.Add("User", user, DbType.String, ParameterDirection.Input);

                using (DAL.DAL dal = new(Catalog_Archive, out var res))
                {
                    var command = ConfigurationManager.AppSettings["Insert_PDF_SP"] ?? "usp_InsertPDF";
                    dal.ExecuteQuery(command, dynamicParameters, CommandType.StoredProcedure, DAL.CommandDirection.Update);
                }
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Warning: Failed to update PDF binary in database: {ex.Message}");
            }
        }

        #endregion

        #region Generate Entity Doc PDF - Original Method

        public byte[] GenerateEntityDocPDFForArchive(EntityDocRequest entityDocRequest)
        {
            var retRes = GetByteArrayForEntityDocPDFForArchive(entityDocRequest);

            // Save to database with empty binary initially (old behavior for compatibility)
            byte[] empty = [];

            DynamicParameters dynamicParameters = new();
            dynamicParameters.Add("PDF", empty, DbType.Binary, ParameterDirection.Input);
            dynamicParameters.Add("Request", JsonConvert.SerializeObject(entityDocRequest, Formatting.Indented),
                DbType.String, ParameterDirection.Input);
            dynamicParameters.Add("ApiMethod", "GenerateEntityDocPDFForArchive", DbType.String, ParameterDirection.Input);
            dynamicParameters.Add("BranchList", entityDocRequest.BranchList, DbType.String, ParameterDirection.Input);
            dynamicParameters.Add("Entity", entityDocRequest.Entity, DbType.String, ParameterDirection.Input);
            dynamicParameters.Add("User", entityDocRequest.User, DbType.String, ParameterDirection.Input);

            using (DAL.DAL dal = new(Catalog_Archive, out var res))
            {
                var command = ConfigurationManager.AppSettings["Insert_PDF_SP"] ?? "usp_InsertPDF";
                dal.ExecuteQuery(command, dynamicParameters, CommandType.StoredProcedure, DAL.CommandDirection.Update);
            }

            return retRes;
        }

        #endregion

        #region PDF Generation Methods (QuestPDF)

        private byte[] GetByteArrayForEntityDocPDFForArchive(EntityDocRequest entityDocRequest)
        {
            Settings.License = LicenseType.Community;
            var FontsFamily = ConfigurationManager.AppSettings["FONT_FAMILY"] ?? "Times New Roman";
            if (!float.TryParse(ConfigurationManager.AppSettings["FONT_SIZE"], out var FontSize)) FontSize = 14f;
            string[] FontFamilyList = FontsFamily.Split(',');

            var retRes = Document.Create(container =>
            {
                container.Page(page =>
                {
                    page.Size(PageSizes.A4);
                    page.Margin(15);
                    page.DefaultTextStyle(x => x.FontFamily(FontFamilyList).FontSize(FontSize));
                    
                    page.Header().Element(h =>
                    {
                        h.Table(t =>
                        {
                            t.ColumnsDefinition(col =>
                            {
                                col.RelativeColumn();
                                col.RelativeColumn();
                            });
                            t.Header(th =>
                            {
                                th.Cell().ColumnSpan(2).Element(HeadMid).Text("SUMMARY OF DELIVERY TO ARCHIVES")
                                    .SemiBold().FontSize(FontSize + 2);
                            });

                            t.Cell().Column(1).Row(2).Element(HeadLStart).Text($"Date: {entityDocRequest.CreationDate}");
                            t.Cell().Column(1).Row(3).Element(HeadL)
                                .Text($"Destruction Date: {entityDocRequest.DestructionDate}");
                            t.Cell().Column(1).Row(4).Element(HeadL).Text($"User: {entityDocRequest.User}");
                            t.Cell().Column(1).Row(5).Element(HeadLEnd).Text($"Entity: {entityDocRequest.Entity}");
                            t.Cell().Column(2).Row(2).RowSpan(4).Element(HeadSpan).Text($"{entityDocRequest.ContainerID}")
                                .FontSize(FontSize * 3f).FontColor(Colors.Grey.Darken2).Bold();
                        });
                    });
                    
                    page.Content().Column(x =>
                    {
                        x.Item().Table(table =>
                        {
                            table.ColumnsDefinition(columns =>
                            {
                                columns.RelativeColumn();
                                columns.RelativeColumn();
                            });
                            table.Header(header =>
                            {
                                header.Cell().Row(1).Column(1).Element(HeaderC).Text("Document type")
                                    .FontSize(FontSize + 2);
                                header.Cell().Row(1).Column(2).Element(HeaderC).Text("Document Description")
                                    .FontSize(FontSize + 2);
                            });

                            uint i = 1;
                            foreach (var entityFile in entityDocRequest.EntityFiles)
                            {
                                table.Cell().Row(i).Column(1).Element(DocumentType).Text(entityFile.DocumentType);
                                table.Cell().Row(i).Column(2).Element(BlockEntity).Text(entityFile.DocumentDescription);
                                i++;
                            }

                            table.Footer(footer =>
                            {
                                footer.Cell().ColumnSpan(2).Element(FooterR)
                                    .Text("Branch / Entity signature and seal");
                            });
                        });
                    });
                    
                    page.Footer().AlignCenter().Text(x =>
                    {
                        x.Span("Page ");
                        x.CurrentPageNumber();
                        x.Span(" Of ");
                        x.TotalPages();
                    });
                });
            }).GeneratePdf();

            return retRes;
        }

        private byte[] GetByteArrayForBranchDocPDFForArchive(BranchDocRequest branchDocRequest)
        {
            Settings.License = LicenseType.Community;
            var FontsFamily = ConfigurationManager.AppSettings["FONT_FAMILY"] ?? "Times New Roman";
            if (!float.TryParse(ConfigurationManager.AppSettings["FONT_SIZE"], out var FontSize)) FontSize = 14f;
            string[] FontFamilyList = FontsFamily.Split(',');

            var retRes = Document.Create(container =>
            {
                container.Page(page =>
                {
                    page.Size(PageSizes.A4);
                    page.Margin(15);
                    page.DefaultTextStyle(x => x.FontFamily(FontFamilyList).FontSize(FontSize));
                    
                    page.Header().Element(h =>
                    {
                        h.Table(t =>
                        {
                            t.ColumnsDefinition(col =>
                            {
                                col.RelativeColumn();
                                col.RelativeColumn();
                            });
                            t.Header(th =>
                            {
                                th.Cell().ColumnSpan(2).Element(HeadMid).Text("SUMMARY OF DELIVERY TO ARCHIVES")
                                    .SemiBold().FontSize(FontSize + 2);
                            });

                            t.Cell().Column(1).Row(2).Element(HeadLStart).Text($"Date: {branchDocRequest.CreationDate}");
                            t.Cell().Column(1).Row(3).Element(HeadL)
                                .Text($"Destruction Date: {branchDocRequest.DestructionDate}");
                            t.Cell().Column(1).Row(4).Element(HeadL).Text($"User: {branchDocRequest.User}");
                            t.Cell().Column(1).Row(5).Element(HeadLEnd).Text($"Entity: {branchDocRequest.Entity}");
                            t.Cell().Column(2).Row(2).RowSpan(4).Element(HeadSpan).Text($"{branchDocRequest.ContainerID}")
                                .FontSize(FontSize * 3f).FontColor(Colors.Grey.Darken2).Bold();
                        });
                    });
                    
                    page.Content().Column(x =>
                    {
                        x.Item().Table(table =>
                        {
                            table.ColumnsDefinition(columns =>
                            {
                                columns.RelativeColumn();
                                columns.RelativeColumn();
                                columns.RelativeColumn();
                            });
                            table.Header(header =>
                            {
                                header.Cell().Row(1).Column(1).Element(HeaderC).Text("Document type")
                                    .FontSize(FontSize + 2);
                                header.Cell().Row(1).Column(2).Element(HeaderC).Text("From Date")
                                    .FontSize(FontSize + 2);
                                header.Cell().Row(1).Column(3).Element(HeaderC).Text("To Date")
                                    .FontSize(FontSize + 2);
                            });

                            uint i = 1;
                            foreach (var branchFile in branchDocRequest.BranchFiles)
                            {
                                table.Cell().Row(i).Column(1).Element(DocumentType).Text(branchFile.DocumentType);
                                table.Cell().Row(i).Column(2).Element(BlockEntity).Text(branchFile.FromDate);
                                table.Cell().Row(i).Column(3).Element(BlockEntity).Text(branchFile.ToDate);
                                i++;
                            }

                            table.Footer(footer =>
                            {
                                footer.Cell().ColumnSpan(3).Element(FooterR)
                                    .Text("Branch / Entity signature and seal");
                            });
                        });
                    });
                    
                    page.Footer().AlignCenter().Text(x =>
                    {
                        x.Span("Page ");
                        x.CurrentPageNumber();
                        x.Span(" Of ");
                        x.TotalPages();
                    });
                });
            }).GeneratePdf();

            return retRes;
        }

        private byte[] GetByteArrayForCustomerDocPDFForArchive(CustomerDocRequest customerDocRequest)
        {
            Settings.License = LicenseType.Community;
            var FontsFamily = ConfigurationManager.AppSettings["FONT_FAMILY"] ?? "Times New Roman";
            if (!float.TryParse(ConfigurationManager.AppSettings["FONT_SIZE"], out var FontSize)) FontSize = 14f;
            string[] FontFamilyList = FontsFamily.Split(',');

            var retRes = Document.Create(container =>
            {
                container.Page(page =>
                {
                    page.Size(PageSizes.A4);
                    page.Margin(15);
                    page.DefaultTextStyle(x => x.FontFamily(FontFamilyList).FontSize(FontSize));
                    
                    page.Header().Element(h =>
                    {
                        h.Table(t =>
                        {
                            t.ColumnsDefinition(col =>
                            {
                                col.RelativeColumn();
                                col.RelativeColumn();
                            });
                            t.Header(th =>
                            {
                                th.Cell().ColumnSpan(2).Element(HeadMid).Text("SUMMARY OF DELIVERY TO ARCHIVES")
                                    .SemiBold().FontSize(FontSize + 2);
                            });

                            t.Cell().Column(1).Row(2).Element(HeadLStart).Text($"Date: {customerDocRequest.CreationDate}");
                            t.Cell().Column(1).Row(3).Element(HeadL)
                                .Text($"Destruction Date: {customerDocRequest.DestructionDate}");
                            t.Cell().Column(1).Row(4).Element(HeadL).Text($"User: {customerDocRequest.User}");
                            t.Cell().Column(1).Row(5).Element(HeadLEnd).Text($"Entity: {customerDocRequest.Entity}");
                            t.Cell().Column(2).Row(2).RowSpan(4).Element(HeadSpan).Text($"{customerDocRequest.ContainerID}")
                                .FontSize(FontSize * 3f).FontColor(Colors.Grey.Darken2).Bold();
                        });
                    });
                    
                    page.Content().Column(x =>
                    {
                        x.Item().Table(table =>
                        {
                            table.ColumnsDefinition(columns =>
                            {
                                columns.RelativeColumn();
                                columns.RelativeColumn(2);
                            });
                            table.Header(header =>
                            {
                                header.Cell().Row(1).Column(1).Element(HeaderC).Text("Document type")
                                    .FontSize(FontSize + 2);
                                header.Cell().Row(1).Column(2).Element(HeaderC).Text("Customer IDs")
                                    .FontSize(FontSize + 2);
                            });

                            uint i = 1;
                            foreach (var customerFile in customerDocRequest.CustomerFiles)
                            {
                                string customerIds = string.Join(", ", customerFile.Id);
                                table.Cell().Row(i).Column(1).Element(DocumentType).Text(customerFile.DocumentType);
                                table.Cell().Row(i).Column(2).Element(BlockEntity).Text(customerIds);
                                i++;
                            }

                            table.Footer(footer =>
                            {
                                footer.Cell().ColumnSpan(2).Element(FooterR)
                                    .Text("Branch / Entity signature and seal");
                            });
                        });
                    });
                    
                    page.Footer().AlignCenter().Text(x =>
                    {
                        x.Span("Page ");
                        x.CurrentPageNumber();
                        x.Span(" Of ");
                        x.TotalPages();
                    });
                });
            }).GeneratePdf();

            return retRes;
        }

        #endregion

        #region QuestPDF Styling Helper Methods

        static IContainer HeadMid(IContainer container)
        {
            return container
                .Border(1)
                .Background(Colors.Grey.Lighten3)
                .Padding(5)
                .AlignCenter()
                .AlignMiddle();
        }

        static IContainer HeadLStart(IContainer container)
        {
            return container
                .BorderLeft(1)
                .BorderRight(1)
                .BorderTop(1)
                .Padding(5);
        }

        static IContainer HeadL(IContainer container)
        {
            return container
                .BorderLeft(1)
                .BorderRight(1)
                .Padding(5);
        }

        static IContainer HeadLEnd(IContainer container)
        {
            return container
                .BorderLeft(1)
                .BorderRight(1)
                .BorderBottom(1)
                .Padding(5);
        }

        static IContainer HeadSpan(IContainer container)
        {
            return container
                .Border(1)
                .Padding(5)
                .AlignCenter()
                .AlignMiddle();
        }

        static IContainer HeaderC(IContainer container)
        {
            return container
                .Border(1)
                .Background(Colors.Grey.Lighten3)
                .Padding(5)
                .AlignCenter()
                .AlignMiddle();
        }

        static IContainer DocumentType(IContainer container)
        {
            return container
                .Border(1)
                .Padding(5)
                .AlignLeft()
                .AlignMiddle();
        }

        static IContainer BlockEntity(IContainer container)
        {
            return container
                .Border(1)
                .Padding(5)
                .AlignLeft()
                .AlignMiddle();
        }

        static IContainer FooterR(IContainer container)
        {
            return container
                .Border(1)
                .Padding(5)
                .AlignRight()
                .AlignMiddle();
        }

        #endregion

        #region Helper Classes

        public class ContainerFileData
        {
            public int ContainerId { get; set; }
            public string ContainerCode { get; set; }
            public string CompanyCode { get; set; }
            public string Entity { get; set; }
            public DateTime? ArchivingDate { get; set; }
            public string StatusCode { get; set; }
            public int FileId { get; set; }
            public string FileName { get; set; }
            public int? CustomerId { get; set; }
            public DateTime? FromDate { get; set; }
            public DateTime? ToDate { get; set; }
            public string AdditionalInfo { get; set; }
            public int ArchivingPeriod { get; set; }
            public string FileTypeDescription { get; set; }
            public string DocumentType { get; set; }
        }

        #endregion
    }
}
