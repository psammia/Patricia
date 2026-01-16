I'll create a complete API to get file imports with flexible filtering.
1. SQL Stored Procedure
sql/****** Object:  StoredProcedure [dbo].[usp_Get_File_Import_By_Where]    Script Date: 16/01/2026 ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
CREATE PROCEDURE [dbo].[usp_Get_File_Import_By_Where]
    @P__FromDate DATETIME2(0) = NULL,
    @P__ToDate DATETIME2(0) = NULL,
    @P__StatusCode NVARCHAR(50) = NULL,
    @P__Error NVARCHAR(4000) OUTPUT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @StartOfToday DATETIME2(0);
    DECLARE @EndOfToday DATETIME2(0);

    SET @P__Error = '';

    -- Set default dates to today if not provided
    IF @P__FromDate IS NULL
    BEGIN
        SET @StartOfToday = CAST(CAST(GETDATE() AS DATE) AS DATETIME2(0));
        SET @P__FromDate = @StartOfToday;
    END

    IF @P__ToDate IS NULL
    BEGIN
        SET @EndOfToday = DATEADD(SECOND, -1, DATEADD(DAY, 1, CAST(CAST(GETDATE() AS DATE) AS DATETIME2(0))));
        SET @P__ToDate = @EndOfToday;
    END

    -- Ensure FromDate is start of day (00:00:00)
    SET @P__FromDate = CAST(CAST(@P__FromDate AS DATE) AS DATETIME2(0));

    -- Ensure ToDate is end of day (23:59:59)
    SET @P__ToDate = DATEADD(SECOND, -1, DATEADD(DAY, 1, CAST(CAST(@P__ToDate AS DATE) AS DATETIME2(0))));

    -- Get file imports based on filters
    SELECT 
        fi.[Id],
        fi.[FileId],
        fi.[AttachmentId],
        att.[Name],
        fi.[CurrencyCode],
        fi.[StatusCode],
        lk.[Description] AS StatusDescription,
        fi.[CheckSum],
        fi.[T24FileCheckSum],
        fi.[CreatedDate],
        fi.[CreatedBy],
        fi.[LastModifiedDate],
        fi.[LastModifiedBy]
    FROM [dbo].[t_File_Import] fi
    INNER JOIN [dbo].[t_Attachment] att 
        ON fi.AttachmentId = att.Id
    LEFT JOIN [dbo].[t_Lookup] lk
        ON lk.Code = fi.StatusCode
       AND lk.TableName = 'FileImportStatus'
       AND lk.IsActive = 1
    WHERE 
        fi.[LastModifiedDate] >= @P__FromDate
        AND fi.[LastModifiedDate] <= @P__ToDate
        AND (@P__StatusCode IS NULL OR fi.[StatusCode] = @P__StatusCode)
    ORDER BY fi.[LastModifiedDate] DESC, fi.[Id] DESC;
END
GO
2. Request Class (Add to Request.cs)
csharp#region GetFileImportByWhereRequest
public class GetFileImportByWhereRequest
{
    public required BaseRequest BaseReq { get; set; }
    
    /// <summary>
    /// Start date filter. If null, defaults to today at 00:00:00
    /// </summary>
    public DateTime? FromDate { get; set; }
    
    /// <summary>
    /// End date filter. If null, defaults to today at 23:59:59
    /// </summary>
    public DateTime? ToDate { get; set; }
    
    /// <summary>
    /// Status code filter. If null or empty, returns all statuses
    /// Valid values: AwaitingT24FileUpload, AwaitingT24FileReturned, AwaitingAlfaUpload, Completed, Discarded
    /// </summary>
    public string? StatusCode { get; set; }
}
#endregion
3. Response Class (Add to Response.cs)
csharp#region GetFileImportByWhereResponse
public class GetFileImportByWhereResponse
{
    public BaseResponse BaseResp { get; set; } = new BaseResponse();
    public GetFileImportByWhereRequest Req { get; set; }
    public List<FileImport> FileImportList { get; set; } = [];
    public int TotalRecords { get; set; }
}
#endregion
4. BAL Method (Add to Bal.cs)
csharp#region GetFileImportByWhere
public async Task<GetFileImportByWhereResponse> GetFileImportByWhere(GetFileImportByWhereRequest request)
{
    DAL.DapperDal dal = new DapperDal(_globalSettings.ConnString);

    DynamicParameters parameters = new DynamicParameters();

    // Pass null if not provided, stored procedure will handle defaults
    parameters.Add("@P__FromDate", request.FromDate);
    parameters.Add("@P__ToDate", request.ToDate);
    parameters.Add("@P__StatusCode", string.IsNullOrWhiteSpace(request.StatusCode) ? null : request.StatusCode.Trim());
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

    List<FileImport> fileImportList = response.ToList();

    return new GetFileImportByWhereResponse
    {
        Req = request,
        FileImportList = fileImportList,
        TotalRecords = fileImportList.Count
    };
}
#endregion
5. Controller Method (Add to TelecomController.cs)
csharp#region GetFileImportByWhere
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

        response = await _bal.GetFileImportByWhere(request);

        // Set BaseResp after getting data from BAL
        response.BaseResp = new BaseResponse()
        {
            CorrelationId = request.BaseReq.CorrelationId,
            ReturnCode = _responseCodesDictionary["200"].Content,
            ReturnDescription = _responseCodesDictionary["200"].Description
        };

        correlationInfo.RDirection = RequestDirection.Response;
        correlationInfo.Reserved = $"GetFileImportByWhere replied with {response.TotalRecords} records";
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
