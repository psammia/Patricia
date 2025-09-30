DECLARE @FileList dbo.FileListType;

INSERT INTO @FileList (File_Name, File_Type, File_Size, File_Data)
VALUES
('Doc1.pdf', 'application/pdf', 12345, 0x01010101),
('Image1.jpg', 'image/jpeg', 54321, 0x02020202);

EXEC dbo.InsertApplicationWithFiles
    @CorrelationId = 'ABC123',
    @External_Id = 'EXT001',
    @Files = @FileList;
