-- =============================================
-- Stored Procedure: usp_GetContainerFilesForBackfill
-- Description: Gets container files by ContainerId for backfill process
-- =============================================
USE [Alterna.Archive]
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetContainerFilesForBackfill]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetContainerFilesForBackfill]
GO

CREATE PROCEDURE [dbo].[usp_GetContainerFilesForBackfill]
    @ContainerId INT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        f.Id AS FileId,
        f.Name AS FileName,
        f.CustomerId,
        f.FromDate,
        f.ToDate,
        f.AdditionalInfo,
        f.CompanyCode,
        ft.ArchivingPeriod,
        ft.Description AS FileTypeDescription
    FROM t_Container c
    INNER JOIN t_CurrentContainerFileRelationship ccfr ON c.Id = ccfr.ContainerId
    INNER JOIN t_File f ON ccfr.FileId = f.Id
    INNER JOIN lkp_FileType ft ON f.FileTypeCode = ft.Code
    WHERE c.Id = @ContainerId
        AND c.isDeleted = 0 
        AND f.isDeleted = 0
    ORDER BY f.Name;
END
GO

-- Test the stored procedure
-- EXEC usp_GetContainerFilesForBackfill @ContainerId = 1;

PRINT 'Stored procedure usp_GetContainerFilesForBackfill created successfully!'
GO




-- =============================================
-- COMPLETE SQL SCRIPT FOR PDF ON-DEMAND GENERATION
-- Database: Alterna.Archive
-- Execute this entire script to create all required stored procedures
-- =============================================
USE [Alterna.Archive]
GO

PRINT '========================================';
PRINT 'Starting Stored Procedures Creation';
PRINT '========================================';
GO

-- =============================================
-- 1. Get Container Data for PDF Generation
-- =============================================
PRINT 'Creating usp_GetContainerDataForPDFGeneration...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetContainerDataForPDFGeneration]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetContainerDataForPDFGeneration]
GO

CREATE PROCEDURE [dbo].[usp_GetContainerDataForPDFGeneration]
    @ContainerCode NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        c.Id AS ContainerId,
        c.Code AS ContainerCode,
        c.CompanyCode,
        c.Entity,
        c.ArchivingDate,
        c.StatusCode,
        f.Id AS FileId,
        f.Name AS FileName,
        f.CustomerId,
        f.FromDate,
        f.ToDate,
        f.AdditionalInfo,
        ft.ArchivingPeriod,
        ft.Description AS FileTypeDescription,
        CASE 
            WHEN f.CustomerId IS NOT NULL THEN 'CUSTOMER'
            WHEN c.CompanyCode LIKE 'LB%' THEN 'BRANCH'
            WHEN c.CompanyCode LIKE 'ET%' THEN 'ENTITY'
            ELSE 'UNKNOWN'
        END AS DocumentType
    FROM t_Container c
    INNER JOIN t_CurrentContainerFileRelationship ccfr ON c.Id = ccfr.ContainerId
    INNER JOIN t_File f ON ccfr.FileId = f.Id
    INNER JOIN lkp_FileType ft ON f.FileTypeCode = ft.Code
    WHERE c.Code = @ContainerCode 
        AND c.isDeleted = 0 
        AND f.isDeleted = 0
    ORDER BY f.Name;
END
GO

PRINT 'usp_GetContainerDataForPDFGeneration created successfully!';
GO

-- =============================================
-- 2. Insert/Update PDF
-- =============================================
PRINT 'Creating usp_InsertPDF...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_InsertPDF]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_InsertPDF]
GO

CREATE PROCEDURE [dbo].[usp_InsertPDF]
(
    @PDF VARBINARY(MAX),
    @Request NVARCHAR(MAX),
    @ApiMethod NVARCHAR(500),
    @BranchList NVARCHAR(MAX),
    @Entity NVARCHAR(10),
    @User NVARCHAR(250)
)
AS 
BEGIN
    SET NOCOUNT ON;

    -- Check if PDF already exists for this container
    DECLARE @ContainerID NVARCHAR(50);
    
    -- Extract ContainerID from Request JSON
    SET @ContainerID = JSON_VALUE(@Request, '$.ContainerID');
    
    IF @ContainerID IS NOT NULL
    BEGIN
        -- Check if record exists
        IF EXISTS (
            SELECT 1 
            FROM t_PDF 
            WHERE Request LIKE '%"ContainerID": "' + @ContainerID + '"%'
                AND ApiMethod = @ApiMethod
        )
        BEGIN
            -- Update existing record with new PDF if provided
            IF @PDF IS NOT NULL AND DATALENGTH(@PDF) > 0
            BEGIN
                UPDATE t_PDF
                SET PDF = @PDF,
                    LastModifiedBy = @User,
                    LastModifiedDate = GETDATE()
                WHERE Request LIKE '%"ContainerID": "' + @ContainerID + '"%'
                    AND ApiMethod = @ApiMethod;
            END
        END
        ELSE
        BEGIN
            -- Insert new record
            INSERT INTO t_PDF (
                PDF,
                Request,
                ApiMethod,
                BranchList,
                Entity,
                CreatedBy,
                LastModifiedBy
            )
            VALUES (
                @PDF,
                @Request,
                @ApiMethod,
                @BranchList,
                @Entity,
                @User,
                @User
            );
        END
    END
    ELSE
    BEGIN
        -- If no ContainerID found, just insert
        INSERT INTO t_PDF (
            PDF,
            Request,
            ApiMethod,
            BranchList,
            Entity,
            CreatedBy,
            LastModifiedBy
        )
        VALUES (
            @PDF,
            @Request,
            @ApiMethod,
            @BranchList,
            @Entity,
            @User,
            @User
        );
    END
END
GO

PRINT 'usp_InsertPDF created successfully!';
GO

-- =============================================
-- 3. Get PDF VarBinary by Box Reference
-- =============================================
PRINT 'Creating usp_GetPDFVarBinaryByBoxReference...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetPDFVarBinaryByBoxReference]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetPDFVarBinaryByBoxReference]
GO

CREATE PROCEDURE [dbo].[usp_GetPDFVarBinaryByBoxReference]
    @BoxReference NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT TOP 1 PDF 
    FROM t_PDF 
    WHERE Request LIKE '%"ContainerID": "' + @BoxReference + '"%' 
        AND ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
        AND PDF IS NOT NULL
        AND DATALENGTH(PDF) > 0
    ORDER BY CreatedDate DESC;
END
GO

PRINT 'usp_GetPDFVarBinaryByBoxReference created successfully!';
GO

-- =============================================
-- 4. Get PDF Request by Box Reference
-- =============================================
PRINT 'Creating usp_GetPDFRequestByBoxReference...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetPDFRequestByBoxReference]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetPDFRequestByBoxReference]
GO

CREATE PROCEDURE [dbo].[usp_GetPDFRequestByBoxReference]
    @BoxReference NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT TOP 1 Request, ApiMethod
    FROM t_PDF 
    WHERE Request LIKE '%"ContainerID": "' + @BoxReference + '"%' 
        AND ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
    ORDER BY CreatedDate DESC;
END
GO

PRINT 'usp_GetPDFRequestByBoxReference created successfully!';
GO

-- =============================================
-- 5. Get Entity by Code
-- =============================================
PRINT 'Creating usp_GetEntityByCode...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetEntityByCode]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetEntityByCode]
GO

CREATE PROCEDURE [dbo].[usp_GetEntityByCode]
    @EntityCode NVARCHAR(11)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        Code,
        Description,
        HasFullAccess,
        Category
    FROM lkp_Entity
    WHERE Code = @EntityCode;
END
GO

PRINT 'usp_GetEntityByCode created successfully!';
GO

-- =============================================
-- 6. Check if PDF Exists for Container
-- =============================================
PRINT 'Creating usp_CheckPDFExistsForContainer...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_CheckPDFExistsForContainer]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_CheckPDFExistsForContainer]
GO

CREATE PROCEDURE [dbo].[usp_CheckPDFExistsForContainer]
    @ContainerCode NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        CASE 
            WHEN EXISTS (
                SELECT 1 
                FROM t_PDF 
                WHERE Request LIKE '%"ContainerID": "' + @ContainerCode + '"%'
                    AND ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
            ) 
            THEN 1 
            ELSE 0 
        END AS PDFExists,
        CASE 
            WHEN EXISTS (
                SELECT 1 
                FROM t_PDF 
                WHERE Request LIKE '%"ContainerID": "' + @ContainerCode + '"%'
                    AND ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
                    AND PDF IS NOT NULL
                    AND DATALENGTH(PDF) > 0
            ) 
            THEN 1 
            ELSE 0 
        END AS PDFBinaryExists;
END
GO

PRINT 'usp_CheckPDFExistsForContainer created successfully!';
GO

-- =============================================
-- 7. Get Containers Without PDF
-- =============================================
PRINT 'Creating usp_GetContainersWithoutPDF...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetContainersWithoutPDF]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetContainersWithoutPDF]
GO

CREATE PROCEDURE [dbo].[usp_GetContainersWithoutPDF]
    @FromDate DATETIME = NULL,
    @ToDate DATETIME = NULL
AS
BEGIN
    SET NOCOUNT ON;

    SELECT DISTINCT 
        c.Id,
        c.Code,
        c.CompanyCode,
        c.Entity,
        c.StatusCode,
        c.ArchivingDate,
        c.CreatedDate,
        c.CurrentLocation
    FROM t_Container c
    WHERE c.StatusCode IN ('SENT', 'RECEIVED', 'TOBEDESTR', 'DESTROYED')
        AND c.isDeleted = 0
        AND NOT EXISTS (
            SELECT 1 
            FROM t_PDF p 
            WHERE p.Request LIKE '%"ContainerID": "' + c.Code + '"%'
                AND p.ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
        )
        AND (@FromDate IS NULL OR c.ArchivingDate >= @FromDate)
        AND (@ToDate IS NULL OR c.ArchivingDate <= @ToDate)
    ORDER BY c.CreatedDate DESC;
END
GO

PRINT 'usp_GetContainersWithoutPDF created successfully!';
GO

-- =============================================
-- 8. Get Customer Files by Container
-- =============================================
PRINT 'Creating usp_GetCustomerFilesByContainer...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetCustomerFilesByContainer]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetCustomerFilesByContainer]
GO

CREATE PROCEDURE [dbo].[usp_GetCustomerFilesByContainer]
    @ContainerCode NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        f.Name AS DocumentType,
        f.CustomerId,
        CAST(f.CustomerId AS NVARCHAR(50)) AS CustomerIdString
    FROM t_Container c
    INNER JOIN t_CurrentContainerFileRelationship ccfr ON c.Id = ccfr.ContainerId
    INNER JOIN t_File f ON ccfr.FileId = f.Id
    WHERE c.Code = @ContainerCode 
        AND c.isDeleted = 0 
        AND f.isDeleted = 0
        AND f.CustomerId IS NOT NULL
    ORDER BY f.Name, f.CustomerId;
END
GO

PRINT 'usp_GetCustomerFilesByContainer created successfully!';
GO

-- =============================================
-- 9. Get Container Sent By User (by ContainerId)
-- =============================================
PRINT 'Creating usp_GetContainerSentByUser...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetContainerSentByUser]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetContainerSentByUser]
GO

CREATE PROCEDURE [dbo].[usp_GetContainerSentByUser]
    @ContainerId INT
AS
BEGIN
    SET NOCOUNT ON;

    -- Get the user who changed status to SENT
    SELECT TOP 1 
        cs.CreatedBy AS SentByUser,
        cs.CreatedDate AS SentDate,
        cs.HoldingEntityCode
    FROM t_ContainerStatus cs
    WHERE cs.ContainerId = @ContainerId
        AND cs.StatusCode = 'SENT'
    ORDER BY cs.CreatedDate DESC;
    
    -- If no SENT status found, get the last user who modified the container
    IF @@ROWCOUNT = 0
    BEGIN
        SELECT 
            c.LastModifiedBy AS SentByUser,
            c.LastModifiedDate AS SentDate,
            c.Entity AS HoldingEntityCode
        FROM t_Container c
        WHERE c.Id = @ContainerId;
    END
END
GO

PRINT 'usp_GetContainerSentByUser created successfully!';
GO

-- =============================================
-- 10. Get Container Sent By User (by Container Code)
-- =============================================
PRINT 'Creating usp_GetContainerSentByUserByCode...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetContainerSentByUserByCode]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetContainerSentByUserByCode]
GO

CREATE PROCEDURE [dbo].[usp_GetContainerSentByUserByCode]
    @ContainerCode NVARCHAR(50)
AS
BEGIN
    SET NOCOUNT ON;

    -- Get the user who changed status to SENT based on container code
    SELECT TOP 1 
        cs.CreatedBy AS SentByUser,
        cs.CreatedDate AS SentDate,
        cs.HoldingEntityCode,
        c.Id AS ContainerId
    FROM t_Container c
    INNER JOIN t_ContainerStatus cs ON c.Id = cs.ContainerId
    WHERE c.Code = @ContainerCode
        AND cs.StatusCode = 'SENT'
    ORDER BY cs.CreatedDate DESC;
    
    -- If no SENT status found, get the last user who modified the container
    IF @@ROWCOUNT = 0
    BEGIN
        SELECT 
            c.LastModifiedBy AS SentByUser,
            c.LastModifiedDate AS SentDate,
            c.Entity AS HoldingEntityCode,
            c.Id AS ContainerId
        FROM t_Container c
        WHERE c.Code = @ContainerCode;
    END
END
GO

PRINT 'usp_GetContainerSentByUserByCode created successfully!';
GO

-- =============================================
-- 11. Get Container Files for Backfill
-- =============================================
PRINT 'Creating usp_GetContainerFilesForBackfill...';
GO

IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetContainerFilesForBackfill]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetContainerFilesForBackfill]
GO

CREATE PROCEDURE [dbo].[usp_GetContainerFilesForBackfill]
    @ContainerId INT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        f.Id AS FileId,
        f.Name AS FileName,
        f.CustomerId,
        f.FromDate,
        f.ToDate,
        f.AdditionalInfo,
        f.CompanyCode,
        ft.ArchivingPeriod,
        ft.Description AS FileTypeDescription
    FROM t_Container c
    INNER JOIN t_CurrentContainerFileRelationship ccfr ON c.Id = ccfr.ContainerId
    INNER JOIN t_File f ON ccfr.FileId = f.Id
    INNER JOIN lkp_FileType ft ON f.FileTypeCode = ft.Code
    WHERE c.Id = @ContainerId
        AND c.isDeleted = 0 
        AND f.isDeleted = 0
    ORDER BY f.Name;
END
GO

PRINT 'usp_GetContainerFilesForBackfill created successfully!';
GO

-- =============================================
-- Verification: List all created stored procedures
-- =============================================
PRINT '';
PRINT '========================================';
PRINT 'Verification: All Created Procedures';
PRINT '========================================';

SELECT 
    name AS ProcedureName,
    create_date AS CreatedDate,
    modify_date AS ModifiedDate
FROM sys.procedures 
WHERE name IN (
    'usp_GetContainerDataForPDFGeneration',
    'usp_InsertPDF',
    'usp_GetPDFVarBinaryByBoxReference',
    'usp_GetPDFRequestByBoxReference',
    'usp_GetEntityByCode',
    'usp_CheckPDFExistsForContainer',
    'usp_GetContainersWithoutPDF',
    'usp_GetCustomerFilesByContainer',
    'usp_GetContainerSentByUser',
    'usp_GetContainerSentByUserByCode',
    'usp_GetContainerFilesForBackfill'
)
ORDER BY name;

PRINT '';
PRINT '========================================';
PRINT 'Total Procedures Created: 11';
PRINT 'Script completed successfully!';
PRINT '========================================';
GO
