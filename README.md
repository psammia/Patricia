DECLARE @SearchValue NVARCHAR(100) = 'YourSearchValue';

SELECT 
    'SELECT ''' + t.name + ''' AS TableName, ''' + c.name + ''' AS ColumnName FROM [' + t.name + '] WHERE [' + c.name + '] LIKE ''%' + @SearchValue + '%''' AS SearchQuery
FROM 
    sys.tables t
JOIN 
    sys.columns c ON t.object_id = c.object_id
JOIN 
    sys.types ty ON c.user_type_id = ty.user_type_id
WHERE 
    ty.name IN ('varchar', 'nvarchar', 'char', 'nchar', 'text');
