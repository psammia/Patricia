// Add to ArchivingController.cs
using ALTERNA.ARCHIVING.BLL; // Make sure this using is at the top

[HttpPost]
[Route("BackfillMissingPDFs")]
public BackfillMissingPDFsRes BackfillMissingPDFs(BackfillMissingPDFsReq backfillReq)
{
    BackfillMissingPDFsRes response = new()
    {
        Req = backfillReq
    };

    CorrelationInfo correlationInfo = new()
    {
        CorrelationId = backfillReq.BaseReq.CorrelationId,
        RDirection = RequestDirection.Request,
        RequestURL = "BackfillMissingPDFs",
        UserName = backfillReq.BaseReq.CurrentUser
    };

    try
    {
        String CorrelationId = String.IsNullOrEmpty(backfillReq.BaseReq.CorrelationId) 
            ? throw new SGBLBadRequestException($"{nameof(CorrelationId)} Cannot Be null or empty") 
            : backfillReq.BaseReq.CorrelationId;
        
        String CurrentUser = String.IsNullOrEmpty(backfillReq.BaseReq.CurrentUser) 
            ? throw new SGBLBadRequestException($"{nameof(CurrentUser)} Cannot Be null or empty") 
            : backfillReq.BaseReq.CurrentUser;

        LogInfo("BackfillMissingPDFs Has been called with the following Request", correlationInfo);
        LogInfoJson(backfillReq, correlationInfo);

        correlationInfo.RDirection = RequestDirection.Processing;

        using (BLL.BLL oBLL = new(CurrentUser))
        {
            LogInfo("Start of BackfillMissingPDFs call", correlationInfo);

            response.Resp = oBLL.BackfillMissingPDFsForLegacyContainers(
                CurrentUser, 
                backfillReq.FromDate, 
                backfillReq.ToDate
            );

            response.WebResp.CorrelationId = CorrelationId;
            response.WebResp.User = CurrentUser;
            response.WebResp.HttpResponseCode = HttpStatusCode.OK;

            correlationInfo.RDirection = RequestDirection.Response;

            LogInfo("BackfillMissingPDFs Has Replied with the Following response", correlationInfo);
            LogInfoJson(response, correlationInfo);
            LogInfo("Calling the BackfillMissingPDFs is completed", correlationInfo);
        }

        return response;
    }
    catch (SGBLBadRequestException ex)
    {
        response.WebResp.CorrelationId = ex.Message.Contains("CorrelationId") 
            ? Guid.NewGuid().ToString() 
            : backfillReq.BaseReq.CorrelationId!;
        response.WebResp.User = ex.Message.Contains("CurrentUser") 
            ? "BadUser" 
            : backfillReq.BaseReq.CurrentUser!;
        response.WebResp.HttpResponseCode = HttpStatusCode.BadRequest;
        response.WebResp.ResponseMessage = ex.StackTrace;

        correlationInfo.CorrelationId = response.WebResp.CorrelationId;
        correlationInfo.UserName = response.WebResp.User;
        correlationInfo.StatusCode = HttpStatusCode.BadRequest;
        correlationInfo.RDirection = RequestDirection.Response;

        LogError(ex.Message, correlationInfo, ex);
        LogErrorJson(response, correlationInfo, ex);

        return response;
    }
    catch (Exception ex)
    {
        response.WebResp.CorrelationId = backfillReq.BaseReq.CorrelationId!;
        response.WebResp.User = backfillReq.BaseReq.CurrentUser!;
        response.WebResp.HttpResponseCode = HttpStatusCode.InternalServerError;
        response.WebResp.ResponseMessage = ex.StackTrace;

        correlationInfo.StatusCode = HttpStatusCode.InternalServerError;
        correlationInfo.RDirection = RequestDirection.Response;

        LogError(ex.StackTrace, correlationInfo);
        LogErrorJson(response, correlationInfo, ex);

        return response;
    }
}

// Request/Response Models - Add to your Models or Controller namespace
public partial class BackfillMissingPDFsReq
{
    public BaseRequest BaseReq { get; set; } = new BaseRequest();
    // Optional: Add filters like date range, specific containers, etc.
    public DateTime? FromDate { get; set; }
    public DateTime? ToDate { get; set; }
    public List<string>? SpecificContainers { get; set; }
}

public partial class BackfillMissingPDFsRes
{
    public BaseResponse WebResp { get; set; } = new BaseResponse();
    public required BackfillMissingPDFsReq Req { get; set; }
    public ALTERNA.ARCHIVING.BLL.BackfillResult? Resp { get; set; } // Use fully qualified name
}
