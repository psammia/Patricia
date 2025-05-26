        public async Task<IActionResult> Index()
        {
            var orders = await _repo.GetAllOrdersAsync();
            return View(orders);
        }

        public async Task<IActionResult> Create()
        {

            ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
            ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
            return View(new Order());
        }

        [HttpPost]
        public async Task<IActionResult> Create(Order order)
        {
            if (ModelState.IsValid)
            {
                await _repo.AddOrderWithCustomersAsync(order);
                return RedirectToAction("Index");
            }

            ViewBag.Customers = await _customerRepo.GetAllCustomersAsync(); 
            ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
            return View(order);
        }

        public async Task<IActionResult> Edit(int id)
        {
            var order = await _repo.GetOrderByIdAsync(id);
            ViewBag.Customers = await _customerRepo.GetAllCustomersAsync();
            ViewBag.Statuses = await _statusRepo.GetAllStatusesAsync();
            return View(order);
        }

        [HttpPost]
        public async Task<IActionResult> Edit(Order order, int[] selectedCustomers)
        {
            await _repo.UpdateOrderWithCustomersAsync(order, selectedCustomers);
            return RedirectToAction("Index");
        }

        public async Task<IActionResult> DeleteConfirmed(int id)
        {
            await _repo.DeleteOrderAsync(id);
            return RedirectToAction("Index");
        }
    }
}
