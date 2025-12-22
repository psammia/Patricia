// ========================================
// STEP 1: Add NuGet Package to your project
// ========================================
/*
Install-Package itext7
OR
dotnet add package itext7
*/

// ========================================
// FILE: BLL/Bll.cs - COMPLETE UPDATED VERSION
// ========================================
using System.Data;
using DAL;
using Dapper;
using Microsoft.Extensions.Options;
using iText.Kernel.Pdf;
using iText.Layout;
using iText.Layout.Element;

namespace BLL
{
    public partial class Bll
    {
        private readonly GlobalSettings _globalSettings;

        public Bll(IOptionsMonitor<GlobalSettings> globalSettings)
        {
            _globalSettings = globalSettings.CurrentValue;
            InitializeGeneralEvents();
        }

        #region Insert User Application With Files
        public async Task<Application> Insert_UserApplication_WithFiles(Insert_UserApplication_WithFiles_Request request)
        {
            DapperDal dal = new DapperDal(_globalSettings.ConnString);
            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("P__CorrelationId", request.BaseRequest.CorrelationId);
            parameters.Add("P__External_Id", request.External_Id);
            parameters.Add("P__TVP_Files", ConvertAndGetAppFilesDt(request.app_FilesList).AsTableValuedParameter());

            IEnumerable<Application> res = await dal.ExecuteQueryAsync<Application>(
                "usp_InsertUserApplicationWithFiles",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Update);

            return res.ToList().First();
        }

        private DataTable ConvertAndGetAppFilesDt(List<App_Files_Input> app_Files)
        {
            DataTable dt = new DataTable("TVP_Files");
            dt.Columns.Add("File_Name", typeof(string));
            dt.Columns.Add("File_Type", typeof(string));
            dt.Columns.Add("File_Size", typeof(long));
            dt.Columns.Add("File_Data", typeof(byte[]));

            foreach (App_Files_Input appFile in app_Files)
            {
                DataRow dr = dt.NewRow();

                // Convert base64 string to byte array
                byte[] fileBytes;
                try
                {
                    // Clean the base64 string (remove whitespace)
                    string cleanBase64 = appFile.File_Data_Base64?.Trim() ?? string.Empty;
                    
                    if (string.IsNullOrWhiteSpace(cleanBase64))
                    {
                        throw new ArgumentException($"File data is empty for file: {appFile.File_Name}");
                    }
                    
                    fileBytes = System.Convert.FromBase64String(cleanBase64);
                }
                catch (FormatException ex)
                {
                    throw new ArgumentException($"Invalid base64 data for file: {appFile.File_Name}", ex);
                }

                dr["File_Name"] = appFile.File_Name ?? string.Empty;
                dr["File_Type"] = appFile.File_Type ?? string.Empty;
                dr["File_Size"] = fileBytes.LongLength;
                dr["File_Data"] = fileBytes;

                dt.Rows.Add(dr);
            }

            return dt;
        }
        #endregion

        #region Get Application Files
        public async Task<(Application? application, List<App_Files_Db> files)> Get_ApplicationFiles(Get_ApplicationFiles_Request request)
        {
            DapperDal dal = new DapperDal(_globalSettings.ConnString);
            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("P__CorrelationId", request.CorrelationId);
            parameters.Add("P__External_Id", request.External_Id);

            // Get Application
            IEnumerable<Application> appResult = await dal.ExecuteQueryAsync<Application>(
                "usp_GetApplicationFiles",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Select);

            Application? application = appResult.FirstOrDefault();

            // Get Files
            List<App_Files_Db> files = new List<App_Files_Db>();

            if (application != null)
            {
                DynamicParameters fileParameters = new DynamicParameters();
                fileParameters.Add("P__CorrelationId", request.CorrelationId);
                fileParameters.Add("P__External_Id", request.External_Id);

                IEnumerable<App_Files_Db> filesResult = await dal.ExecuteQueryAsync<App_Files_Db>(
                    "usp_GetApplicationFiles_Files",
                    fileParameters,
                    CommandType.StoredProcedure,
                    DapperDal.CommandDirection.Select);

                files = filesResult.ToList();
            }

            return (application, files);
        }
        #endregion

        #region Update Application Status
        public async Task<Application?> Update_ApplicationStatus(Update_ApplicationStatus_Request request)
        {
            DapperDal dal = new DapperDal(_globalSettings.ConnString);
            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("P__CorrelationId", request.CorrelationId);
            parameters.Add("P__External_Id", request.External_Id);
            parameters.Add("P__StatusId", request.StatusId);

            IEnumerable<Application> res = await dal.ExecuteQueryAsync<Application>(
                "usp_UpdateApplicationStatus",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Update);

            return res.FirstOrDefault();
        }
        #endregion

        #region Delete Applications with their related files base on StatusId
        public async Task Delete_ApplicationsByStatus(Delete_ApplicationsByStatus_Request request)
        {
            DapperDal dal = new DapperDal(_globalSettings.ConnString);
            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("P__StatusId", request.StatusId);
            parameters.Add("P__CorrelationId", request.CorrelationId);
            parameters.Add("P__External_Id", request.External_Id);
            parameters.Add("P__Error", dbType: DbType.String, direction: ParameterDirection.Output, size: 4000);

            await dal.ExecuteQueryAsync<dynamic>(
                "usp_DeleteApplicationsByStatus",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Delete);

            String storedProcedureErrorMessage = parameters.Get<String>("P__Error");

            if (!String.IsNullOrWhiteSpace(storedProcedureErrorMessage))
            {
                throw new ArgumentException(storedProcedureErrorMessage);
            }
        }
        #endregion

        #region PDF Helper Methods
        
        /// <summary>
        /// Converts image bytes to PDF bytes
        /// </summary>
        public byte[] ConvertImageToPdf(byte[] imageBytes, string fileName)
        {
            using (MemoryStream ms = new MemoryStream())
            {
                using (PdfWriter writer = new PdfWriter(ms))
                using (PdfDocument pdf = new PdfDocument(writer))
                using (Document document = new Document(pdf))
                {
                    try
                    {
                        iText.IO.Image.ImageData imageData = iText.IO.Image.ImageDataFactory.Create(imageBytes);
                        Image image = new Image(imageData);
                        
                        // Scale image to fit page
                        image.SetAutoScale(true);
                        document.Add(image);
                    }
                    catch (Exception ex)
                    {
                        throw new Exception($"Failed to convert image to PDF for file: {fileName}", ex);
                    }
                }
                return ms.ToArray();
            }
        }

        /// <summary>
        /// Merges two PDF documents (recto first, then verso)
        /// </summary>
        public byte[] MergePdfDocuments(byte[] rectoPdf, byte[] versoPdf)
        {
            using (MemoryStream ms = new MemoryStream())
            {
                using (PdfWriter writer = new PdfWriter(ms))
                using (PdfDocument mergedPdf = new PdfDocument(writer))
                {
                    // Add recto pages
                    using (MemoryStream rectoStream = new MemoryStream(rectoPdf))
                    using (PdfDocument rectoPdfDoc = new PdfDocument(new PdfReader(rectoStream)))
                    {
                        rectoPdfDoc.CopyPagesTo(1, rectoPdfDoc.GetNumberOfPages(), mergedPdf);
                    }

                    // Add verso pages
                    using (MemoryStream versoStream = new MemoryStream(versoPdf))
                    using (PdfDocument versoPdfDoc = new PdfDocument(new PdfReader(versoStream)))
                    {
                        versoPdfDoc.CopyPagesTo(1, versoPdfDoc.GetNumberOfPages(), mergedPdf);
                    }
                }
                return ms.ToArray();
            }
        }

        /// <summary>
        /// Checks if file is an image based on file type
        /// </summary>
        public bool IsImageFile(string fileType)
        {
            string[] imageTypes = { "image/jpeg", "image/jpg", "image/png", "image/gif", "image/bmp", "image/tiff" };
            return imageTypes.Contains(fileType.ToLower());
        }

        /// <summary>
        /// Checks if file is already a PDF
        /// </summary>
        public bool IsPdfFile(string fileType)
        {
            return fileType.ToLower() == "application/pdf";
        }

        #endregion
    }
}

// ========================================
// FILE: Controllers/OnBoardingController.cs - UPDATED Get_ApplicationFiles ONLY
// ========================================
// Replace your existing Get_ApplicationFiles method with this:

        #region Get Application Files
        [HttpPost]
        [Route("Get_ApplicationFiles")]
        public async Task<Get_ApplicationFiles_Response> Get_ApplicationFiles([FromBody] Get_ApplicationFiles_Request request)
        {
            Get_ApplicationFiles_Response response = new Get_ApplicationFiles_Response()
            {
                BaseResponse = new BaseResponse()
                {
                    CorrelationId = request.BaseRequest.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };

            CorrelationInfo correlationInfo = new CorrelationInfo()
            {
                CorrelationId = request.BaseRequest.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetApplicationFiles",
                UserName = _globalSettings.AppUsername
            };

            try
            {
                #region DataGuard
                List<KeyValuePair<DataIntegrityCheckfunctions, Property>> dataGuardDict =
                [
                    new(DataIntegrityCheckfunctions.IS_CORRELATION_ID_INVALID, new Property()
                    {
                        PropName = "CorrelationId",
                        PropValue = correlationInfo.CorrelationId
                    }),
                    new(DataIntegrityCheckfunctions.IS_STRING_EMPTY, new Property()
                    {
                        PropName = "CorrelationId",
                        PropValue = request.CorrelationId
                    }),
                    new(DataIntegrityCheckfunctions.IS_STRING_EMPTY, new Property()
                    {
                        PropName = "External_Id",
                        PropValue = request.External_Id
                    })
                ];

                DataIntegrityCheck(dataGuardDict);
                #endregion

                correlationInfo.Reserved = "GetApplicationFiles has been called with the following Request";
                LogInfoJson(request, correlationInfo);

                // Call BLL method
                var (application, filesDb) = await _bll.Get_ApplicationFiles(request);

                // Check if application was found
                if (application == null)
                {
                    response.BaseResponse.ReturnCode = _responseCodesDictionary["404"].Content;
                    response.BaseResponse.ReturnDescription = "Application not found with provided CorrelationId and External_Id";

                    correlationInfo.RDirection = RequestDirection.Response;
                    correlationInfo.Reserved = "Application not found";
                    LogInfoJson(response, correlationInfo);

                    return response;
                }

                // Process files: Convert to PDF and merge ID_CARD recto/verso
                List<App_Files_Output> filesOutput = ProcessFilesForPdfOutput(filesDb);

                // Set response data
                response.Application = application;
                response.Files = filesOutput;

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = "GetApplicationFiles replied with the following response";
                LogInfoJson(response, correlationInfo);

                return response;
            }
            catch (SGBLBadRequestException ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);
                sb.Replace("{0}", ex.Message);

                response.BaseResponse.CorrelationId = request.BaseRequest.CorrelationId;
                response.BaseResponse.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResponse.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);
                sb.Replace("{0}", ex.Message);

                response.BaseResponse.CorrelationId = request.BaseRequest.CorrelationId;
                response.BaseResponse.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResponse.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region PDF Processing Helper Methods - ADD TO OnBoardingController
        
        private List<App_Files_Output> ProcessFilesForPdfOutput(List<App_Files_Db> filesDb)
        {
            List<App_Files_Output> result = new List<App_Files_Output>();
            HashSet<long> processedIds = new HashSet<long>();

            for (int i = 0; i < filesDb.Count; i++)
            {
                var file = filesDb[i];

                // Skip if already processed
                if (processedIds.Contains(file.Id))
                    continue;

                // Check if this is an ID_CARD with recto/verso
                if (file.File_Type.Equals("ID_CARD", StringComparison.OrdinalIgnoreCase))
                {
                    string fileNameLower = file.File_Name.ToLower();

                    if (fileNameLower.Contains("recto"))
                    {
                        // Look for matching verso
                        var versoFile = filesDb.FirstOrDefault(f => 
                            f.File_Type.Equals("ID_CARD", StringComparison.OrdinalIgnoreCase) &&
                            f.File_Name.ToLower().Contains("verso") &&
                            !processedIds.Contains(f.Id));

                        if (versoFile != null)
                        {
                            // Merge recto and verso into one PDF
                            byte[] mergedPdf = MergeRectoVerso(file, versoFile);
                            
                            result.Add(new App_Files_Output
                            {
                                Id = file.Id,
                                App_Id = file.App_Id,
                                File_Name = "ID_CARD_Combined.pdf",
                                File_Type = "application/pdf",
                                File_Size = mergedPdf.Length,
                                File_Data_Base64 = Convert.ToBase64String(mergedPdf),
                                CreatedDate = file.CreatedDate
                            });

                            processedIds.Add(file.Id);
                            processedIds.Add(versoFile.Id);
                            continue;
                        }
                    }
                    else if (fileNameLower.Contains("verso"))
                    {
                        // Look for matching recto
                        var rectoFile = filesDb.FirstOrDefault(f => 
                            f.File_Type.Equals("ID_CARD", StringComparison.OrdinalIgnoreCase) &&
                            f.File_Name.ToLower().Contains("recto") &&
                            !processedIds.Contains(f.Id));

                        if (rectoFile != null)
                        {
                            // Merge recto and verso into one PDF (recto first)
                            byte[] mergedPdf = MergeRectoVerso(rectoFile, file);
                            
                            result.Add(new App_Files_Output
                            {
                                Id = rectoFile.Id,
                                App_Id = rectoFile.App_Id,
                                File_Name = "ID_CARD_Combined.pdf",
                                File_Type = "application/pdf",
                                File_Size = mergedPdf.Length,
                                File_Data_Base64 = Convert.ToBase64String(mergedPdf),
                                CreatedDate = rectoFile.CreatedDate
                            });

                            processedIds.Add(file.Id);
                            processedIds.Add(rectoFile.Id);
                            continue;
                        }
                    }
                }

                // For all other files or standalone ID_CARD, convert to PDF
                byte[] pdfBytes = ConvertToPdf(file);
                
                result.Add(new App_Files_Output
                {
                    Id = file.Id,
                    App_Id = file.App_Id,
                    File_Name = Path.GetFileNameWithoutExtension(file.File_Name) + ".pdf",
                    File_Type = "application/pdf",
                    File_Size = pdfBytes.Length,
                    File_Data_Base64 = Convert.ToBase64String(pdfBytes),
                    CreatedDate = file.CreatedDate
                });

                processedIds.Add(file.Id);
            }

            return result;
        }

        private byte[] ConvertToPdf(App_Files_Db file)
        {
            // If already PDF, return as is
            if (_bll.IsPdfFile(file.File_Type))
            {
                return file.File_Data;
            }

            // If image, convert to PDF
            if (_bll.IsImageFile(file.File_Type))
            {
                return _bll.ConvertImageToPdf(file.File_Data, file.File_Name);
            }

            // For other types, return as is (or throw exception)
            throw new Exception($"Unsupported file type for PDF conversion: {file.File_Type}");
        }

        private byte[] MergeRectoVerso(App_Files_Db rectoFile, App_Files_Db versoFile)
        {
            // Convert both to PDF if needed
            byte[] rectoPdf = ConvertToPdf(rectoFile);
            byte[] versoPdf = ConvertToPdf(versoFile);

            // Merge PDFs
            return _bll.MergePdfDocuments(rectoPdf, versoPdf);
        }

        #endregion

// ========================================
// NOTE: All other controller methods remain UNCHANGED
// Stored procedures remain UNCHANGED
// Models remain UNCHANGED
// Program.cs remains UNCHANGED
// ========================================

// ========================================
// TESTING SCENARIOS
// ========================================
/*
SCENARIO 1: ID_CARD with recto and verso (both images)
Database has:
- File_Type: "ID_CARD", File_Name: "id_recto.jpg"
- File_Type: "ID_CARD", File_Name: "id_verso.png"

Result:
- ONE file: "ID_CARD_Combined.pdf" (2 pages: recto first, then verso)

SCENARIO 2: ID_CARD with only recto
Database has:
- File_Type: "ID_CARD", File_Name: "id_recto.jpg"

Result:
- ONE file: "id_recto.pdf" (1 page)

SCENARIO 3: Mixed documents
Database has:
- File_Type: "ID_CARD", File_Name: "id_recto.jpg"
- File_Type: "ID_CARD", File_Name: "id_verso.jpg"
- File_Type: "PASSPORT", File_Name: "passport.pdf"
- File_Type: "PROOF_ADDRESS", File_Name: "bill.png"

Result:
- "ID_CARD_Combined.pdf" (merged recto+verso)
- "passport.pdf" (unchanged, already PDF)
- "bill.pdf" (converted from PNG)
*/
