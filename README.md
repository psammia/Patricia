INSERT INTO [dbo].[t_Container]
    ([Code]
    ,[CompanyCode]
    ,[Entity]
    ,[CurrentLocation]
    ,[StatusCode]
    ,[ArchivingDate]
    ,[isDeleted]
    ,[CreatedBy]
    ,[CreatedDate]
    ,[LastModifiedBy]
    ,[LastModifiedDate]
    ,[isNotified])
OUTPUT
    d.RowId, inserted.Id
INTO @InsertedContainers(RowId, ContainerId)
SELECT 
    d.BoxRef, 
    d.Code,
    d.CompanyName,
    '', 
    d.StatusCode, 
    d.BoxSentDate, 
    0,
    @User, 
    @Now, 
    @User, 
    @Now, 
    1
FROM (
    SELECT src.RowId, src.BoxRef, src.Code, src.CompanyName, src.StatusCode, src.BoxSentDate
    FROM @MyInputTableType src
) d;
