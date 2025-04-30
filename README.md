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
