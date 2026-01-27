
==Constants.cs=======================================
namespace Common.Constants
{
    #region FileConfigCodes 
    public enum FileConfigCodes 
    {
        AlfaConfig,
        AlfaT24Config
    }
    #endregion

    #region FileType
    public enum FileType
    {
        EXCEL = 1,
        CSV = 2,
        TEXT = 3,
    }
    #endregion

    #region EncodingEnum
    public enum EncodingEnum
    {
        UTF8 = 1,
        LATIN1 = 2,
        UNICODE = 3,
        BIGENDIANUNICODE = 4,
        UTF32 = 5,
        ASCII = 6,
        WINDOWS1256 = 7
    }
    #endregion

    #region ReadEndCondition
    public enum ReadEndCondition
    {
        FILE_END = 1,
    }
    #endregion

    #region ColumnType
    public enum ColumnType
    {
        Bank_Code,
        Bank_Name,
        Bank_Branch,
        Bank_Account_Number,
        Customer_Name,
        Primary_Account_Number,
        MSISDN_Primary_Contact,
        Account_Balance,
        Invoice_Date,
        Amount_Paid,
        Sayrafa_Rate
    }
    #endregion

    #region GeneratedT24FileFields
    public enum GeneratedT24FileFields
    {
        BANK_NAME,
        BRANCH_NAME,
        ACCOUNT,
        CUSTOMER_NAME,
        CUSTOMER_ID,
        REFERENCE,
        AMOUNT,
        INV_DATE,
        PAYMENT_DETAILS
    }
    #endregion

    #region CurrencyType
    public enum CurrencyType
    {
        LBP = 422,
        USD = 840,
    }
    #endregion
    
    #region FileImportStatus

    public enum FileImportStatus
    {
        AwaitingT24FileUpload,
        AwaitingT24FileReturned,
        AwaitingAlfaUpload,
        Completed,
        Discarded
    }
    #endregion
}

==CustomCode.cs=======================================
using Common.Constants;
using Common.CustomExceptions;
using Common.Dto;
using Common.Model;
using Common.Request;
using Common.Response;
using DAL;
using Dapper;
using Microsoft.AspNetCore.Http;
using System.Data;
using System.Globalization;
using static NLog.NLogUtil;
using System.Net;
using Common;

namespace BAL
{
    public partial class Bal
    {
        public void InitializeEvents()
        {
        }

        #region GetFileConfigurationByCode

        public async Task<Common.Model.File> GetFileConfigurationByCode(string fileCode)
        {
            DapperDal dal = new DapperDal(_globalSettings.ConnString);

            DynamicParameters paramsOne = new DynamicParameters();
            paramsOne.Add("P__Code", fileCode);

            Common.Model.File? fileConfig = await dal.QueryMultipleAsync(
                "usp_Get_File_By_Code",
                paramsOne,
                CommandType.StoredProcedure,
                async multi =>
                {
                    Common.Model.File? file = multi.ReadFirstOrDefault<Common.Model.File>();

                    if (file != null)
                    {
                        IEnumerable<FileColumn> fileColumns = await multi.ReadAsync<FileColumn>();

                        file.FileColumns = fileColumns.ToList();
                    }

                    return file;
                });

            if (fileConfig == null)
            {
                throw new SGBLBadRequestException("file config code did not match any configurations in the system");
            }

            if (fileConfig is { FileColumns: null } || fileConfig.FileColumns.Count == 0)
            {
                throw new SGBLBadRequestException("file configuration is missing the columns configuration");
            }

            List<int> columnIdList = fileConfig.FileColumns.Select(c => c.Id).ToList();

            if (columnIdList.Count > 0)
            {
                string joinedColumns = string.Join(',', columnIdList);

                DynamicParameters paramsTwo = new DynamicParameters();
                paramsTwo.Add("P__ColumnIds", joinedColumns);

                IEnumerable<FileColumnValidation> fileColumnValidations =
                    await dal.ExecuteQueryAsync<FileColumnValidation>(
                        "usp_Get_File_Columns_Validation_By_File_Column_List",
                        paramsTwo,
                        CommandType.StoredProcedure,
                        DapperDal.CommandDirection.Select);

                foreach (FileColumn fileColumn in fileConfig.FileColumns)
                {
                    fileColumn.FileColumnValidations =
                        fileColumnValidations.Where(v => v.FileColumnId == fileColumn.Id).ToList();
                }
            }

            return fileConfig;
        }

        #endregion

        #region GetAlfaClientsFromParsedData

        private List<AlfaClient> GetAlfaClientsFromParsedData(List<ParsedData> parsedDataList,
            Common.Model.File fileConfig)
        {
            List<string> errors = new List<string>();
            List<AlfaClient> alfaClients = new List<AlfaClient>();

            // Resolve ParsedData per column (by index first, then header name)
            ParsedData? ResolveParsedData(FileColumn column) =>
                parsedDataList.FirstOrDefault(pd => pd.ColumnIndex == column.Column) ?? parsedDataList.FirstOrDefault(
                    pd =>
                        pd.ColumnName.Equals(column.HeaderColumnName, StringComparison.OrdinalIgnoreCase));

            // Resolve FileColumns from configuration
            FileColumn bankCodeColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Bank_Code));

            FileColumn bankNameColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Bank_Name));

            FileColumn bankBranchColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Bank_Branch));

            FileColumn bankAccountNumberColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Bank_Account_Number));

            FileColumn customerNameColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Customer_Name));

            FileColumn primaryAccountNumberColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Primary_Account_Number));

            FileColumn msisdnColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.MSISDN_Primary_Contact));

            FileColumn accountBalanceColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Account_Balance));

            FileColumn invoiceDateColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Invoice_Date));

            FileColumn amountPaidColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Amount_Paid));

            FileColumn sayrafaRateColumn = fileConfig.FileColumns!
                .First(fc => fc.ColTypeCode.Trim() == nameof(ColumnType.Sayrafa_Rate));

            // Resolve ParsedData
            ParsedData? bankCodeParsedData = ResolveParsedData(bankCodeColumn);
            ParsedData? bankNameParsedData = ResolveParsedData(bankNameColumn);
            ParsedData? bankBranchParsedData = ResolveParsedData(bankBranchColumn);
            ParsedData? bankAccountParsedData = ResolveParsedData(bankAccountNumberColumn);
            ParsedData? customerNameParsedData = ResolveParsedData(customerNameColumn);
            ParsedData? primaryAccountParsedData = ResolveParsedData(primaryAccountNumberColumn);
            ParsedData? msisdnParsedData = ResolveParsedData(msisdnColumn);
            ParsedData? balanceParsedData = ResolveParsedData(accountBalanceColumn);
            ParsedData? invoiceDateParsedData = ResolveParsedData(invoiceDateColumn);
            ParsedData? amountPaidParsedData = ResolveParsedData(amountPaidColumn);
            ParsedData? sayrafaRateParsedData = ResolveParsedData(sayrafaRateColumn);

            if (msisdnParsedData == null)
            {
                throw new Exception("MSISDN column not found in parsed data.");
            }

            if (balanceParsedData == null)
            {
                throw new Exception("Account Balance column not found in parsed data.");
            }

            // Build row-based dictionaries
            Dictionary<int, string?> bankCodeByRow =
                bankCodeParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> bankNameByRow =
                bankNameParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> bankBranchByRow =
                bankBranchParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> bankAccountByRow =
                bankAccountParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> customerNameByRow =
                customerNameParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> primaryAccountByRow =
                primaryAccountParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> msisdnByRow =
                msisdnParsedData.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim());

            Dictionary<int, string?> balanceByRow =
                balanceParsedData.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim());

            Dictionary<int, string?> invoiceDateByRow =
                invoiceDateParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> amountPaidByRow =
                amountPaidParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> sayrafaRateByRow =
                sayrafaRateParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            // Union of all row numbers
            List<int> rowNumbers =
                msisdnByRow.Keys
                    .Union(balanceByRow.Keys)
                    .OrderBy(r => r)
                    .ToList();

            foreach (int rowNumber in rowNumbers)
            {
                msisdnByRow.TryGetValue(rowNumber, out string? msisdn);
                balanceByRow.TryGetValue(rowNumber, out string? balanceRaw);

                if (string.IsNullOrWhiteSpace(msisdn))
                {
                    throw new SGBLBadRequestException($"Row {rowNumber}: MSISDN is empty.");
                }

                if (!decimal.TryParse(balanceRaw, out decimal balance))
                {
                    throw new SGBLBadRequestException($"Row {rowNumber}: Invalid Account Balance '{balanceRaw}'.");
                }

                string? invoiceDateRaw = Utils.GetT24DateFormat(invoiceDateByRow.GetValueOrDefault(rowNumber)!);

                if (!DateTime.TryParseExact(
                        invoiceDateRaw,
                        "yyyyMMdd",
                        CultureInfo.InvariantCulture,
                        DateTimeStyles.None,
                        out DateTime invoiceDate))
                {
                    throw new SGBLBadRequestException(
                        $"Row {rowNumber}: Invalid Invoice Date '{invoiceDateRaw}'. Expected format MM/dd/yyyy.");
                }

                string? amountPaidRaw = amountPaidByRow.GetValueOrDefault(rowNumber);
                decimal amountPaid = string.IsNullOrWhiteSpace(amountPaidRaw)
                    ? 0
                    : decimal.Parse(amountPaidRaw, CultureInfo.InvariantCulture);

                string? sayrafaRateRaw = sayrafaRateByRow.GetValueOrDefault(rowNumber);
                decimal sayrafaRate = string.IsNullOrWhiteSpace(sayrafaRateRaw)
                    ? 0
                    : decimal.Parse(sayrafaRateRaw, CultureInfo.InvariantCulture);

                alfaClients.Add(new AlfaClient
                {
                    BankCode = bankCodeByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    BankName = bankNameByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    BankBranch = bankBranchByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    BankAccountNumber = bankAccountByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    CustomerName = customerNameByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    PrimaryAccountNumber = primaryAccountByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    MsisdnPrimaryContact = msisdn,
                    AccountBalance = balance,
                    InvoiceDate = invoiceDate,
                    AmountPaid = amountPaid,
                    SayrafaRate = sayrafaRate
                });
            }

            if (errors.Any())
            {
                throw new Exception(string.Join(Environment.NewLine, errors));
            }

            return alfaClients;
        }

        #endregion

        #region GetAlfaT24ClientsFromParsedData
        private GetAlfaT24ClientsFromParsedDataResponse GetAlfaT24ClientsFromParsedData(List<ParsedData> parsedDataList, Common.Model.File fileConfig)
        {
            List<string> errors = new List<string>();
            List<T24File> alfaClients = new List<T24File>();

            // Resolve ParsedData per column (by index first, then header name)
            ParsedData? ResolveParsedData(FileColumn column) =>
                parsedDataList.FirstOrDefault(pd => pd.ColumnIndex == column.Column) ?? parsedDataList.FirstOrDefault(
                    pd =>
                        pd.ColumnName.Equals(column.HeaderColumnName, StringComparison.OrdinalIgnoreCase));

            // Resolve FileColumns from configuration
            FileColumn bankNameColumn = fileConfig.FileColumns!.First(fc => fc.ColTypeCode.Trim() == nameof(GeneratedT24FileFields.BANK_NAME));
            FileColumn bankBranchColumn = fileConfig.FileColumns!.First(fc => fc.ColTypeCode.Trim() == nameof(GeneratedT24FileFields.BRANCH_NAME));
            FileColumn bankAccountNumberColumn = fileConfig.FileColumns!.First(fc => fc.ColTypeCode.Trim() == nameof(GeneratedT24FileFields.ACCOUNT));
            FileColumn customerNameColumn = fileConfig.FileColumns!.First(fc => fc.ColTypeCode.Trim() == nameof(GeneratedT24FileFields.CUSTOMER_NAME));
            FileColumn primaryAccountNumberColumn = fileConfig.FileColumns!.First(fc => fc.ColTypeCode.Trim() == nameof(GeneratedT24FileFields.CUSTOMER_ID));
            FileColumn msisdnColumn = fileConfig.FileColumns!.First(fc => fc.ColTypeCode.Trim() == nameof(GeneratedT24FileFields.REFERENCE));
            FileColumn amountColumn = fileConfig.FileColumns!.First(fc => fc.ColTypeCode.Trim() == nameof(GeneratedT24FileFields.AMOUNT));
            FileColumn invoiceDateColumn = fileConfig.FileColumns!.First(fc => fc.ColTypeCode.Trim() == nameof(GeneratedT24FileFields.INV_DATE));
            FileColumn amountPaidColumn = fileConfig.FileColumns!.First(fc => fc.ColTypeCode.Trim() == nameof(GeneratedT24FileFields.PAYMENT_DETAILS));


            // Resolve ParsedData
            ParsedData? bankNameParsedData = ResolveParsedData(bankNameColumn);
            ParsedData? bankBranchParsedData = ResolveParsedData(bankBranchColumn);
            ParsedData? bankAccountParsedData = ResolveParsedData(bankAccountNumberColumn);
            ParsedData? customerNameParsedData = ResolveParsedData(customerNameColumn);
            ParsedData? primaryAccountParsedData = ResolveParsedData(primaryAccountNumberColumn);
            ParsedData? msisdnParsedData = ResolveParsedData(msisdnColumn);
            ParsedData? amountParsedData = ResolveParsedData(amountColumn);
            ParsedData? invoiceDateParsedData = ResolveParsedData(invoiceDateColumn);
            ParsedData? amountPaidParsedData = ResolveParsedData(amountPaidColumn);

            if (msisdnParsedData == null)
            {
                throw new Exception("MSISDN column not found in parsed data.");
            }

            // Build row-based dictionaries
            Dictionary<int, string?> bankNameByRow =
                bankNameParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();
            
            Dictionary<int, string?> bankBranchByRow =
                bankBranchParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();
            
            Dictionary<int, string?> bankAccountByRow =
                bankAccountParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> customerNameByRow =
                customerNameParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> primaryAccountByRow = 
                primaryAccountParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();      
            
            Dictionary<int, string?> msisdnByRow = 
                msisdnParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();            
            
            Dictionary<int, string?> amountByRow = 
                amountParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            Dictionary<int, string?> invoiceDateByRow =
                invoiceDateParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();
            
            Dictionary<int, string?> amountPaidByRow =
                amountPaidParsedData?.ColumnData.ToDictionary(x => x.RowNumber, x => x.Data?.Trim())
                ?? new Dictionary<int, string?>();

            // Union of all row numbers
            List<int> rowNumbers = 
                msisdnByRow.Keys
                    .ToList();

            foreach (int rowNumber in rowNumbers)
            {
                alfaClients.Add(new T24File
                {
                    BANK_NAME = bankNameByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    BRANCH_NAME = bankBranchByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    ACCOUNT = bankAccountByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    CUSTOMER_NAME = customerNameByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    CUSTOMER_ID = primaryAccountByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    REFERENCE = msisdnByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    AMOUNT = amountByRow.GetValueOrDefault(rowNumber)!.Trim().TrimStart('0') ?? string.Empty,
                    INV_DATE = invoiceDateByRow.GetValueOrDefault(rowNumber) ?? string.Empty,
                    PAYMENT_DETAILS = amountPaidByRow.GetValueOrDefault(rowNumber) ?? string.Empty
                });
            }

            if (errors.Any())
            {
                throw new Exception(string.Join(Environment.NewLine, errors));
            }

            alfaClients = alfaClients.OrderBy(x => x.REFERENCE).ThenBy(x => x.CUSTOMER_NAME).ThenBy(x => x.AMOUNT).ToList();
            var normalizedClients = alfaClients.Select(NormalizeT24Object).OrderBy(x => x["REFERENCE"])
                .ThenBy(x => x["CUSTOMER_NAME"]).ThenBy(x => x["AMOUNT"]).ToList();
            GetAlfaT24ClientsFromParsedDataResponse response = new GetAlfaT24ClientsFromParsedDataResponse()
            {
                t24File = alfaClients,
                t24NormalizedData = normalizedClients,
            };
            
            return response;
        }
        

        #endregion

        #region InsertAlfaClients

        public async Task InsertAlfaClients(
            int fileConfigId,
            CurrencyType currencyCode,
            string attachmentName,
            string checkSum,
            DateTime cycle,
            string directory,
            List<AlfaClient> alfaClients,
            string userName)
        {
            DapperDal dal = new DapperDal(_globalSettings.ConnString);

            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("P__FileId", fileConfigId);
            parameters.Add("P__CurrencyCode", currencyCode);
            parameters.Add("P__Name", attachmentName);
            parameters.Add("P__CheckSum", checkSum);
            parameters.Add("P__Directory", directory);
            parameters.Add("P__FileImportContentAlfa", GetFileImportContentAlfaDt(alfaClients).AsTableValuedParameter());
            parameters.Add("P__Cycle", cycle);
            parameters.Add("P__User", userName);
            parameters.Add("P__Error", direction: ParameterDirection.Output, size: 4000);

            _ = await dal.ExecuteQueryAsync<dynamic>(
                "usp_Bulk_Insert_File_Import_Content_Alfa",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Update);

            string errorMessage = parameters.Get<string>("P__Error");

            if (!string.IsNullOrWhiteSpace(errorMessage))
            {
                throw new SGBLBadRequestException(errorMessage);
            }
        }

        #endregion

        #region GetFileImportContentAlfaDt

        private DataTable GetFileImportContentAlfaDt(List<AlfaClient> alfaClients)
        {
            DataTable alfaClientDt = new("TVP_File_Import_Content_Alfa");

            alfaClientDt.Columns.Add("BankCode");
            alfaClientDt.Columns.Add("BankName");
            alfaClientDt.Columns.Add("BankBranch");
            alfaClientDt.Columns.Add("BankAccountNumber");
            alfaClientDt.Columns.Add("CustomerName");
            alfaClientDt.Columns.Add("PrimaryAccountNumber");
            alfaClientDt.Columns.Add("MsisdnPrimaryContact");
            alfaClientDt.Columns.Add("AccountBalance");
            alfaClientDt.Columns.Add("InvoiceDate");
            alfaClientDt.Columns.Add("AmountPaid");
            alfaClientDt.Columns.Add("SayrafaRate");

            foreach (AlfaClient alfaClient in alfaClients)
            {
                DataRow dr = alfaClientDt.NewRow();

                dr["BankCode"] = alfaClient.BankCode;
                dr["BankName"] = alfaClient.BankName;
                dr["BankBranch"] = alfaClient.BankBranch;
                dr["BankAccountNumber"] = alfaClient.BankAccountNumber;
                dr["CustomerName"] = alfaClient.CustomerName;
                dr["PrimaryAccountNumber"] = alfaClient.PrimaryAccountNumber;
                dr["MsisdnPrimaryContact"] = alfaClient.MsisdnPrimaryContact;
                dr["AccountBalance"] = alfaClient.AccountBalance;
                dr["InvoiceDate"] = alfaClient.InvoiceDate;
                dr["AmountPaid"] = alfaClient.AmountPaid;
                dr["SayrafaRate"] = alfaClient.SayrafaRate;

                alfaClientDt.Rows.Add(dr);
            }

            return alfaClientDt;
        }

        #endregion

        #region CheckFileCheckSum

        public async Task CheckFileCheckSum(string checkSum)
        {
            DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

            DynamicParameters parameters = new DynamicParameters();

            parameters.Add("P__CheckSum", checkSum);
            parameters.Add("P__Error", direction: ParameterDirection.Output, size: 4000);

            _ = await dal.ExecuteQueryAsync<dynamic>(
                "usp_Check_File_CheckSum",
                parameters,
                CommandType.StoredProcedure,
                cmdDirection: DapperDal.CommandDirection.Select);

            string errorMessage = parameters.Get<string>("P__Error");

            if (!string.IsNullOrWhiteSpace(errorMessage))
            {
                throw new SGBLBadRequestException(errorMessage);
            }
        }

        #endregion

        #region GetAlfaT24FileContentRecords

        private List<Dictionary<string, string>> GetAlfaT24FileContentRecords(
            List<AlfaFileImportContent> alfaFileImportContents)
        {
            // List<Dictionary<string, string>> bodyRecords = [];
            List<Dictionary<string, string>> bodyRecords = [];

            int index = 1;

            foreach (AlfaFileImportContent alfaFileImportContent in alfaFileImportContents)
            {
                bodyRecords.Add(new()
                {
                    { GeneratedT24FileFields.BANK_NAME.ToString(), _globalSettings.BankName },
                    {
                        GeneratedT24FileFields.BRANCH_NAME.ToString(),
                        alfaFileImportContent.BankBranch.SafeSubstring(0, 9)
                    },
                    {
                        GeneratedT24FileFields.ACCOUNT.ToString(),
                        alfaFileImportContent.BankAccountNumber.TryGetAccNumber()
                    },
                    {
                        GeneratedT24FileFields.CUSTOMER_NAME.ToString(),
                        alfaFileImportContent.CustomerName.SafeSubstring(0, 32)
                    },
                    {
                        GeneratedT24FileFields.CUSTOMER_ID.ToString(),
                        alfaFileImportContent.PrimaryAccountNumber.GetLastSixCharOfPrimaryAccount()
                    },
                    { GeneratedT24FileFields.REFERENCE.ToString(), alfaFileImportContent.MsisdnPrimaryContact },
                    {
                        GeneratedT24FileFields.AMOUNT.ToString(),
                        Utils.GetAmountInT24Format(alfaFileImportContent.AccountBalance)
                    }, // 1000.500 -> 1000500 -- explanation for how decimal is treated by T24
                    {
                        GeneratedT24FileFields.INV_DATE.ToString(),
                        alfaFileImportContent.InvoiceDate.ToString("yyyyMMdd")
                    }
                });

                index++;
            }

            return bodyRecords.OrderBy(x => x[GeneratedT24FileFields.REFERENCE.ToString()])
                .ThenBy(x => x[GeneratedT24FileFields.CUSTOMER_NAME.ToString()])
                .ThenBy(x =>
                    long.TryParse(x[GeneratedT24FileFields.AMOUNT.ToString()], out var amount)
                        ? amount
                        : 0
                )
                .ToList();
        }

        #endregion
        
        #region NormalizeT24Object

        private static Dictionary<string, string> NormalizeT24Object(T24File file)
        {
            return new Dictionary<string, string>
            {
                ["BANK_NAME"] = file.BANK_NAME ?? "",
                ["BRANCH_NAME"] = file.BRANCH_NAME ?? "",
                ["ACCOUNT"] = file.ACCOUNT ?? "",
                ["CUSTOMER_NAME"] = file.CUSTOMER_NAME ?? "",
                ["CUSTOMER_ID"] = file.CUSTOMER_ID ?? "",
                ["REFERENCE"] = file.REFERENCE ?? "",
                ["AMOUNT"] = file.AMOUNT ?? "",
                ["INV_DATE"] = file.INV_DATE ?? ""
            };
        }

        #endregion
    }    
}

==Controller.cs============================
using BAL;
using Common;
using Common.CustomExceptions;
using Common.Request;
using Common.Response;
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Options;
using System.Text;
using Dapper;
using static NLog.NLogUtil;

namespace Alterna_Telecom_Backend.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class TelecomController : ControllerBase
    {
        private readonly Bal _bal;
        private readonly GlobalSettings _globalSettings;
        private readonly Dictionary<string, TelecomResponseCode> _responseCodesDictionary = [];

        public TelecomController(
            Bal bal,
            IOptionsMonitor<GlobalSettings> globalSettings,
            IOptionsMonitor<TelecomResponseCodes> responseCodes)
        {
            _bal = bal;
            _globalSettings = globalSettings.CurrentValue;

            foreach (TelecomResponseCode responseCode in responseCodes.CurrentValue.ResponseCodes)
            {
                _responseCodesDictionary.Add(responseCode.Code, new TelecomResponseCode()
                {
                    Content = responseCode.Content,
                    Description = responseCode.Description
                });
            }
        }

        #region ProcessAlfaAttachment

        [HttpPost]
        [Route("ProcessAlfaAttachment")]
        public async Task<ProcessAlfaAttachmentResponse> ProcessAlfaAttachment([FromForm] ProcessAlfaAttachmentRequest request)
        {
            ProcessAlfaAttachmentResponse response = new ProcessAlfaAttachmentResponse()
            {
                Req = request,
                BaseResp = new BaseResponse()
                {
                    CorrelationId = request.BaseReq.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "ProcessAlfaAttachment",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "ProcessAlfaAttachment has been called with the following Request";

                LogInfoJson(request, correlationInfo);

                await _bal.ProcessAlfaAttachment(request);

                correlationInfo.Reserved = "ProcessAlfaAttachment requested with the following response";

                LogInfoJson(response, correlationInfo);

                return response;
            }
            catch (SGBLBadRequestException ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }

        #endregion

        #region DownloadArchivedFile

        [HttpPost]
        [Route("DownloadArchivedFile")]
        public async Task<IActionResult> DownloadArchivedFile(DownloadArchivedFileRequest request)
        {
            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "DownloadArchivedFile",
                UserName = request.BaseReq.UserName,
                Reserved = "DownloadArchivedFile has been called with the following Request"
            };

            try
            {
                LogInfoJson(request, correlationInfo);

                (byte[] fileBytes, string contentType, string fileName) result =
                    await _bal.DownloadArchivedFile(request);

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = "DownloadArchivedFile finished retrieving the file";

                LogInfo("DownloadArchivedFile finished retrieving the file", correlationInfo);

                return File(
                    result.fileBytes,
                    result.contentType,
                    result.fileName);
            }
            catch (SGBLBadRequestException ex)
            {
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo);

                return StatusCode(400, ex.Message);
            }
            catch (Exception ex)
            {
                correlationInfo.RDirection = RequestDirection.Response;

                LogError(ex.Message, correlationInfo);

                return StatusCode(500, ex.Message);
            }
        }

        #endregion

        #region GetAllFileImport

        [HttpPost]
        [Route("GetAllFileImport")]
        public async Task<GetAttachmentsResponse> GetAllFileImport(GetAllFileImportRequest request)
        {
            GetAttachmentsResponse response = new GetAttachmentsResponse()
            {
                Req = request,
                BaseResp = new BaseResponse()
                {
                    CorrelationId = request.BaseReq.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };

            CorrelationInfo correlationInfo = new CorrelationInfo()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetAllModules",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "GetAllFileImport has been called with the following Request";
                LogInfoJson(request, correlationInfo);

                response.FileImportList = await _bal.GetAllFileImport(request);

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = "GetAllFileImport replied with the following response";
                LogInfoJson(response, correlationInfo);

                return response;
            }
            catch (SGBLBadRequestException ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();
                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }

        #endregion

        #region GetFileImportContentByFileImportId

        [HttpPost]
        [Route("GetFileImportContentByFileImportId")]
        public async Task<GetFileImportContentByFileImportIdResponse> GetFileImportContentByFileImportId(GetFileImportContentByFileImportIdRequest request)
        {
            GetFileImportContentByFileImportIdResponse response = new GetFileImportContentByFileImportIdResponse()
            {
                Req = request,
                BaseResp = new BaseResponse()
                {
                    CorrelationId = request.BaseReq.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };

            CorrelationInfo correlationInfo = new CorrelationInfo()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetFileImportContentByFileImportId",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "GetFileImportContentByFileImportId has been called with the following Request";
                LogInfoJson(request, correlationInfo);

                response.FileImportContentList = await _bal.GetFileImportContentByFileImportId(request);

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = "GetFileImportContentByFileImportId replied with the following response";
                LogInfoJson(response, correlationInfo);

                return response;
            }
            catch (SGBLBadRequestException ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();
                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }

        #endregion

        #region GenerateAndUploadT24File

        [HttpPost]
        [Route("GenerateAndUploadT24FileViaFtp")]
        public async Task<GenerateAndUploadT24FileViaFtpResponse> GenerateAndUploadT24File(GenerateAndUploadT24FileViaFtpRequest request)
        {
            GenerateAndUploadT24FileViaFtpResponse response = new GenerateAndUploadT24FileViaFtpResponse()
            {
                Req = request,
                BaseResp = new BaseResponse()
                {
                    CorrelationId = request.BaseReq.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "ProcessAlfaAttachment",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "GenerateAndUploadT24File has been called with the following Request";

                LogInfoJson(request, correlationInfo);

                await _bal.GenerateAndUploadT24FileViaFtp(request);

                correlationInfo.Reserved = "GenerateAndUploadT24File requested with the following response";

                LogInfoJson(response, correlationInfo);

                return response;
            }
            catch (SGBLBadRequestException ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }

        #endregion
        
        #region UploadAndValidateT24File

        [HttpPost]
        [Route("UploadAndValidateT24File")]
        public async Task<UploadAndValidateT24FileResponse> UploadAndValidateT24File([FromForm] UploadAndValidateT24FileRequest request)
        {
            UploadAndValidateT24FileResponse response = new UploadAndValidateT24FileResponse()
            {
                Req = request,
                BaseResp = new BaseResponse()
                {
                    CorrelationId = request.BaseReq.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };

            CorrelationInfo correlationInfo = new()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UploadT24FileAndCrossMatch",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "UploadT24FileAndCrossMatch has been called with the following Request";

                LogInfoJson(request, correlationInfo);

                dynamic test = await _bal.UploadAndValidateT24File(request);

                correlationInfo.Reserved = "UploadT24FileAndCrossMatch requested with the following response";

                LogInfoJson(response, correlationInfo);

                return test;
            }
            catch (SGBLBadRequestException ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }
        #endregion

        #region GetLookupsByTableName

        [HttpPost]
        [Route("GetLookupsByTableName")]
        public async Task<GetLookupsByTableNameResponse> GetLookupsByTableName(GetLookupsByTableNameRequest request)
        {
            GetLookupsByTableNameResponse response = new GetLookupsByTableNameResponse()
            {
                Req = request,
                BaseResp = new BaseResponse()
                {
                    CorrelationId = request.BaseReq.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };

            CorrelationInfo correlationInfo = new CorrelationInfo()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetLookupsByTableName",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "GetAllLookups has been called with the following Request";
                LogInfoJson(request, correlationInfo);

                response.LookupList = await _bal.GetLookupsByTableName(request);

                correlationInfo.RDirection = RequestDirection.Request;

                correlationInfo.Reserved = "GetAllLookups replied with the following response";
                LogInfoJson(request, correlationInfo);

                return response;
            }
            catch (SGBLBadRequestException ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();
                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);

                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();

                correlationInfo.RDirection = RequestDirection.Response;

                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }

        #endregion

        #region UpdateFileImportContentMsisdn

        [HttpPost]
        [Route("UpdateFileImportContentMsisdn")]
        public async Task<UpdateFileImportContentMsisdnResponse> UpdateFileImportContentMsisdn(
            [FromBody] UpdateFileImportContentMsisdnRequest request)
        {
            UpdateFileImportContentMsisdnResponse response = new UpdateFileImportContentMsisdnResponse()
            {
                Req = request,
                BaseResp = new BaseResponse()
                {
                    CorrelationId = request.BaseReq.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };

            CorrelationInfo correlationInfo = new CorrelationInfo()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateFileImportContentMsisdn",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "UpdateFileImportContentMsisdn has been called with the following Request";
                LogInfoJson(request, correlationInfo);

                await _bal.UpdateFileImportContentMsisdn(request);

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = "UpdateFileImportContentMsisdn completed successfully";
                LogInfoJson(response, correlationInfo);

                return response;
            }
            catch (SGBLBadRequestException ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);
                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();
                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);
                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();
                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }

        #endregion

        #region ExportFileImportContentToExcel

        [HttpPost]
        [Route("ExportFileImportContentToExcel")]
        public async Task<IActionResult> ExportFileImportContentToExcel(ExportFileImportContentToExcelRequest request)
        {
            CorrelationInfo correlationInfo = new CorrelationInfo()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "ExportFileImportContentToExcel",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "ExportFileImportContentToExcel has been called with the following Request";
                LogInfoJson(request, correlationInfo);

                (byte[] fileBytes, string fileName) = await _bal.ExportFileImportContentToExcel(request);

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = "ExportFileImportContentToExcel completed successfully";
                LogInfo($"Excel file generated: {fileName}", correlationInfo);

                return File(
                    fileBytes,
                    "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                    fileName);
            }
            catch (SGBLBadRequestException ex)
            {
                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = ex.Message;
                LogError(ex.Message, correlationInfo);

                return StatusCode(400, ex.Message);
            }
            catch (Exception ex)
            {
                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = ex.Message;
                LogError(ex.Message, correlationInfo);

                return StatusCode(500, ex.Message);
            }
        }

        #endregion

        #region GetFileImportByWhere

        [HttpPost]
        [Route("GetFileImportByWhere")]
        public async Task<GetFileImportByWhereResponse> GetFileImportByWhere(GetFileImportByWhereRequest request)
        {
            GetFileImportByWhereResponse response = new GetFileImportByWhereResponse()
            {
                Req = request,
                BaseResp = new BaseResponse()
                {
                    CorrelationId = request.BaseReq.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };

            CorrelationInfo correlationInfo = new CorrelationInfo()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "GetFileImportByWhere",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "GetFileImportByWhere has been called with the following Request";
                LogInfoJson(request, correlationInfo);

                response.FileImportList = await _bal.GetFileImportByWhere(request);

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = "GetFileImportByWhere requested with the following response";
                LogInfoJson(response, correlationInfo);

                return response;
            }
            catch (SGBLBadRequestException ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);
                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();
                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch (Exception ex)
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);
                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();
                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
        }

        #endregion

        #region UpdateFileImportDetails
        [HttpPost]
        [Route("UpdateFileImportDetails")]
       public async Task<UpdateFileImportDetailsResponse> UpdateFileImportDetails(UpdateFileImportDetailsRequest request)
        {
            UpdateFileImportDetailsResponse response = new UpdateFileImportDetailsResponse()
            {
                Req = request, 
                BaseResp = new BaseResponse()
                {
                    CorrelationId = request.BaseReq.CorrelationId,
                    ReturnCode = _responseCodesDictionary["200"].Content,
                    ReturnDescription = _responseCodesDictionary["200"].Description
                }
            };
            CorrelationInfo correlationInfo = new CorrelationInfo()
            {
                CorrelationId = request.BaseReq.CorrelationId,
                RDirection = RequestDirection.Request,
                RequestURL = "UpdateFileImportDetails",
                UserName = request.BaseReq.UserName
            };

            try
            {
                correlationInfo.Reserved = "UpdateFileImportDetails has been called with the following Request";
                LogInfoJson(request, correlationInfo);

                await _bal.UpdateFileImportDetails(request);

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = "UpdateFileImportContentMsisdn completed successfully";
                LogInfoJson(response, correlationInfo);

                return response;
            }
            catch (SGBLBadRequestException ex) 
            {
                StringBuilder sb = new(_responseCodesDictionary["400"].Description);
                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["400"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();
                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            }
            catch(Exception ex) 
            {
                StringBuilder sb = new(_responseCodesDictionary["500"].Description);
                sb.Replace("{0}", ex.Message);

                response.BaseResp.CorrelationId = request.BaseReq.CorrelationId;
                response.BaseResp.ReturnCode = _responseCodesDictionary["500"].Content;
                response.BaseResp.ReturnDescription = sb.ToString();
                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = ex.Message;
                LogErrorJson(response, correlationInfo, ex);

                return response;
            } 
            
        }

        #endregion

    }
}

==Request.cs================
using System.ComponentModel.DataAnnotations;
using System.Runtime.InteropServices.JavaScript;
using System.Text.Json;
using Common.Constants;
using Common.Model;
using Common.Request;

using Microsoft.AspNetCore.Http;

namespace Common.Request
{
    #region BaseRequest

    public class BaseRequest
    {
        public string CorrelationId { get; set; } = string.Empty;
        public string UserName { get; set; } = string.Empty;
    }

    #endregion

    #region ProcessAlfaAttachment

    public class ProcessAlfaAttachmentRequest()
    {
        public required BaseRequest BaseReq { get; set; }
        public required IFormFile Attachment { get; set; }
        public CurrencyType CurrencyCode { get; set; }
        public DateTime Cycle { get; set; }
    }

    #endregion

    #region DataExportValidateFileRequest

    public class DataExportValidateFileRequest
    {
        public required string CorrelationId { get; set; }
        public required Dto.FileInfo FileInfo { get; set; }
        public string? FileCode { get; set; }
        public List<string>? FieldNames { get; set; }
    }

    #endregion

    #region DataExportUploadFileViaFtpReq

    public class DataExportUploadFileViaFtpReq
    {
        public string CorrelationId { get; set; } = string.Empty;
        public IFormFile? File { get; set; }
        public string FtpHost { get; set; } = string.Empty;
        public int? FtpPort { get; set; }
        public string Username { get; set; } = string.Empty;
        public string Password { get; set; } = string.Empty;
        public string RemotePath { get; set; } = string.Empty;
        public bool CreateRemoteDirectory { get; set; }
        public bool IsFtpsEnabled { get; set; }
    }

    #endregion

    #region DownloadArchivedFileRequest

    public class DownloadArchivedFileRequest
    {
        public required BaseRequest BaseReq { get; set; }
        public long AttachmentId { get; set; }
    }

    #endregion

    #region GetAttachmentsRequest

    public class GetAllFileImportRequest
    {
        public required BaseRequest BaseReq { get; set; }
    }

    #endregion

    #region GetFileImportContentByFileImportId

    public class GetFileImportContentByFileImportIdRequest
    {
        public required BaseRequest BaseReq { get; set; }
        public required long FileImportId { get; set; }
    }

    #endregion

    #region GenerateAndUploadT24FileViaFtpRequest

    public class GenerateAndUploadT24FileViaFtpRequest
    {
        public required BaseRequest BaseReq { get; set; }
        public required long FileImportId { get; set; }
    }

    #endregion

    #region GetLookupsByTableNameRequest

    public class GetLookupsByTableNameRequest
    {
        public required BaseRequest BaseReq { get; set; }
        public string TableNames { get; set; } = string.Empty;
    }

    #endregion

    #region UpdateFileImportContentMsisdnRequest

    public class UpdateFileImportContentMsisdnRequest
    {
        public required BaseRequest BaseReq { get; set; }
        public required long Id { get; set; }
        public required string MsisdnPrimaryContact { get; set; }
    }

    #endregion

    #region ExportFileImportContentToExcelRequest

    public class ExportFileImportContentToExcelRequest
    {
        public required BaseRequest BaseReq { get; set; }
        public required long FileImportId { get; set; }
    }

    #endregion

    #region GetFileImportByWhereRequest

    public class GetFileImportByWhereRequest
    {
        public required BaseRequest BaseReq { get; set; }

        public DateTime? FromDate { get; set; }

        public DateTime? ToDate { get; set; }

        public string? StatusCode { get; set; }
    }

    #endregion

    #region DataExportGenerateAndUploadFileViaFtpRequest

    public class DataExportGenerateAndUploadFileViaFtpRequest
    {
        public required String CorrelationId { get; set; }
        public required String FileCode { get; set; } = String.Empty;
        public required String FileName { get; set; } = String.Empty;
        public required String FilePath { get; set; } = String.Empty;
        public List<Dictionary<String, String>> HeaderRecords { get; set; } = [];
        public List<Dictionary<String, String>> BodyRecords { get; set; } = [];
        public List<Dictionary<String, String>> FooterRecords { get; set; } = [];
    }

    #endregion

    #region UploadT24FileAndCrossMatchRequest

    public class UploadAndValidateT24FileRequest
    {
        public required BaseRequest BaseReq { get; set; }
        public required IFormFile T24File { get; set; }
        
        public required long FileImportId { get; set; }
    }

    #endregion
    #region UpdateFileImportDetailsRequest
    public class UpdateFileImportDetailsRequest
    {
        public required BaseRequest BaseReq { get; set; }
        public required long Id { get; set; }
        public required string StatusCode { get; set; }
        public string? T24FileCheckSum {  get; set; }
    }

    #endregion
}

==Response.cs==============================

using Common.Dto;
using Common.Model;
using Common.Request;
using Microsoft.AspNetCore.Http;
using System.Net;

namespace Common.Response
{
    #region BaseResponse
    public class BaseResponse
    {
        public string CorrelationId { get; set; } = string.Empty;
        public string ReturnCode { get; set; } = string.Empty;
        public string ReturnDescription { get; set; } = string.Empty;
        public HttpStatusCode StatusCode { get; set; } = HttpStatusCode.OK;
        public List<string> Errors { get; set; } = [];
    }
    #endregion

    #region DataExportBaseRes
    public partial class DataExportBaseRes
    {
        public string CorrelationId { get; set; } = string.Empty;
        public string Ticket { get; set; } = string.Empty;
        public HttpStatusCode StatusCode { get; set; } = HttpStatusCode.OK;
        public List<string> Errors { get; set; } = [];
    }
    #endregion

    #region ProcessAlfaAttachement
    public class ProcessAlfaAttachmentResponse()
    {
        public BaseResponse BaseResp { get; set; } = new BaseResponse();
        public required ProcessAlfaAttachmentRequest Req { get; set; }
    }
    #endregion

    #region DataExportValidateFileResponse
    public class DataExportValidateFileResponse
    {
        public required DataExportBaseRes WebResp { get; set; }
        public List<ParsedData> ParsedDataList { get; set; } = [];
    }
    #endregion

    #region DataExportUploadFileViaFtpRes
    public class DataExportUploadFileViaFtpRes
    {
        public required DataExportBaseRes WebResp { get; set; }
    }
    #endregion
    
    #region GetAttachmentsResponse
    public class GetAttachmentsResponse
    {
        public BaseResponse BaseResp { get; set; } = new BaseResponse();
        public GetAllFileImportRequest Req { get; set; }
        public List<FileImport> FileImportList { get; set; } = [];
    }
    #endregion
    
    #region GetFileImportContentByFileImportId
    public class GetFileImportContentByFileImportIdResponse
    {
        public BaseResponse BaseResp { get; set; } = new BaseResponse();
        public GetFileImportContentByFileImportIdRequest Req { get; set; }
        public List<AlfaFileImportContent> FileImportContentList { get; set; } = [];
        public decimal TotalAmount { get; set; }
        public int TotalRecord { get;set; }
    }
    #endregion

    #region GenerateAndUploadT24FileViaFtpResponse
    public class GenerateAndUploadT24FileViaFtpResponse()
    {
        public BaseResponse BaseResp { get; set; } = new BaseResponse();
        public required GenerateAndUploadT24FileViaFtpRequest Req { get; set; }
    }
    #endregion

    #region GetLookupsByTableNameResponse

    public class GetLookupsByTableNameResponse
    {
        public BaseResponse BaseResp { get; set; } = new BaseResponse();
        public GetLookupsByTableNameRequest Req { get; set; }
        public Dictionary<string, List<Lookup>> LookupList { get; set; } = [];
    }
    #endregion

    #region UpdateFileImportContentMsisdnResponse
    public class UpdateFileImportContentMsisdnResponse
    {
        public BaseResponse BaseResp { get; set; } = new BaseResponse();
        public required UpdateFileImportContentMsisdnRequest Req { get; set; }
    }
    #endregion

    #region GetFileImportByWhereResponse
    public class GetFileImportByWhereResponse
    {
        public BaseResponse BaseResp { get; set; } = new BaseResponse();
        public GetFileImportByWhereRequest Req { get; set; }
        public List<FileImport> FileImportList { get; set; } = [];
    }
    #endregion
    
    #region DataExportGenerateFileContentResponse
    public class DataExportGenerateFileContentResponse
    {
        public BaseResponse WebResp { get; set; } = new();
        public String FileContent { get; set; } = String.Empty;
    }
    #endregion

    #region UploadAndValidateT24FileResponse

    public class UploadAndValidateT24FileResponse
    {
        public BaseResponse BaseResp { get; set; } = new BaseResponse();
        public required UploadAndValidateT24FileRequest Req { get; set; }
        // public List<AlfaClientExport> FileImportList { get; set; } = [];
    }

    #endregion
    #region UpdateFileImportDetailsResponse
    public class UpdateFileImportDetailsResponse
    {
        public BaseResponse BaseResp { get; set; } = new BaseResponse();
        public required UpdateFileImportDetailsRequest Req { get; set; }
    }
    #endregion

}


==Bal.cs============
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

==Model.cs=============
using Common.Constants;

namespace Common.Model
{
    #region File

    public class File
    {
        public int Id { get; set; }
        public int InstitutionId { get; set; }
        public string? InstitutionDescription { get; set; }
        public string Code { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
        public string FileTypeCode { get; set; } = string.Empty;
        public string? FileTypeDescription { get; set; }
        public string? EncodingCode { get; set; }
        public string? EncodingDescription { get; set; }
        public string? Password { get; set; }
        public int? SheetNumber { get; set; }
        public string? Delimeter { get; set; }
        public bool ValidateHeader { get; set; }
        public int? HeaderRow { get; set; }
        public string? ValidateHeaderContainsFields { get; set; }

        public List<FileColumn>? FileColumns { get; set; }
    }

    #endregion

    #region FileColumn

    public class FileColumn
    {
        public int Id { get; set; }
        public int FileId { get; set; }
        public string ColTypeCode { get; set; } = string.Empty;
        public string? ColTypeDescription { get; set; }
        public bool FindFromHeader { get; set; }
        public string? HeaderColumnName { get; set; }
        public int? Column { get; set; }
        public int StartRow { get; set; }
        public string ReadEndConditionCode { get; set; } = string.Empty;
        public string? ReadEndConditionDescription { get; set; }
        public string? ReadEndConditionValue { get; set; } = string.Empty;

        public List<FileColumnValidation>? FileColumnValidations { get; set; }
    }

    #endregion

    #region FileColumnValidation

    public class FileColumnValidation
    {
        public int Id { get; set; }
        public int FileColumnId { get; set; }
        public int IngestPropId { get; set; }
    }

    #endregion

    #region FileImport

    public class FileImport
    {
        public long Id { get; set; }
        public int FileId { get; set; }
        public string? AttachmentName { get; set; } = string.Empty;
        public long AttachmentId { get; set; }
        public string CurrencyCode { get; set; } = string.Empty;
        public string? CurrencyDescription { get; set; } = string.Empty;
        public string StatusCode { get; set; } = string.Empty;
        public string? StatusDescription { get; set; } = string.Empty;
        public DateTime Cycle { get; set; }
        public DateTime LastModifiedDate { get; set; }
    }

    #endregion

    #region AlfaFileImportContent

    public class AlfaFileImportContent
    {
        public long Id { get; set; }
        public long FileImportId { get; set; }
        public string? FileImportDescription { get; set; }
        public string BankCode { get; set; } = string.Empty;
        public string BankName { get; set; } = string.Empty;
        public string BankBranch { get; set; } = string.Empty;
        public string BankAccountNumber { get; set; } = string.Empty;
        public string CustomerName { get; set; } = string.Empty;
        public string PrimaryAccountNumber { get; set; } = string.Empty;
        public string MsisdnPrimaryContact { get; set; } = string.Empty;
        public decimal AccountBalance { get; set; }
        public DateTime InvoiceDate { get; set; }
        public decimal? AmountPaid { get; set; }
        public decimal? SayrafaRate { get; set; }

        public DateTime? LastModifiedDate { get; set; }
        public string? LastModifiedBy { get; set; }
    }

    #endregion

    #region Attachment

    public class Attachment
    {
        public long Id { get; set; }
        public string Name { get; set; } = string.Empty;
        public string Directory { get; set; } = string.Empty;
    }

    #endregion

    #region Lookup

    public class Lookup
    {
        public int Id { get; set; }
        public string TableName { get; set; } = string.Empty;
        public string Code { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
        public string Value { get; set; } = string.Empty;
        public Boolean IsActive { get; set; }
    }

    #endregion
}

==Utils.cs==================
using Common.CustomExceptions;
using Microsoft.AspNetCore.Http;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;
using System.Threading.Tasks;

namespace Common
{
    public static class Utils
    {
        #region HexStringToByteArray

        public static byte[] HexStringToByteArray(string hexString)
        {
            if (string.IsNullOrWhiteSpace(hexString))
            {
                throw new ArgumentException("Hex string cannot be null or empty.", nameof(hexString));
            }

            if (hexString.Length % 2 != 0)
            {
                throw new FormatException("Hex string must have an even length.");
            }

            byte[] bytes = new byte[hexString.Length / 2];
            for (int i = 0; i < hexString.Length; i += 2)
            {
                bytes[i / 2] = Convert.ToByte(hexString.Substring(i, 2), 16);
            }

            return bytes;
        }

        #endregion

        #region ByteArrayToHexString

        public static string ByteArrayToHexString(byte[] contentBytes)
        {
            StringBuilder sb = new StringBuilder(contentBytes.Length * 2);

            foreach (byte b in contentBytes)
            {
                sb.AppendFormat("{0:x2}", b);
            }

            return sb.ToString();
        }

        #endregion

        #region ToByteArrayAsync

        public static async Task<byte[]> ToByteArrayAsync(IFormFile formFile)
        {
            if (formFile == null)
            {
                throw new ArgumentNullException(nameof(formFile));
            }

            if (formFile.Length == 0)
            {
                return Array.Empty<byte>();
            }

            using MemoryStream memoryStream = new MemoryStream();
            await formFile.CopyToAsync(memoryStream);
            return memoryStream.ToArray();
        }

        #endregion

        #region CheckSumUtil

        public static async Task<string> GetFileChecksumAsync(Stream fileStream)
        {
            using System.Security.Cryptography.SHA256 sha256 = System.Security.Cryptography.SHA256.Create();
            byte[] hashBytes = await sha256.ComputeHashAsync(fileStream);
            return BytesToHex(hashBytes);
        }

        private static string BytesToHex(byte[] bytes)
        {
            return Convert.ToHexString(bytes).ToLowerInvariant();
        }

        public static async Task<string> GetFileChecksumAsync(IFormFile file)
        {
            using Stream stream = file.OpenReadStream();
            return await GetFileChecksumAsync(stream);
        }

        #endregion

        // public static string TestGetListChecksum<T>(List<T> list)
        // {
        //     return TestGetObjectChecksum(list);
        // }

        public static string GetObjectChecksum<T>(List<T> obj)
        {
            string json = JsonSerializer.Serialize(obj, new JsonSerializerOptions
            {
                WriteIndented = false,
                PropertyNamingPolicy = null,
                DefaultIgnoreCondition = System.Text.Json.Serialization.JsonIgnoreCondition.Never
            });

            byte[] bytes = Encoding.UTF8.GetBytes(json);

            // Compute SHA256 hash
            using SHA256 sha256 = SHA256.Create();
            byte[] hashBytes = sha256.ComputeHash(bytes);
            return BytesToHex(hashBytes);
        }

        #region UploadFile

        public static void UploadFile(string directory, IFormFile attachment)
        {
            if (!Directory.Exists(directory))
            {
                Directory.CreateDirectory(directory);
            }

            string filePath = Path.Combine(directory, attachment.FileName);

            using (FileStream stream = new FileStream(filePath, FileMode.Create))
            {
                attachment.CopyTo(stream);
            }

            if (!System.IO.File.Exists(filePath))
            {
                throw new Exception("An Error has occured while saving file");
            }
        }

        #endregion

        #region DownloadFile

        public static (byte[] fileBytes, string contentType, string fileName) DownloadFile(string arhcivedPath,
            string fileName)
        {
            string fullFilePath = Path.Combine(arhcivedPath, fileName);

            if (!System.IO.File.Exists(fullFilePath))
            {
                throw new Exception("The requested file was not found");
            }

            byte[] fileBytes = System.IO.File.ReadAllBytes(fullFilePath);

            string contentType = GetContentType(fileName);

            return (fileBytes, contentType, fileName);
        }

        #endregion

        #region GetContentType

        public static string GetContentType(string fileName)
        {
            string extension = Path.GetExtension(fileName).ToLowerInvariant();
            return extension switch
            {
                ".txt" => "text/plain",
                ".csv" => "text/csv",
                ".pdf" => "application/pdf",
                ".doc" => "application/msword",
                ".docx" => "application/vnd.openxmlformats-officedocument.wordprocessingml.document",
                ".xls" => "application/vnd.ms-excel",
                ".xlsx" => "application/vnd.openxmlformats-officedocument.spreadsheetml.sheet",
                ".png" => "image/png",
                ".jpg" or ".jpeg" => "image/jpeg",
                ".gif" => "image/gif",
                ".zip" => "application/zip",
                ".xml" => "application/xml",
                ".json" => "application/json",
                _ => throw new Exception("extension did not match any content type")
            };
        }

        #endregion

        #region GetT24DateFormat

        public static string GetT24DateFormat(string extractedDate)
        {
            return $"{extractedDate.Substring(6, 4)}{extractedDate.Substring(0, 2)}{extractedDate.Substring(3, 2)}";
        }

        #endregion

        #region GetAmountFormatForGeneratedT24File

        public static string GetAmountFormatForGeneratedT24File(Decimal amount)
        {
            long result = (long)(amount * 1000);
            return result.ToString();
        }

        #endregion

        #region SafeSubstring

        public static string SafeSubstring(this string str, int startIndex, int? length = null)
        {
            // Handle null/empty
            if (string.IsNullOrEmpty(str))
            {
                return string.Empty;
            }

            // Handle negative start index (convert to positive from end)
            if (startIndex < 0)
            {
                startIndex = Math.Max(0, str.Length + startIndex);
            }

            // If start is beyond string length, return empty
            if (startIndex >= str.Length)
            {
                return string.Empty;
            }

            // Clamp start to valid range
            startIndex = Math.Max(0, startIndex);

            // Calculate actual length to extract
            int actualLength;

            if (length.HasValue)
            {
                // Handle negative length (not standard, but can mean from end)
                if (length.Value < 0)
                {
                    return string.Empty;
                }

                // Clamp length to available characters
                actualLength = Math.Min(length.Value, str.Length - startIndex);
            }
            else
            {
                // No length specified, take rest of string
                actualLength = str.Length - startIndex;
            }

            // Ensure we don't have negative length
            if (actualLength <= 0)
            {
                return string.Empty;
            }

            return str.Substring(startIndex, actualLength);
        }

        #endregion

        #region GetAccNumber

        public static string TryGetAccNumber(this string receivedAccNumber)
        {
            if (string.IsNullOrWhiteSpace(receivedAccNumber))
            {
                return "01000000000000000";
            }

            //check if it ends with 422 or 480 then return last 15 chars
            if (receivedAccNumber.EndsWith("422") || receivedAccNumber.EndsWith("840"))
            {
                return receivedAccNumber.Length <= 15
                    ? receivedAccNumber
                    : receivedAccNumber.Substring(receivedAccNumber.Length - 15);
            }

            //otherwise return last 18 chars
            return receivedAccNumber.Length <= 18
                ? receivedAccNumber
                : receivedAccNumber.Substring(receivedAccNumber.Length - 18);
        }

        #endregion

        #region GetLastSixCharOfPrimaryAccount

        public static string GetLastSixCharOfPrimaryAccount(this string input)
        {
            return input.Length <= 6 ? input : input.Substring(input.Length - 6);
        }

        #endregion

        #region GetAmountInT24Format

        public static string GetAmountInT24Format(decimal input)
        {
            long result = (long)(input * 1000);

            return result.ToString();
        }

        #endregion

        #region GetJulianDate

        public static Int32 GetJulianDate()
        {
            return DateTime.Now.DayOfYear; // Returns the day number in the year
        }

        #endregion

        #region GetPaddedSeconds

        public static String GetPaddedSeconds()
        {
            Int32 seconds = (Int32)(DateTime.Now.TimeOfDay.TotalSeconds);
            return seconds.ToString().PadLeft(5, '0'); // Ensures 5-digit format with leading zeros
        }

        #endregion
    }
}
