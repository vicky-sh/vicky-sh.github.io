---
date: "2025-05-01T13:55:09+02:00"
draft: true
title: "Controllers"
cascade:
  type: docs
---

In ASP.NET Core, a controller is a class that handles incoming HTTP requests, processes them (often with the help of a model or service), and returns a response—usually a View, JSON, or a status code.

Controllers are a core part of the MVC pattern (Model-View-Controller) and are typically located in the Controllers folder of your project.

A controller:

- Inherits from Controller or ControllerBase
- Contains action methods—each mapped to an HTTP request like GET, POST, PUT, DELETE
- Uses attributes like [HttpGet], [HttpPost], [Route] to define routing behavior

```csharp
using Microsoft.AspNetCore.Mvc;
using System.Collections.Generic;

namespace MyApp.Controllers
{
    [ApiController]
    [Route("api/[controller]")]
    public class ProductController : ControllerBase
    {
        // Mock data
        private static List<string> products = new List<string>
        {
            "Laptop", "Mouse", "Keyboard"
        };

        // GET: api/product
        [HttpGet]
        public ActionResult<List<string>> GetAll()
        {
            return products;
        }

        // GET: api/product/1
        [HttpGet("{id}")]
        public ActionResult<string> GetById(int id)
        {
            if (id < 0 || id >= products.Count)
                return NotFound("Product not found");
            
            return products[id];
        }

        // POST: api/product
        [HttpPost]
        public ActionResult AddProduct([FromBody] string product)
        {
            products.Add(product);
            return Ok("Product added");
        }
    }
}
```