public IActionResult Create()
{
    using (var connection = new SqlConnection(_connectionString))
    {
        var customers = connection.Query<Customer>("SELECT CustomerId, Name FROM Customers").ToList();

        ViewBag.Customers = customers.Select(c => new SelectListItem
        {
            Value = c.CustomerId.ToString(),
            Text = c.Name
        }).ToList();

        return View();
    }
}
