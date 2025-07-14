USE [Alterna.Loyalty]
GO
/****** Object:  StoredProcedure [dbo].[Get_NonExpiredPoints_ByCustomer]    Script Date: 14/07/2025 9:46:58 AM ******/
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
-- =============================================
-- Author:		<Kamal Abbas>
-- Create date: <12/07/2025>
-- Description:	<Get Non Expired Points By Customer List>
-- =============================================
CREATE or ALTER PROCEDURE [dbo].[Get_NonExpiredPoints_ByCustomerList]
    @Customer_Id_List NVARCHAR(MAX)
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @ExpirationYears INT;

    -- Fetch the expiration years from config
    SELECT TOP 1 
        @ExpirationYears = CAST(LTRIM(RTRIM(SettingValue)) AS INT)
    FROM dbo.t_Config
    WHERE SettingName = 'ExpirationYears' AND IsActive = 1;

    IF @ExpirationYears IS NULL
    BEGIN
        RAISERROR('Expiration configuration not found or invalid.', 16, 1);
        RETURN;
    END

    -- Sum of non-expired points from t_Transactions
    SELECT 
        Customer_Id, SUM(ISNULL(Points, 0)) AS Total_points, 
    FROM 
        dbo.t_Transactions
    WHERE 
        Customer_Id IN 
			(SELECT VALUE FROM STRING_SPLIT(@Customer_Id_List, ','))
        AND GETDATE() <= EOMONTH(DATEADD(YEAR, @ExpirationYears, CreatedDate))
	GROUP BY Customer_Id;
END
