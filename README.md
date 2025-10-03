
Model
=====
public class Application
{
    public Int64 Id { get; set; }

    public String CorrelationId { get; set; }=String.Empty;
    public String External_Id { get; set; } = String.Empty;
    public Int32 StatusId { get; set;}
}

public class App_Files
{
    //public Int64 Id { get; set; }
    //public Int64 App_Id { get; set; }
    public String File_Name { get; set; } = String.Empty;
    public String File_Type { get; set; } = String.Empty;
    public Int64 File_Size { get; set; }
    public Byte[] File_Data { get; set; } = [];
}

Request
=======
namespace BLL;

public class BaseRequest
{
    public required String CorrelationId { get; set; }
}

#region Insert User Application with files Request
public class Insert_UserApplication_WithFiles_Request
{
    public required BaseRequest BaseRequest { get; set; }
    public required String CorrelationId { get; set; } = string.Empty;
    public required String External_Id { get; set; }=string.Empty;
    public List<App_Files> app_FilesList { get; set; } = [];    
}
#endregion

Response
========
namespace BLL;

public class BaseResponse
{
    public String CorrelationId { get; set; } = string.Empty;
    public String ReturnCode { get; set; } = String.Empty;
    public String ReturnDescription { get; set; } = String.Empty;
}

#region Insert User Application with files Request Response
public class Insert_UserApp_WithFiles_Response
{
    public BaseResponse BaseResponse { get; set; }=new BaseResponse();
    public required Insert_UserApplication_WithFiles_Request Request { get; set; }
}
#endregion

BLL
====
using System.Data;

using DAL;

using Dapper;

using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;

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
            parameters.Add("P__CorrelationId", request.CorrelationId);
            parameters.Add("P__External_Id", request.External_Id);
            parameters.Add("P__TVP_Files", GetAppFilesDt(request.app_FilesList).AsTableValuedParameter());


            IEnumerable<Application> res = await dal.ExecuteQueryAsync<Application>(
                "usp_InsertUserApplicationWithFiles",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Update);

            return res.ToList().First();

        }

        private DataTable GetAppFilesDt(List<App_Files> app_Files)
        {
            DataTable dt = new DataTable("TVP_Files");
            dt.Columns.Add("File_Name");
            dt.Columns.Add("File_Type");
            dt.Columns.Add("File_Size");
            dt.Columns.Add("File_Data");

            foreach (App_Files appFile in app_Files)
            {
                DataRow dr = dt.NewRow();

                dr["File_Name"] = appFile.File_Name;
                dr["File_Type"] = appFile.File_Type;
                dr["File_Size"] = appFile.File_Size;
                dr["File_Data"] = appFile.File_Data;

                dt.Rows.Add(dr);
            }
            return dt;
        }
        #endregion
    }
}

Conroller
========

using Microsoft.AspNetCore.Mvc;
using BLL;
using Microsoft.Extensions.Options;
using static NLog.NLogUtil;
using static BLL.DataGuard;
using System.Text;

namespace Alterna_OnBoarding.Controllers;

[Route("api/[controller]")]
[ApiController]
public class OnBoardingController : ControllerBase
{
    private readonly Bll _bll;
    private readonly GlobalSettings _globalSettings;
    private readonly Dictionary<String, OnBoardingResponseCode> _responseCodesDictionary = [];

    public OnBoardingController(Bll bll, IOptionsMonitor<GlobalSettings> globalSettings, IOptionsMonitor<OnBoardingResponseCodes> responseCodes)
    {
        _bll = bll;
        _globalSettings = globalSettings.CurrentValue;

        foreach (OnBoardingResponseCode responseCode in responseCodes.CurrentValue.ResponseCodes)
        {
            _responseCodesDictionary.Add(responseCode.Code, new OnBoardingResponseCode()
            {
                Description = responseCode.Description,
                Content = responseCode.Content
            });
        }
    }

    #region Insert User Application With Files      
    [HttpPost]
    [Route("Insert_UserApplication_WithFiles")]
    public async Task<Insert_UserApp_WithFiles_Response> Insert_UserApplication_WithFiles(Insert_UserApplication_WithFiles_Request request)
    {
        Insert_UserApp_WithFiles_Response response = new Insert_UserApp_WithFiles_Response()
        {
            Request = request,
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
            RequestURL = "InsertUserApplicationWithFiles",
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
                    })
            ];

            DataIntegrityCheck(dataGuardDict);
            #endregion

            correlationInfo.Reserved = "InsertUserApplicationWithFiles has been called with the following Request";

            LogInfoJson(request, correlationInfo);

            await _bll.Insert_UserApplication_WithFiles(request);

            correlationInfo.RDirection = RequestDirection.Response;

            correlationInfo.Reserved = "InsertUserApplicationWithFiles replied with the following response";

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
}

Stored procedure
================
USE [Alterna.OnBoarding]
GO
/****** Object:  StoredProcedure [dbo].[usp_InsertUserApplicationWithFiles]    Script Date: 03/10/2025 8:46:27 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =====================================================================
-- Author:		<Patricia Sammia>
-- Create date: <01-10-2025>
-- Description:	<Insert Client Application through BB with files list>
-- =====================================================================
ALTER      PROCEDURE [dbo].[usp_InsertUserApplicationWithFiles]
(
    @P__CorrelationId NVARCHAR(250),
    @P__External_Id NVARCHAR(10),
    @P__TVP_Files dbo.FileListType READONLY
)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @AppId BIGINT;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- 1. Insert into t_Application with StatusId = 1
        INSERT INTO dbo.t_Application
        (
            External_Id,
            CorrelationId,
            StatusId,
            CreatedBy,
            CreatedDate,
            LastModifiedBy,
            LastModifiedDate
        )
        VALUES
        (
            @P__External_Id,
            @P__CorrelationId,
            1,               
            'AlternaSysUser',   
            GETDATE(),
            'AlternaSysUser',
            GETDATE()
        );

        SET @AppId = SCOPE_IDENTITY();

        -- 2. Insert related files into t_App_Files
        INSERT INTO dbo.t_App_Files
        (
            App_Id,
            File_Name,
            File_Type,
            File_Size,
            File_Data,
            CreatedBy,
            CreatedDate,
            LastModifiedBy,
            LastModifiedDate
        )
        SELECT
            @AppId,
            f.File_Name,
            f.File_Type,
            f.File_Size,
            f.File_Data,
            'AlternaSysUser',
            GETDATE(),
            'AlternaSysUser',
            GETDATE()
        FROM P__TVP_Files f;

        COMMIT TRANSACTION;
    END TRY

    BEGIN CATCH
        -- Rollback if any error happens
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

        -- Re-throw the error with details
        DECLARE @ErrorMessage NVARCHAR(4000),
                @ErrorSeverity INT,
                @ErrorState INT;

        SELECT
            @ErrorMessage = ERROR_MESSAGE(),
            @ErrorSeverity = ERROR_SEVERITY(),
            @ErrorState = ERROR_STATE();

        RAISERROR (@ErrorMessage, @ErrorSeverity, @ErrorState);
    END CATCH
END


