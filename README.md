// ========================================
// FILE: Models/Request.cs (ADD TO EXISTING)
// ========================================
namespace BLL.Models
{
    #region Delete Applications By Status Request
    public class Delete_ApplicationsByStatus_Request
    {
        public required BaseRequest BaseRequest { get; set; }
        public Int32 StatusId { get; set; } = 2;
    }
    #endregion
}

// ========================================
// FILE: Models/Response.cs (ADD TO EXISTING)
// ========================================
namespace BLL.Models
{
    #region Delete Applications By Status Response
    public class Delete_ApplicationsByStatus_Response
    {
        public BaseResponse BaseResponse { get; set; } = new BaseResponse();
        public Int32 DeletedApplicationsCount { get; set; }
        public Int32 DeletedFilesCount { get; set; }
        public List<DeletedApplicationInfo> DeletedApplications { get; set; } = [];
    }

    public class DeletedApplicationInfo
    {
        public Int64 Id { get; set; }
        public String CorrelationId { get; set; } = String.Empty;
        public String External_Id { get; set; } = String.Empty;
        public Int32 StatusId { get; set; }
    }
    #endregion
}

// ========================================
// FILE: BLL/Bll.cs (ADD TO EXISTING CLASS)
// ========================================
namespace BLL
{
    public partial class Bll
    {
        #region Delete Applications By Status
        public async Task<(List<DeletedApplicationInfo> deletedApps, int filesCount)> Delete_ApplicationsByStatus(Delete_ApplicationsByStatus_Request request)
        {
            DapperDal dal = new DapperDal(_globalSettings.ConnString);
            DynamicParameters parameters = new DynamicParameters();
            
            parameters.Add("P__StatusId", request.StatusId);

            // Get deleted applications info
            IEnumerable<DeletedApplicationInfo> deletedAppsResult = await dal.ExecuteQueryAsync<DeletedApplicationInfo>(
                "usp_DeleteApplicationsByStatus",
                parameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Delete);

            List<DeletedApplicationInfo> deletedApps = deletedAppsResult.ToList();

            // Get deleted files count
            DynamicParameters countParameters = new DynamicParameters();
            countParameters.Add("P__StatusId", request.StatusId);

            IEnumerable<DeletedFilesCount> filesCountResult = await dal.ExecuteQueryAsync<DeletedFilesCount>(
                "usp_GetDeletedFilesCount",
                countParameters,
                CommandType.StoredProcedure,
                DapperDal.CommandDirection.Select);

            int filesCount = filesCountResult.FirstOrDefault()?.FilesCount ?? 0;

            return (deletedApps, filesCount);
        }

        // Helper class for files count
        private class DeletedFilesCount
        {
            public int FilesCount { get; set; }
        }
        #endregion
    }
}

// ========================================
// FILE: Controllers/OnBoardingController.cs (ADD TO EXISTING)
// ========================================
namespace Alterna_OnBoarding.Controllers
{
    public partial class OnBoardingController
    {
        #region Delete Applications By Status
        [HttpPost]
        [Route("Delete_ApplicationsByStatus")]
        public async Task<Delete_ApplicationsByStatus_Response> Delete_ApplicationsByStatus([FromBody] Delete_ApplicationsByStatus_Request request)
        {
            Delete_ApplicationsByStatus_Response response = new Delete_ApplicationsByStatus_Response()
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
                RequestURL = "DeleteApplicationsByStatus",
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

                // Validate StatusId
                if (request.StatusId <= 0)
                {
                    throw new SGBLBadRequestException("StatusId must be greater than 0");
                }

                DataIntegrityCheck(dataGuardDict);
                #endregion

                correlationInfo.Reserved = "DeleteApplicationsByStatus has been called with the following Request";
                LogInfoJson(request, correlationInfo);

                // Call BLL method
                var (deletedApps, filesCount) = await _bll.Delete_ApplicationsByStatus(request);

                // Check if any applications were deleted
                if (deletedApps.Count == 0)
                {
                    response.BaseResponse.ReturnCode = _responseCodesDictionary["404"].Content;
                    response.BaseResponse.ReturnDescription = $"No applications found with StatusId = {request.StatusId}";
                    response.DeletedApplicationsCount = 0;
                    response.DeletedFilesCount = 0;
                    
                    correlationInfo.RDirection = RequestDirection.Response;
                    correlationInfo.Reserved = "No applications found to delete";
                    LogInfoJson(response, correlationInfo);
                    
                    return response;
                }

                // Set response data
                response.DeletedApplications = deletedApps;
                response.DeletedApplicationsCount = deletedApps.Count;
                response.DeletedFilesCount = filesCount;

                correlationInfo.RDirection = RequestDirection.Response;
                correlationInfo.Reserved = $"Deleted {deletedApps.Count} applications and {filesCount} files";
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
}

// ========================================
// FILE: Database/usp_DeleteApplicationsByStatus.sql
// ========================================
/*
USE [Alterna.OnBoarding]
GO

-- =====================================================================
-- Author:      Patricia Sammia
-- Create date: 03-10-2025
-- Description: Delete Applications and their Files by StatusId
-- =====================================================================
CREATE OR ALTER PROCEDURE [dbo].[usp_DeleteApplicationsByStatus]
(
    @P__StatusId INT
)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        BEGIN TRANSACTION;

        -- Store applications to be deleted before deletion
        DECLARE @DeletedApplications TABLE
        (
            Id BIGINT,
            CorrelationId NVARCHAR(250),
            External_Id NVARCHAR(10),
            StatusId INT
        );

        -- Get applications that will be deleted
        INSERT INTO @DeletedApplications (Id, CorrelationId, External_Id, StatusId)
        SELECT 
            Id,
            CorrelationId,
            External_Id,
            StatusId
        FROM dbo.t_Application
        WHERE StatusId = @P__StatusId;

        -- Delete related files first (foreign key constraint)
        DELETE f
        FROM dbo.t_App_Files f
        INNER JOIN dbo.t_Application a ON f.App_Id = a.Id
        WHERE a.StatusId = @P__StatusId;

        -- Delete applications
        DELETE FROM dbo.t_Application
        WHERE StatusId = @P__StatusId;

        COMMIT TRANSACTION;

        -- Return deleted applications info
        SELECT 
            Id,
            CorrelationId,
            External_Id,
            StatusId
        FROM @DeletedApplications;

    END TRY
    BEGIN CATCH
        -- Rollback if any error happens
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;

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
GO

-- =====================================================================
-- Author:      Patricia Sammia
-- Create date: 03-10-2025
-- Description: Get count of deleted files by StatusId
-- =====================================================================
CREATE OR ALTER PROCEDURE [dbo].[usp_GetDeletedFilesCount]
(
    @P__StatusId INT
)
AS
BEGIN
    SET NOCOUNT ON;

    BEGIN TRY
        -- Return count of files that would be deleted
        SELECT 
            COUNT(*) AS FilesCount
        FROM dbo.t_App_Files f
        INNER JOIN dbo.t_Application a ON f.App_Id = a.Id
        WHERE a.StatusId = @P__StatusId;

    END TRY
    BEGIN CATCH
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
GO
*/

// ========================================
// Sample Request/Response
// ========================================
/*
POST /api/OnBoarding/Delete_ApplicationsByStatus
Content-Type: application/json

REQUEST (Default StatusId = 2):
{
  "baseRequest": {
    "correlationId": "12345-67890-ABCDE-2025"
  },
  "statusId": 2
}

REQUEST (Custom StatusId):
{
  "baseRequest": {
    "correlationId": "12345-67890-ABCDE-2025"
  },
  "statusId": 3
}

RESPONSE (Success - Applications Deleted):
{
  "baseResponse": {
    "correlationId": "12345-67890-ABCDE-2025",
    "returnCode": "200",
    "returnDescription": "Success"
  },
  "deletedApplicationsCount": 3,
  "deletedFilesCount": 7,
  "deletedApplications": [
    {
      "id": 1001,
      "correlationId": "CORR-001",
      "external_Id": "EXT001",
      "statusId": 2
    },
    {
      "id": 1002,
      "correlationId": "CORR-002",
      "external_Id": "EXT002",
      "statusId": 2
    },
    {
      "id": 1003,
      "correlationId": "CORR-003",
      "external_Id": "EXT003",
      "statusId": 2
    }
  ]
}

RESPONSE (No Applications Found):
{
  "baseResponse": {
    "correlationId": "12345-67890-ABCDE-2025",
    "returnCode": "404",
    "returnDescription": "No applications found with StatusId = 2"
  },
  "deletedApplicationsCount": 0,
  "deletedFilesCount": 0,
  "deletedApplications": []
}

RESPONSE (Bad Request):
{
  "baseResponse": {
    "correlationId": "12345-67890-ABCDE-2025",
    "returnCode": "400",
    "returnDescription": "Bad Request: StatusId must be greater than 0"
  },
  "deletedApplicationsCount": 0,
  "deletedFilesCount": 0,
  "deletedApplications": []
}
*/
