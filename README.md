using System;
using YourNamespace.DAL;

namespace YourNamespace.BLL
{
    public class ApplicationBLL
    {
        private readonly ApplicationDAL _applicationDAL;

        public ApplicationBLL()
        {
            _applicationDAL = new ApplicationDAL();
        }

        public ApplicationBLL(ApplicationDAL applicationDAL)
        {
            _applicationDAL = applicationDAL;
        }

        /// <summary>
        /// Deletes all applications with the specified status ID along with their files (stored as byte arrays in database)
        /// </summary>
        /// <param name="statusId">The status ID to filter applications</param>
        /// <returns>Deletion result with counts</returns>
        public DeletionResult DeleteApplicationsByStatus(int statusId)
        {
            try
            {
                // Business validation
                if (statusId <= 0)
                {
                    throw new ArgumentException("Status ID must be greater than zero.", nameof(statusId));
                }

                // Delete from database (applications and appfiles records with byte arrays)
                var (deletedApps, deletedFiles) = _applicationDAL.DeleteApplicationsByStatus(statusId);

                // You can add additional business logic here
                // For example: Log the deletion, send notifications, update audit logs, etc.

                return new DeletionResult
                {
                    DeletedApplicationsCount = deletedApps,
                    DeletedFilesCount = deletedFiles,
                    Success = true
                };
            }
            catch (ArgumentException ex)
            {
                throw;
            }
            catch (Exception ex)
            {
                throw new Exception($"Business logic error while deleting applications: {ex.Message}", ex);
            }
        }

        /// <summary>
        /// Deletes all applications with Status ID = 2 (specific method)
        /// </summary>
        /// <returns>Deletion result</returns>
        public DeletionResult DeleteApplicationsWithStatusTwo()
        {
            return DeleteApplicationsByStatus(2);
        }
    }

    /// <summary>
    /// Result class for deletion operations
    /// </summary>
    public class DeletionResult
    {
        public bool Success { get; set; }
        public int DeletedApplicationsCount { get; set; }
        public int DeletedFilesCount { get; set; }

        public string GetMessage()
        {
            return $"Successfully deleted {DeletedApplicationsCount} application(s) and {DeletedFilesCount} file(s) from database.";
        }
    }
}
