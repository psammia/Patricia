public class CustomerRepository
{
    private readonly DatabaseHelper _db;

    public CustomerRepository(DatabaseHelper db)
    {
        _db = db;
    }

    public List<Customer> GetAll()
    {
        var customers = new List<Customer>();
        using var conn = _db.GetConnection();
        conn.Open();
        using var cmd = new SqlCommand("SELECT * FROM Customers", conn);
        using var reader = cmd.ExecuteReader();
        while (reader.Read())
        {
            customers.Add(new Customer
            {
                CustomerId = (int)reader["CustomerId"],
                Name = reader["Name"].ToString(),
                Email = reader["Email"].ToString(),
                Phone = reader["Phone"].ToString()
            });
        }
        return customers;
    }

    public void Add(Customer customer)
    {
        using var conn = _db.GetConnection();
        conn.Open();
        using var cmd = new SqlCommand("INSERT INTO Customers (Name, Email, Phone) VALUES (@Name, @Email, @Phone)", conn);
        cmd.Parameters.AddWithValue("@Name", customer.Name);
        cmd.Parameters.AddWithValue("@Email", customer.Email ?? (object)DBNull.Value);
        cmd.Parameters.AddWithValue("@Phone", customer.Phone ?? (object)DBNull.Value);
        cmd.ExecuteNonQuery();
    }
}



public class DatabaseHelper
{
    private readonly string _connectionString;

    public DatabaseHelper(IConfiguration configuration)
    {
        _connectionString = configuration.GetConnectionString("DefaultConnection");
    }

    public SqlConnection GetConnection() => new SqlConnection(_connectionString);
}




<script>
    function CancelInvoice(invoiceRef) {
        swal({
            title: "",
            text: "Do you really want to delete this invoice?",
            icon: "warning",
            buttons: true,
            dangerMode: true,
        }).then((willDelete) => {
            if (willDelete) {
                $.ajax({
                    url: '/Invoice/CancelPendingInvoiceFromBranch',
                    type: 'POST',
                    data: {
                        invoiceRef: invoiceRef
                    },
                    success: function () {
                        swal("Deleted!", "Record deleted.", "success").then(() => {
                            location.reload();
                        });
                    },
                    error: function () {
                        swal("Error!", "There was a problem deleting the record.", "error");
                    }
                });
            }
        });
    }
</script>




swal("Error", "you have to fill ft value", "error");
sweetalert.min.js


I want to add are you sure with yes no and I want to track the function on yes
