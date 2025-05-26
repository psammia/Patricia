public IActionResult Create()
{
    var customers = _yourRepositoryOrService.GetCustomers(); // Get the customers from DB
    ViewBag.Customers = customers.Select(c => new SelectListItem
    {
        Value = c.CustomerId.ToString(),
        Text = c.Name
    }).ToList();

    return View();
}
