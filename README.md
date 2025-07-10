CREATE OR ALTER PROCEDURE Get_NonExpiredPoints_ByCustomer
    @Customer_Id INT
AS
BEGIN
    SET NOCOUNT ON;

    DECLARE @ExpirationYears INT;

    -- Get the dynamic expiration duration from config
    SELECT TOP 1 
        @ExpirationYears = CAST(LTRIM(RTRIM(SettingValue)) AS INT)
    FROM dbo.t_Config
    WHERE SettingName = 'ExpirationYears' AND IsActive = 1;

    IF @ExpirationYears IS NULL
    BEGIN
        RAISERROR('Expiration configuration not found or invalid.', 16, 1);
        RETURN;
    END

    -- Return sum of non-expired points
    SELECT 
        SUM(ISNULL(Points, 0)) AS TotalNonExpiredPoints
    FROM 
        dbo.t_Transactions
    WHERE 
        Customer_Id = @Customer_Id
        AND GETDATE() <= EOMONTH(DATEADD(YEAR, @ExpirationYears, CreatedDate));
END
