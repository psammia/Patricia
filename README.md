DECLARE @SearchValue NVARCHAR(100) = 'Value1';

DECLARE @TableName NVARCHAR(256), @ColumnName NVARCHAR(128), @DataType NVARCHAR(128);
DECLARE @sql NVARCHAR(MAX);

DECLARE cur CURSOR FOR
SELECT t.name, c.name, ty.name
FROM sys.tables t
JOIN sys.columns c ON t.object_id = c.object_id
JOIN sys.types ty ON c.user_type_id = ty.user_type_id
WHERE ty.name IN ('char', 'nchar', 'varchar', 'nvarchar', 'text', 'ntext');

OPEN cur;
FETCH NEXT FROM cur INTO @TableName, @ColumnName, @DataType;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @sql = 
        'IF EXISTS (SELECT 1 FROM [' + @TableName + '] WHERE [' + @ColumnName + '] LIKE ''%' + @SearchValue + '%'') 
         PRINT ''Found in table: ' + @TableName + ', column: ' + @ColumnName + '''';

    EXEC sp_executesql @sql;

    FETCH NEXT FROM cur INTO @TableName, @ColumnName, @DataType;
END

CLOSE cur;
DEALLOCATE cur;
