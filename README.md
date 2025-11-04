-- =============================================
-- 1. Get Container with Files Data for PDF Generation
-- =============================================
USE [Alterna.Archive]
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

-- =============================================
-- 2. Modified usp_InsertPDF to handle both empty and full varbinary
-- =============================================
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

-- =============================================
-- 3. Enhanced usp_GetPDFVarBinaryByBoxReference
-- =============================================
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

-- =============================================
-- 4. Enhanced usp_GetPDFRequestByBoxReference
-- =============================================
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

-- =============================================
-- 5. Get Entity by Code
-- =============================================
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

-- =============================================
-- 6. Check if PDF Exists for Container
-- =============================================
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

-- =============================================
-- 7. Get all containers without PDFs (for reporting/backfill)
-- =============================================
IF EXISTS (SELECT * FROM sys.objects WHERE object_id = OBJECT_ID(N'[dbo].[usp_GetContainersWithoutPDF]') AND type in (N'P', N'PC'))
DROP PROCEDURE [dbo].[usp_GetContainersWithoutPDF]
GO

CREATE PROCEDURE [dbo].[usp_GetContainersWithoutPDF]
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
        c.CreatedDate
    FROM t_Container c
    WHERE c.StatusCode IN ('SENT', 'RECEIVED', 'TOBEDESTR', 'DESTROYED')
        AND c.isDeleted = 0
        AND NOT EXISTS (
            SELECT 1 
            FROM t_PDF p 
            WHERE p.Request LIKE '%"ContainerID": "' + c.Code + '"%'
                AND p.ApiMethod IN ('GenerateBranchDocPDFForArchive','GenerateCustomerDocPDFForArchive', 'GenerateEntityDocPDFForArchive')
        )
    ORDER BY c.CreatedDate DESC;
END
GO

-- =============================================
-- 8. Get Customer IDs grouped by File Type
-- =============================================
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

-- =============================================
-- 9. Get User who sent the container (Status = SENT)
-- =============================================
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

-- =============================================
-- 10. Get User by Container Code (for legacy containers)
-- =============================================
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

PRINT 'All stored procedures created successfully!'
GO
