-- Stored Procedure to delete all applications with StatusId and their related files
-- ================================================
-- Stored Procedure: sp_DeleteApplicationsByStatus
-- Description: Deletes all applications with a specific StatusId 
--              and their related files (stored as byte arrays)
-- ================================================
CREATE PROCEDURE sp_DeleteApplicationsByStatus
    @StatusId INT,
    @DeletedCount INT OUTPUT,
    @DeletedFilesCount INT OUTPUT
AS
BEGIN
    SET NOCOUNT ON;
    
    BEGIN TRY
        BEGIN TRANSACTION;
        
        -- Get count of applications to be deleted
        SELECT @DeletedCount = COUNT(*)
        FROM Applications
        WHERE StatusId = @StatusId;
        
        -- Get count of files to be deleted
        SELECT @DeletedFilesCount = COUNT(*)
        FROM appfiles
        WHERE ApplicationId IN (SELECT ApplicationId FROM Applications WHERE StatusId = @StatusId);
        
        -- Delete files from appfiles table first (foreign key constraint)
        -- This will remove all file records including the byte arrays
        DELETE FROM appfiles
        WHERE ApplicationId IN (SELECT ApplicationId FROM Applications WHERE StatusId = @StatusId);
        
        -- Delete other related records if any (add more as needed)
        -- Example: DELETE FROM ApplicationDocuments WHERE ApplicationId IN (SELECT ApplicationId FROM Applications WHERE StatusId = @StatusId)
        -- Example: DELETE FROM ApplicationNotes WHERE ApplicationId IN (SELECT ApplicationId FROM Applications WHERE StatusId = @StatusId)
        
        -- Delete applications
        DELETE FROM Applications
        WHERE StatusId = @StatusId;
        
        COMMIT TRANSACTION;
        
        RETURN 0; -- Success
    END TRY
    BEGIN CATCH
        IF @@TRANCOUNT > 0
            ROLLBACK TRANSACTION;
            
        -- Return error
        DECLARE @ErrorMessage NVARCHAR(4000) = ERROR_MESSAGE();
        DECLARE @ErrorSeverity INT = ERROR_SEVERITY();
        DECLARE @ErrorState INT = ERROR_STATE();
        
        RAISERROR(@ErrorMessage, @ErrorSeverity, @ErrorState);
        RETURN -1; -- Error
    END CATCH
END
GO

-- ================================================
-- Example Usage:
-- ================================================
-- DECLARE @DeletedApps INT, @DeletedFiles INT
-- EXEC sp_DeleteApplicationsByStatus @StatusId = 2, @DeletedCount = @DeletedApps OUTPUT, @DeletedFilesCount = @DeletedFiles OUTPUT
-- SELECT @DeletedApps AS DeletedApplications, @DeletedFiles AS DeletedFiles
-- GO
