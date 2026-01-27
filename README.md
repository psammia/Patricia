using Azure.Core;
using Common;
using Common.Constants;
using Common.CustomExceptions;
using Common.Dto;
using Common.Model;
using Common.Request;
using Common.Response;
using DAL;
using Dapper;
using System.Data;
using System.Net;
using System.Reflection.Metadata.Ecma335;
using ClosedXML.Excel;
using System.IO;

namespace BAL
{
    public partial class Bal
    {
        #region ProcessAlfaAttachement

        public async Task ProcessAlfaAttachment(ProcessAlfaAttachmentRequest request)
        {
            string dataExportValidateFileUrl = $"{_globalSettings.DataExportUrl}/ValidateFile";
            string archiveDate = $"{DateTime.Today:yyyyMMdd}";
            string archivePath = Path.Combine(_globalSettings.ArchivePath, archiveDate);

            if (request.Attachment == null || request.Attachment.Length == 0)
            {
                throw new SGBLBadRequestException("Invalid attachment");
            }

            //get the file configurations
            Common.Model.File fileConfig = await GetFileConfigurationByCode(nameof(FileConfigCodes.AlfaConfig));

            //
            Common.Dto.FileInfo dataExportFileInfo = fileConfig.ToFileInfoDto();

            //calculate checksum
            string checkSum = await Utils.GetFileChecksumAsync(request.Attachment);

            //check if file checksum already exists
            await CheckFileCheckSum(checkSum);

            //archive the imported file
            Utils.UploadFile(archivePath, request.Attachment);

            //read data from excel using data export
            byte[] fileBytes = await Utils.ToByteArrayAsync(request.Attachment);

            string hex = Utils.ByteArrayToHexString(fileBytes);

            dataExportFileInfo.FileBinary = hex;

            DataExportValidateFileRequest dataExportValidateFileRequest = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                FileInfo = dataExportFileInfo
            };

            DataExportValidateFileResponse responseData =
                await PostAsync<DataExportValidateFileResponse>(dataExportValidateFileRequest,
                    dataExportValidateFileUrl);

            if (responseData is not { WebResp.StatusCode: HttpStatusCode.OK })
            {
                throw new SGBLBadGateWayException(string.Join(Environment.NewLine, responseData.WebResp.Errors));
            }

            List<AlfaClient> alfaClients = GetAlfaClientsFromParsedData(responseData.ParsedDataList, fileConfig!);

            //insert alfa clients to database
            await InsertAlfaClients(
                fileConfig.Id,
                request.CurrencyCode,
                request.Attachment.FileName,
                checkSum,
                request.Cycle,
                archiveDate,
                alfaClients,
                request.BaseReq.UserName);
        }

        #endregion

        #region DownloadArchivedFile

        public async Task<(byte[] fileBytes, string contentType, string fileName)> DownloadArchivedFile(
            DownloadArchivedFileRequest request)
        {
            DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

            DynamicParameters parameters = new DynamicParameters();
            parameters.Add("P__AttachmentId", request.AttachmentId);

            IEnumerable<Common.Model.Attachment> attachmentResult = await dal.ExecuteQueryAsync<Attachment>(
                "usp_Get_Attachment_By_Id",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Select);

            Common.Model.Attachment? result = attachmentResult.FirstOrDefault();

            if (result == null)
            {
                throw new SGBLBadRequestException($"The requested attachment was not found");
            }

            string archivePath = Path.Combine(_globalSettings.ArchivePath, result.Directory);

            return Utils.DownloadFile(archivePath, result.Name);
        }

        #endregion

        #region GetAllFileImport

        public async Task<List<FileImport>> GetAllFileImport(GetAllFileImportRequest request)
        {
            DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

            IEnumerable<FileImport> response = await dal.ExecuteQueryAsync<FileImport>(
                "usp_Get_All_File_Import",
                null,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Select);

            return response.ToList();
        }

        #endregion

        #region GetFileImportContentByFileImportId

        public async Task<List<AlfaFileImportContent>> GetFileImportContentByFileImportId(
            GetFileImportContentByFileImportIdRequest request)
        {
            DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("@P__Id", request.FileImportId);
            parameters.Add("@P__Error", dbType: DbType.String, direction: ParameterDirection.Output, size: 4000);

            IEnumerable<AlfaFileImportContent> response = await dal.ExecuteQueryAsync<AlfaFileImportContent>(
                "usp_Get_File_Import_Content_By_File_Import_Id",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Select);

            string storedProcedureErrorMessage = parameters.Get<string>("@P__Error");

            if (!string.IsNullOrEmpty(storedProcedureErrorMessage))
            {
                throw new SGBLBadRequestException(storedProcedureErrorMessage);
            }

            return response.ToList();
        }

        #endregion

        #region GetLookupsByTableName

        public async Task<Dictionary<string, List<Lookup>>> GetLookupsByTableName(GetLookupsByTableNameRequest request)
        {
            DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);
            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("@P__TableNames", request.TableNames);

            IEnumerable<Lookup> response = await dal.ExecuteQueryAsync<Lookup>(
                "usp_Get_Lookups_By_TableNames",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Select
            );

            return response.GroupBy(x => x.TableName).ToDictionary(g => g.Key, g => g.ToList());
        }

        #endregion

        #region UpdateFileImportContentMsisdn

        public async Task UpdateFileImportContentMsisdn(UpdateFileImportContentMsisdnRequest request)
        {
            DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("@P__Id", request.Id);
            parameters.Add("@P__MsisdnPrimaryContact", request.MsisdnPrimaryContact.Trim());
            parameters.Add("@P__User", request.BaseReq.UserName);
            parameters.Add("@P__Error", dbType: DbType.String, direction: ParameterDirection.Output, size: 4000);

            _ = await dal.ExecuteQueryAsync<dynamic>(
                "usp_Update_File_Import_Content_Alfa_Msisdn",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Update);

            string storedProcedureErrorMessage = parameters.Get<string>("@P__Error");

            if (!string.IsNullOrWhiteSpace(storedProcedureErrorMessage))
            {
                throw new SGBLBadRequestException(storedProcedureErrorMessage);
            }
        }

        #endregion

        #region ExportFileImportContentToExcel

        public async Task<(byte[] fileBytes, string fileName)> ExportFileImportContentToExcel(
            ExportFileImportContentToExcelRequest request)
        {
            DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("@P__FileImportId", request.FileImportId);
            parameters.Add("@P__Error", dbType: DbType.String, direction: ParameterDirection.Output, size: 4000);

            IEnumerable<AlfaClientExport> data = await dal.ExecuteQueryAsync<AlfaClientExport>(
                "usp_Get_File_Import_Content_Alfa_For_Export",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Select);

            string storedProcedureErrorMessage = parameters.Get<string>("@P__Error");

            if (!string.IsNullOrWhiteSpace(storedProcedureErrorMessage))
            {
                throw new SGBLBadRequestException(storedProcedureErrorMessage);
            }

            List<AlfaClientExport> dataList = data.ToList();

            if (dataList.Count == 0)
            {
                throw new SGBLBadRequestException("No data found for the specified File Import.");
            }

            byte[] excelBytes = GenerateAlfaExcelFile(dataList);

            string fileName = $"FileImport_{request.FileImportId}_{DateTime.Now:yyyyMMdd_HHmmss}.xlsx";

            return (excelBytes, fileName);
        }

        #endregion

        #region GenerateAlfaExcelFile

        private byte[] GenerateAlfaExcelFile(List<AlfaClientExport> data)
        {
            using (var workbook = new XLWorkbook())
            {
                var worksheet = workbook.Worksheets.Add("Alfa Data");

                var headers = new Dictionary<int, string>
                {
                    { 1, "BANK CODE" },
                    { 2, "BANK NAME" },
                    { 3, "BANK BRANCH" },
                    { 4, "BANK ACCOUNT NUMBER" },
                    { 5, "CUSTOMER NAME" },
                    { 6, "Primary Account Number" },
                    { 7, "MSISDN (one)" },
                    { 8, "ACCOUNT BALANCE" },
                    { 9, "INVOICE DATE" },
                    { 10, "PAID AMOUNT" },
                    { 11, "SAYRAFA RATE" }
                };

                foreach (var header in headers)
                {
                    var cell = worksheet.Cell(1, header.Key);
                    cell.Value = header.Value;
                }

                int row = 2;
                foreach (var item in data)
                {
                    worksheet.Cell(row, 1).Value = item.BankCode;
                    worksheet.Cell(row, 2).Value = item.BankName;
                    worksheet.Cell(row, 3).Value = item.BankBranch;
                    worksheet.Cell(row, 4).Value = item.BankAccountNumber;
                    worksheet.Cell(row, 5).Value = item.CustomerName;
                    worksheet.Cell(row, 6).Value = item.PrimaryAccountNumber;
                    worksheet.Cell(row, 7).Value = item.MsisdnPrimaryContact;
                    worksheet.Cell(row, 8).Value = item.AccountBalance;
                    worksheet.Cell(row, 9).Value = item.InvoiceDate.ToString("dd/MM/yyyy");
                    worksheet.Cell(row, 10).Value = item.AmountPaid ?? 0;
                    worksheet.Cell(row, 11).Value = item.SayrafaRate ?? 0;

                    row++;
                }

                worksheet.Columns().AdjustToContents();

                using (var stream = new MemoryStream())
                {
                    workbook.SaveAs(stream);
                    return stream.ToArray();
                }
            }
        }

        #endregion

        #region GetFileImportByWhere

        public async Task<dynamic> GetFileImportByWhere(GetFileImportByWhereRequest request)
        {
            DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

            DynamicParameters parameters = new DynamicParameters();

            // Pass null if not provided, stored procedure will handle defaults
            parameters.Add("@P__FromDate", request.FromDate);
            parameters.Add("@P__ToDate", request.ToDate);
            parameters.Add("@P__StatusCode",
                string.IsNullOrWhiteSpace(request.StatusCode) ? null : request.StatusCode.Trim());
            parameters.Add("@P__Error", dbType: DbType.String, direction: ParameterDirection.Output, size: 4000);

            IEnumerable<FileImport> response = await dal.ExecuteQueryAsync<FileImport>(
                "usp_Get_File_Import_By_Where",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Select);

            string storedProcedureErrorMessage = parameters.Get<string>("@P__Error");

            if (!string.IsNullOrEmpty(storedProcedureErrorMessage))
            {
                throw new SGBLBadRequestException(storedProcedureErrorMessage);
            }

            return response.ToList();
        }

        #endregion

        #region UpdateFileImportDetails

        public async Task UpdateFileImportDetails(UpdateFileImportDetailsRequest request)
        {
            DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);
            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("@P__Id", request.Id);
            parameters.Add("@P__StatusCode", request.StatusCode);
            parameters.Add("@P__T24FileCheckSum", request.T24FileCheckSum);
            parameters.Add("@P__User", request.T24FileCheckSum);
            parameters.Add("@P__Error", dbType: DbType.String, direction: ParameterDirection.Output, size: 4000);

            _ = await dal.ExecuteQueryAsync<dynamic>(
                "usp_Update_File_Import_Details",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Update);

            string storedProcedureErrorMessage = parameters.Get<string>("@P__Error");
            if (!string.IsNullOrEmpty(storedProcedureErrorMessage))
            {
                throw new SGBLBadRequestException(storedProcedureErrorMessage);
            }
        }

        #endregion

        #region GenerateAndUploadT24FileViaFtp

        public async Task<GenerateAndUploadT24FileViaFtpResponse> GenerateAndUploadT24FileViaFtp(GenerateAndUploadT24FileViaFtpRequest request)
        {
            GenerateAndUploadT24FileViaFtpResponse response = new GenerateAndUploadT24FileViaFtpResponse()
            {
                Req = request,
            };

            GetFileImportContentByFileImportIdRequest getFileImportContentByFileImportIdRequest = new()
            {
                BaseReq = request.BaseReq,
                FileImportId = request.FileImportId
            };

            List<AlfaFileImportContent> fileImportContent =
                await GetFileImportContentByFileImportId(getFileImportContentByFileImportIdRequest);

            if (fileImportContent.Count == 0)
            {
                throw new SGBLBadRequestException("Cannot Generate T24 File For a File That Does Not Have Content!");
            }

            String fileName = $"{Utils.GetJulianDate()}-{Utils.GetPaddedSeconds()}_ALFA.txt";
            ;
            // String totalFileName = $"total_alfa_TSAL.txt";

            // Mapping FileImportContent Records to T24 File Fields
            List<Dictionary<String, String>> bodyRecords = GetAlfaT24FileContentRecords(fileImportContent);

            // Generate checksum for data before generating the t24 file
            string t24Checksum = Utils.GetObjectChecksum(bodyRecords);

            // Calling the Data Export API to Generate the actual T24 file
            String dataExportGenerateFileContentUrl = $"{_globalSettings.DataExportUrl}/GenerateAndUploadFileViaFtp";

            DataExportGenerateAndUploadFileViaFtpRequest dataExportGenerateFileContentRequest = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                FileCode = _globalSettings.GenerateT24FileCode,
                FileName = fileName,
                FilePath = _globalSettings.FtpConfigurations.AlfaFtpConfig.RemotePath,
                BodyRecords = bodyRecords
            };

            DataExportGenerateFileContentResponse responseData = await PostAsync<DataExportGenerateFileContentResponse>(dataExportGenerateFileContentRequest, dataExportGenerateFileContentUrl);

            if (responseData is not { WebResp.StatusCode: HttpStatusCode.OK })
            {
                throw new SGBLBadRequestException(String.Join(',', responseData.WebResp.Errors));
            }

            // Update file import status and t24 checksum
            UpdateFileImportDetailsRequest updateFileImportDetailsRequest = new UpdateFileImportDetailsRequest()
            {
                Id = request.FileImportId,
                BaseReq = request.BaseReq,
                StatusCode = FileImportStatus.AwaitingT24FileReturned.ToString(),
                T24FileCheckSum = t24Checksum
            };

            await UpdateFileImportDetails(updateFileImportDetailsRequest);

            return response;
        }

        #endregion

        #region UploadAndValidateT24File

        public async Task<UploadAndValidateT24FileResponse> UploadAndValidateT24File(UploadAndValidateT24FileRequest request)
        {
            UploadAndValidateT24FileResponse response = new UploadAndValidateT24FileResponse()
            {
                Req = request
            };

            string dataExportValidateFileUrl = $"{_globalSettings.DataExportUrl}/ValidateFile";

            if (request.T24File == null || request.T24File.Length == 0)
            {
                throw new SGBLBadRequestException("Invalid t24 file");
            }

            //get the file configurations
            Common.Model.File fileConfig = await GetFileConfigurationByCode(nameof(FileConfigCodes.AlfaT24Config));
            
            Common.Dto.FileInfo dataExportFileInfo = fileConfig.ToFileInfoDto();

            // Read data from TEXT using data export
            byte[] fileBytes = await Utils.ToByteArrayAsync(request.T24File);

            string hex = Utils.ByteArrayToHexString(fileBytes);

            dataExportFileInfo.FileBinary = hex;

            DataExportValidateFileRequest dataExportValidateFileRequest = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                FileInfo = dataExportFileInfo,
                FileCode = _globalSettings.GenerateT24FileCode,
                FieldNames = fileConfig.FileColumns!.Select(x => x.ColTypeCode).ToList()
            };

            DataExportValidateFileResponse responseData = await PostAsync<DataExportValidateFileResponse>(dataExportValidateFileRequest, dataExportValidateFileUrl);

            if (responseData is not { WebResp.StatusCode: HttpStatusCode.OK })
            {
                throw new SGBLBadGateWayException(string.Join(Environment.NewLine, responseData.WebResp.Errors));
            }

            GetAlfaT24ClientsFromParsedDataResponse t24ParsedData = GetAlfaT24ClientsFromParsedData(responseData.ParsedDataList, fileConfig);
            // DataExportGenerateFileContentResponse t24ParsedDataWithPaidAmount = GetAlfaT24ClientsFromParsedData(responseData.ParsedDataList, fileConfig);
            

            // Generate checksum for data before generating the t24 file
            string t24Checksum = Utils.GetObjectChecksum(t24ParsedData.t24NormalizedData);

            return response;
        }

        #endregion
    }
}
