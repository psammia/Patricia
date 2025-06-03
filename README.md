🧠 3. JavaScript: DataTables + Checkbox Handling
Inside the partial view (or better: move to layout if reused), update your script:

html
Copy
Edit
<script>
    $(document).ready(() => {
        const table = $("#TblContainertoNotifyWarehouseTable").DataTable({
            pagingType: 'full_numbers',
            scrollX: true
        });

        // Check all functionality
        $('#checkAllBoxes').on('change', function () {
            $('.row-checkbox').prop('checked', this.checked);
            toggleNotifyButton();
        });

        // Individual checkbox toggle
        $(document).on('change', '.row-checkbox', function () {
            const allChecked = $('.row-checkbox').length === $('.row-checkbox:checked').length;
            $('#checkAllBoxes').prop('checked', allChecked);
            toggleNotifyButton();
        });

        // Enable/Disable Notify button
        function toggleNotifyButton() {
            const selectedCount = $('.row-checkbox:checked').length;
            $('#notifyButton').prop('disabled', selectedCount === 0);
        }
    });
</script>
✅ Optional: Get selected IDs on button click
Add this if you want to collect checked IDs when clicking "Notify":

html
Copy
Edit
<script>
    $('#notifyButton').on('click', function () {
        const selectedIds = $('.row-checkbox:checked').map(function () {
            return $(this).val();
        }).get();

        console.log("Selected IDs:", selectedIds);
        // Submit via AJAX or redirect as needed
    });
</script>
🧪 Result
Checkbox in each row and a header "Select All" checkbox.

Notify button stays disabled until at least one box is checked.

Button is placed top-right as requested.

Fully compatible with Bootstrap 5.3.6 and DataTables v2.3.1.

Would you like the Notify button to trigger an AJAX POST with the selected box IDs?
