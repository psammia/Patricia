using Common.Constants;

namespace Common.Dto
{
    #region FileInfo
    public class FileInfo
    {
        public string FileBinary { get; set; } = String.Empty;
        public FileType FileType { get; set; }
        public string Delimiter { get; set; } = String.Empty;
        public EncodingEnum Encoding { get; set; }
        public bool ValidateHeader { get; set; }
        public int HeaderRowNumber { get; set; }
        public List<string> ValidateHeaderContainsFields { get; set; } = [];
        public List<SkipRowString> SkipRowStrings { get; set; } = [];
        public List<Column> Columns { get; set; } = [];
        public string? ExcelPassword { get; set; }
    }
    #endregion

    #region SkipRowString
    public class SkipRowString
    {
        public string SkipString { get; set; } = String.Empty;
        public bool IsRegExp { get; set; }
    }
    #endregion

    #region CustomDataValidation
    public class CustomDataValidation
    {
        public int InjestFilePropertyId { get; set; }
    }
    #endregion

    #region Column
    public class Column
    {
        public bool FindFromHeader { get; set; }
        public string HeaderColumnName { get; set; } = String.Empty;
        public int ColumnNumber { get; set; }
        public int StartRowNumber { get; set; }
        public ReadEndCondition ReadEndCondition { get; set; }
        public string ReadEndConditionValue { get; set; } = String.Empty;
        public List<CustomDataValidation> CustomDataValidation { get; set; } = [];
    }
    #endregion

    #region ParsedData
    public class ParsedData
    {
        public int ColumnIndex { get; set; }
        public string ColumnName { get; set; } = String.Empty;
        public List<ColumnData> ColumnData { get; set; } = [];
    }
    #endregion

    #region ColumnData
    public class ColumnData
    {
        public string Data { get; set; } = String.Empty;
        public int RowNumber { get; set; }
    }
    #endregion

    #region AlfaClient
    public class AlfaClient
    {
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
    }
    #endregion

    #region LookupItemDTO

    public class LookupItemDTO
    {
        public string Code { get; set; } = string.Empty;
        public string Description { get; set; } = string.Empty;
        public string Value { get; set; } = string.Empty;
    }

    #endregion

    #region AlfaClientExport
    public class AlfaClientExport
    {
        public int BankCode { get; set; }
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
    }
    #endregion
    
    #region t24File
    
    public class T24File
    {
        public string BANK_NAME { get; set; } = string.Empty;
        public string BRANCH_NAME { get; set; } = string.Empty;
        public string ACCOUNT { get; set; } = string.Empty;
        public string CUSTOMER_NAME { get; set; } = string.Empty;
        public string CUSTOMER_ID { get; set; } = string.Empty;
        public string REFERENCE { get; set; } = string.Empty;
        public string AMOUNT { get; set; } = string.Empty;
        public string INV_DATE { get; set; } = string.Empty;
        public string PAYMENT_DETAILS { get; set; } = string.Empty;
    }
    
    #endregion
    
    #region GetAlfaT24ClientsFromParsedDataResponse

    public class GetAlfaT24ClientsFromParsedDataResponse
    {
        public List<T24File> t24File { get; set; }
        public List<Dictionary<string, string>> t24NormalizedData {get;set;}
    }
    #endregion

}
