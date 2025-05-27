function getCustomerEntryHtml(i) {
    return `
        <div class="customer-entry mb-3 border p-3 rounded">
            <label>Customer</label>
            <select class="form-select customer-select" name="CustomerOrders[${i}].CustomerId" required>
                <option value="">-- Select Customer --</option>
                ${customerOptionsHtml}
            </select>

            <label class="mt-2">Amount</label>
            <input name="CustomerOrders[${i}].Amount" class="form-control" required />

            <label class="mt-2">No.of Product</label>
            <input name="CustomerOrders[${i}].NoOfProductperCustomer" type="number" class="form-control" required />

            <div class="form-check mt-2">
                <input name="CustomerOrders[${i}].IsPaid" type="hidden" value="false" /> <input name="CustomerOrders[${i}].IsPaid" type="checkbox" value="true" class="form-check-input" />
                <label class="form-check-label">Is Paid</label>
            </div>

            <button type="button" class="btn btn-danger btn-sm mt-2 remove-entry">Remove</button>
        </div>
    `;
}
