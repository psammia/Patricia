@section Scripts {
    <partial name="_ValidationScriptsPartial" />
    <script>
        let index = 1;

        // Render options once
        const customerOptions = `@foreach (var c in customers) {<text><option value="@c.CustomerId">@c.Name</option></text>}`.trim();

        $('#addCustomer').on('click', function () {
            let newEntry = `
                <div class="customer-entry mb-2">
                    <select class="form-select customer-select" name="CustomerOrders[${index}].CustomerId">
                        <option value="">-- Select Customer --</option>
                        ${customerOptions}
                    </select>
                    <input name="CustomerOrders[${index}].Amount" type="number" placeholder="Amount" class="form-control d-inline-block mx-1" style="width: 150px;" />
                    <input name="CustomerOrders[${index}].IsPaid" type="checkbox" class="form-check-input" />
                    <button type="button" class="btn btn-danger btn-sm remove-entry">×</button>
                </div>`;
            $('#customerRepeater').append(newEntry);
            index++;
        });

        $(document).on('click', '.remove-entry', function () {
            $(this).closest('.customer-entry').remove();
        });

        $(document).ready(function () {
            $('.customer-select').select2();
        });
    </script>
}
