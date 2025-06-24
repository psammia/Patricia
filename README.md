        void ComposeTable(IContainer container)
        {
            container.Table(table =>
            {
                table.ColumnsDefinition(columns =>
                {
                    columns.RelativeColumn(1);
                    columns.RelativeColumn(1);
                    columns.RelativeColumn(1);
                    columns.RelativeColumn(1);
                    columns.RelativeColumn(1);
                    columns.RelativeColumn(1);
                    columns.RelativeColumn(1);
                });

                table.Header(header =>
                {
                    header.Cell().Element(CellStyle).Text("Box Ref");
                    header.Cell().Element(CellStyle).Text("Branch");
                    header.Cell().Element(CellStyle).Text("Status");
                    header.Cell().Element(CellStyle).Text("Archiving Date");
                    header.Cell().Element(CellStyle).Text("Destruction Date");
                    header.Cell().Element(CellStyle).Text("Archive Period");
                    header.Cell().Element(CellStyle).Text("Sent By");


                    static IContainer CellStyle(IContainer container)
                    {
                        return container
                            .DefaultTextStyle(x => x.SemiBold().FontSize(9))
                            .Border(1)
                            .BorderColor(Colors.Grey.Lighten1)
                            .Padding(2);
                    }
                });


                foreach (ExportWareouseContainersDto item in ExportWarehouseContainersRes.WarehouseContainersList)
                {
                    table.Cell().Element(CellStyle).Text(item.ContainerCode.ToString());
                    table.Cell().Element(CellStyle).Text(item.Branch);
                    table.Cell().Element(CellStyle).Text(item.StatusCode);
                    table.Cell().Element(CellStyle).Text(item.ArchivingDate.ToString());
                    table.Cell().Element(CellStyle).Text(item.DestructionDate.ToString());
                    table.Cell().Element(CellStyle).Text(item.ArchivingPeriod.ToString());
                    table.Cell().Element(CellStyle).Text(item.SentBy);

                    static IContainer CellStyle(IContainer container)
                    {
                        return container
                            .DefaultTextStyle(x => x.SemiBold().FontSize(9))
                            .Border(1)
                            .BorderColor(Colors.Grey.Lighten1)
                            .Padding(5);
                    }
                }
            });
        }
