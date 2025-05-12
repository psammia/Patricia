DECLARE @SearchValue NVARCHAR(100) = 'YourSearchValue';
DECLARE @TableName NVARCHAR(256);
DECLARE @ColumnName NVARCHAR(128);
DECLARE @SQL NVARCHAR(MAX);

DECLARE cur CURSOR FOR
SELECT 
    t.name AS TableName,
    c.name AS ColumnName
FROM 
    sys.tables t
JOIN 
    sys.columns c ON t.object_id = c.object_id
JOIN 
    sys.types ty ON c.user_type_id = ty.user_type_id
WHERE 
    ty.name IN ('varchar', 'nvarchar', 'char', 'nchar', 'text');

OPEN cur;
FETCH NEXT FROM cur INTO @TableName, @ColumnName;

WHILE @@FETCH_STATUS = 0
BEGIN
    SET @SQL = 
    'IF EXISTS (SELECT 1 FROM ' + QUOTENAME(@TableName) + 
    ' WHERE ' + QUOTENAME(@ColumnName) + ' LIKE @SearchValue) ' +
    'PRINT ''Found in Table: ' + @TableName + ', Column: ' + @ColumnName + ''';';

    EXEC sp_executesql @SQL, N'@SearchValue NVARCHAR(100)', @SearchValue = '%' + @SearchValue + '%';

    FETCH NEXT FROM cur INTO @TableName, @ColumnName;
END

CLOSE cur;
DEALLOCATE cur;
