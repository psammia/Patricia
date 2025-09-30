CREATE OR ALTER PROCEDURE dbo.InsertApplicationWithFiles
(
    @CorrelationId NVARCHAR(250),
    @External_Id NVARCHAR(10),
    @Files dbo.FileListType READONLY
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
            @External_Id,
            @CorrelationId,
            1,                -- Always StatusId = 1
            SUSER_SNAME(),    -- Or 'AlternaSysUser'
            GETDATE(),
            SUSER_SNAME(),
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
            SUSER_SNAME(),
            GETDATE(),
            SUSER_SNAME(),
            GETDATE()
        FROM @Files f;

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
GO
====================================




============================================
DECLARE @FileList dbo.FileListType;

INSERT INTO @FileList (File_Name, File_Type, File_Size, File_Data)
VALUES
('Doc1.pdf', 'application/pdf', 12345, 0x01010101),
('Image1.jpg', 'image/jpeg', 54321, 0x02020202);

EXEC dbo.InsertApplicationWithFiles
    @CorrelationId = 'ABC123',
    @External_Id = 'EXT001',
    @Files = @FileList;
