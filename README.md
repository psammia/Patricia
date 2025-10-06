using System;
using System.Net;
using System.Web.Http;
using YourNamespace.BLL;

namespace YourNamespace.API.Controllers
{
    [RoutePrefix("api/applications")]
    public class ApplicationsController : ApiController
    {
        private readonly ApplicationBLL _applicationBLL;

        public ApplicationsController()
        {
            _applicationBLL = new ApplicationBLL();
        }

        public ApplicationsController(ApplicationBLL applicationBLL)
        {
            _applicationBLL = applicationBLL;
        }

        /// <summary>
        /// Deletes all applications with Status ID = 2 and their related files (byte arrays in database)
        /// </summary>
        /// <returns>Result with deletion details</returns>
        [HttpDelete]
        [Route("delete-by-status-two")]
        public IHttpActionResult DeleteApplicationsWithStatusTwo()
        {
            try
            {
                DeletionResult result = _applicationBLL.DeleteApplicationsWithStatusTwo();

                return Ok(new
                {
                    success = result.Success,
                    message = result.GetMessage(),
                    data = new
                    {
                        deletedApplications = result.DeletedApplicationsCount,
                        deletedFiles = result.DeletedFilesCount
                    }
                });
            }
            catch (ArgumentException ex)
            {
                return BadRequest(ex.Message);
            }
            catch (Exception ex)
            {
                // Log the exception
                return InternalServerError(new Exception($"An error occurred while deleting applications: {ex.Message}"));
            }
        }

        /// <summary>
        /// Deletes all applications with the specified status ID and their related files (byte arrays in database)
        /// </summary>
        /// <param name="statusId">The status ID to filter applications</param>
        /// <returns>Result with deletion details</returns>
        [HttpDelete]
        [Route("delete-by-status/{statusId:int}")]
        public IHttpActionResult DeleteApplicationsByStatus(int statusId)
        {
            try
            {
                DeletionResult result = _applicationBLL.DeleteApplicationsByStatus(statusId);

                return Ok(new
                {
                    success = result.Success,
                    message = result.GetMessage(),
                    statusId = statusId,
                    data = new
                    {
                        deletedApplications = result.DeletedApplicationsCount,
                        deletedFiles = result.DeletedFilesCount
                    }
                });
            }
            catch (ArgumentException ex)
            {
                return BadRequest(ex.Message);
            }
            catch (Exception ex)
            {
                // Log the exception
                return InternalServerError(new Exception($"An error occurred while deleting applications: {ex.Message}"));
            }
        }
    }
}

// ===== For ASP.NET Core (Alternative) =====
/*
using Microsoft.AspNetCore.Mvc;
using Microsoft.Extensions.Logging;
using YourNamespace.BLL;

namespace YourNamespace.API.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class ApplicationsController : ControllerBase
    {
        private readonly ApplicationBLL _applicationBLL;
        private readonly ILogger<ApplicationsController> _logger;

        public ApplicationsController(ApplicationBLL applicationBLL, ILogger<ApplicationsController> logger)
        {
            _applicationBLL = applicationBLL;
            _logger = logger;
        }

        /// <summary>
        /// Deletes all applications with Status ID = 2 and their related files (byte arrays in database)
        /// </summary>
        [HttpDelete("delete-by-status-two")]
        public IActionResult DeleteApplicationsWithStatusTwo()
        {
            try
            {
                DeletionResult result = _applicationBLL.DeleteApplicationsWithStatusTwo();

                _logger.LogInformation($"Deleted {result.DeletedApplicationsCount} applications and {result.DeletedFilesCount} files with Status ID = 2");

                return Ok(new
                {
                    success = result.Success,
                    message = result.GetMessage(),
                    data = new
                    {
                        deletedApplications = result.DeletedApplicationsCount,
                        deletedFiles = result.DeletedFilesCount
                    }
                });
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { success = false, message = ex.Message });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, "Error deleting applications with status 2");
                return StatusCode(500, new 
                { 
                    success = false, 
                    message = $"An error occurred while deleting applications: {ex.Message}" 
                });
            }
        }

        /// <summary>
        /// Deletes all applications with the specified status ID and their related files (byte arrays in database)
        /// </summary>
        [HttpDelete("delete-by-status/{statusId}")]
        public IActionResult DeleteApplicationsByStatus(int statusId)
        {
            try
            {
                DeletionResult result = _applicationBLL.DeleteApplicationsByStatus(statusId);

                _logger.LogInformation($"Deleted {result.DeletedApplicationsCount} applications and {result.DeletedFilesCount} files with Status ID = {statusId}");

                return Ok(new
                {
                    success = result.Success,
                    message = result.GetMessage(),
                    statusId = statusId,
                    data = new
                    {
                        deletedApplications = result.DeletedApplicationsCount,
                        deletedFiles = result.DeletedFilesCount
                    }
                });
            }
            catch (ArgumentException ex)
            {
                return BadRequest(new { success = false, message = ex.Message });
            }
            catch (Exception ex)
            {
                _logger.LogError(ex, $"Error deleting applications with status {statusId}");
                return StatusCode(500, new 
                { 
                    success = false, 
                    message = $"An error occurred while deleting applications: {ex.Message}" 
                });
            }
        }
    }
}
*/
