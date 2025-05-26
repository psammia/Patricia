        public async Task<IActionResult> Create()
        {
            ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
            var customers = _customerRepo.GetAllCustomersAsync(); // Get the customers from DB
            ViewBag.Customers = customers.Select(c => new SelectListItem
            {
                Value = c.CustomerId.ToString(),
                Text = c.Name
            }).ToList();

            return View(new Order());
        }
Severity	Code	Description	Project	File	Line	Suppression State
Error	CS1061	'Task<IEnumerable<Customer>>' does not contain a definition for 'Select' and no accessible extension method 'Select' accepting a first argument of type 'Task<IEnumerable<Customer>>' could be found (are you missing a using directive or an assembly reference?)	OrdersTracking	D:\@Workspace\deve-repo\OrdersTracking\Controllers\OrderController.cs	41	Active


        
