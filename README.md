using System.ComponentModel.DataAnnotations;
using System.Globalization;

using Microsoft.VisualBasic;

namespace OrdersTracking.Models
{
    public class Order
    {
        public int OrderId { get; set; }
        [Required]
        [DataType(DataType.Date)]
        [DisplayFormat(DataFormatString = "{0:yyyy-MM-dd}", ApplyFormatInEditMode = true)]
        public DateTime OrderDate { get; set; }
        [Required]
        public decimal? Cost { get; set; }
        [Required]
        public decimal? Profit { get; set; }
        [Required]
        public Int32? NoOfProduct { get; set; }
        [Required]
        public decimal? TotalAmount { get; set; }
      
        [Required]
        public string StatusCode { get; set; } = string.Empty;
        
        public List<CustomerOrder>? CustomerOrders { get; set; }
    }
}
