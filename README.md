CREATE PROCEDURE [dbo].[Get_CustomerCards_ByCustomerId]
    @Customer_Id INT
AS
BEGIN
    SET NOCOUNT ON;

    SELECT 
        c.Customer_Id,
        c.CardNumber,
        ot.Code AS OperationTypeCode,
        CAST(c.CreatedDate AS DATE) AS CreatedDate
    FROM 
        dbo.t_Cards c
    INNER JOIN 
        dbo.t_OperationTypes ot ON c.OperationType_Id = ot.OperationType_Id
    WHERE 
        c.Customer_Id = @Customer_Id
    ORDER BY 
        c.CreatedDate DESC;
END
