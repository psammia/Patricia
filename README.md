                            table.rows().every(function () {
                                const row = this.node(); // get the actual DOM node of the row

                                const checkbox = $(row).find('#' + row.attributes.id.nodeValue + '.row-checkbox');

                                if (checkbox.is('.row-checkbox:checked')) {
                                    this.remove(); // use DataTables API to remove the row
                                }
                            });
