# Building a Production-Ready Customer Management API with Spring Boot: A Complete Guide

*Part 1 of 9 in the series: "Production-Ready Spring Boot on OCI with Vault"*

![Spring Boot REST API Header](https://images.unsplash.com/photo-1555066931-4365d14bab8c?w=1200&h=400&fit=crop)

## What You'll Build

In this tutorial, you'll create a real-world **Customer Management API** for a multi-tenant SaaS platform. This is the foundation for our series where we'll eventually deploy this application to Oracle Cloud Infrastructure (OCI) with enterprise-grade security using HashiCorp Vault.

By the end of this post, you'll have:
- ✅ A fully functional REST API with CRUD operations
- ✅ Clean architecture following Spring Boot best practices
- ✅ Input validation and error handling
- ✅ In-memory H2 database for quick local testing
- ✅ Complete test coverage with Postman collection

**Time to complete**: 45 minutes  
**Difficulty**: Beginner to Intermediate

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project Setup](#project-setup)
3. [Understanding the Domain Model](#understanding-the-domain-model)
4. [Building the Customer Entity](#building-the-customer-entity)
5. [Creating the Repository Layer](#creating-the-repository-layer)
6. [Implementing Business Logic](#implementing-business-logic)
7. [Building REST Controllers](#building-rest-controllers)
8. [Adding Validation and Error Handling](#adding-validation-and-error-handling)
9. [Testing Your API](#testing-your-api)
10. [What's Next](#whats-next)

---

## Prerequisites

Before we begin, make sure you have:

- **Java 17 or higher** installed ([Download here](https://adoptium.net/))
- **Maven 3.8+** or Gradle 7+ ([Maven install guide](https://maven.apache.org/install.html))
- **Your favorite IDE** (IntelliJ IDEA, VS Code, or Eclipse)
- **Postman or curl** for API testing
- Basic understanding of Java and REST APIs

Check your Java version:
```bash
java -version
# Should show: openjdk version "17.0.x" or higher
```

---

## Project Setup

### Using Spring Initializr

We'll use Spring Initializr to bootstrap our project quickly:

1. Go to [start.spring.io](https://start.spring.io)
2. Configure your project:
   - **Project**: Maven
   - **Language**: Java
   - **Spring Boot**: 3.2.1 (or latest stable)
   - **Group**: com.example
   - **Artifact**: customer-api
   - **Name**: customer-api
   - **Package name**: com.example.customerapi
   - **Packaging**: Jar
   - **Java**: 17

3. Add these dependencies:
   - **Spring Web** - For building REST APIs
   - **Spring Data JPA** - For database operations
   - **H2 Database** - In-memory database for development
   - **Validation** - For input validation
   - **Lombok** - To reduce boilerplate code

4. Click **Generate** to download the project

### Manual Setup Alternative

If you prefer command line:

```bash
curl https://start.spring.io/starter.zip \
  -d dependencies=web,data-jpa,h2,validation,lombok \
  -d type=maven-project \
  -d language=java \
  -d bootVersion=3.2.1 \
  -d groupId=com.example \
  -d artifactId=customer-api \
  -d name=customer-api \
  -d packageName=com.example.customerapi \
  -d javaVersion=17 \
  -o customer-api.zip

unzip customer-api.zip
cd customer-api
```

### Project Structure

Your project should look like this:

```
customer-api/
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/customerapi/
│   │   │       ├── CustomerApiApplication.java
│   │   │       ├── controller/
│   │   │       ├── service/
│   │   │       ├── repository/
│   │   │       ├── model/
│   │   │       └── exception/
│   │   └── resources/
│   │       └── application.properties
│   └── test/
├── pom.xml
└── README.md
```

---

## Understanding the Domain Model

Our Customer Management API will handle two main entities:

### Customer
Represents a customer account in our SaaS platform.

**Fields**:
- `id` - Unique identifier (auto-generated)
- `email` - Customer email (unique)
- `companyName` - Company name
- `contactName` - Contact person name
- `phone` - Contact phone number
- `status` - Account status (ACTIVE, SUSPENDED, CLOSED)
- `createdAt` - Registration timestamp
- `updatedAt` - Last update timestamp

### Invoice
Represents an invoice issued to a customer.

**Fields**:
- `id` - Unique identifier
- `customerId` - Reference to customer
- `invoiceNumber` - Human-readable invoice number
- `amount` - Invoice amount
- `currency` - Currency code (USD, EUR, etc.)
- `status` - Payment status (PENDING, PAID, OVERDUE, CANCELLED)
- `issuedAt` - Invoice issue date
- `dueAt` - Payment due date

---

## Building the Customer Entity

Create `src/main/java/com/example/customerapi/model/Customer.java`:

```java
package com.example.customerapi.model;

import jakarta.persistence.*;
import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.Size;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;
import org.hibernate.annotations.UpdateTimestamp;

import java.time.LocalDateTime;

@Entity
@Table(name = "customers")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Customer {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Email(message = "Email should be valid")
    @NotBlank(message = "Email is required")
    @Column(unique = true, nullable = false)
    private String email;

    @NotBlank(message = "Company name is required")
    @Size(min = 2, max = 100, message = "Company name must be between 2 and 100 characters")
    @Column(nullable = false)
    private String companyName;

    @NotBlank(message = "Contact name is required")
    @Size(min = 2, max = 100, message = "Contact name must be between 2 and 100 characters")
    @Column(nullable = false)
    private String contactName;

    @Size(max = 20, message = "Phone number cannot exceed 20 characters")
    private String phone;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private CustomerStatus status = CustomerStatus.ACTIVE;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @UpdateTimestamp
    @Column(nullable = false)
    private LocalDateTime updatedAt;

    public enum CustomerStatus {
        ACTIVE,
        SUSPENDED,
        CLOSED
    }
}
```

### Create the Invoice Entity

Create `src/main/java/com/example/customerapi/model/Invoice.java`:

```java
package com.example.customerapi.model;

import jakarta.persistence.*;
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.hibernate.annotations.CreationTimestamp;

import java.math.BigDecimal;
import java.time.LocalDate;
import java.time.LocalDateTime;

@Entity
@Table(name = "invoices")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Invoice {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @NotNull(message = "Customer ID is required")
    @Column(nullable = false)
    private Long customerId;

    @NotBlank(message = "Invoice number is required")
    @Column(unique = true, nullable = false)
    private String invoiceNumber;

    @NotNull(message = "Amount is required")
    @DecimalMin(value = "0.01", message = "Amount must be greater than zero")
    @Column(nullable = false)
    private BigDecimal amount;

    @NotBlank(message = "Currency is required")
    @Column(nullable = false, length = 3)
    private String currency = "USD";

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private InvoiceStatus status = InvoiceStatus.PENDING;

    @CreationTimestamp
    @Column(nullable = false, updatable = false)
    private LocalDateTime issuedAt;

    @NotNull(message = "Due date is required")
    @Column(nullable = false)
    private LocalDate dueAt;

    public enum InvoiceStatus {
        PENDING,
        PAID,
        OVERDUE,
        CANCELLED
    }
}
```

---

## Creating the Repository Layer

Spring Data JPA makes database operations trivial. Create repository interfaces:

### Customer Repository

Create `src/main/java/com/example/customerapi/repository/CustomerRepository.java`:

```java
package com.example.customerapi.repository;

import com.example.customerapi.model.Customer;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface CustomerRepository extends JpaRepository<Customer, Long> {
    
    Optional<Customer> findByEmail(String email);
    
    List<Customer> findByStatus(Customer.CustomerStatus status);
    
    boolean existsByEmail(String email);
}
```

### Invoice Repository

Create `src/main/java/com/example/customerapi/repository/InvoiceRepository.java`:

```java
package com.example.customerapi.repository;

import com.example.customerapi.model.Invoice;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.stereotype.Repository;

import java.util.List;
import java.util.Optional;

@Repository
public interface InvoiceRepository extends JpaRepository<Invoice, Long> {
    
    List<Invoice> findByCustomerId(Long customerId);
    
    List<Invoice> findByStatus(Invoice.InvoiceStatus status);
    
    Optional<Invoice> findByInvoiceNumber(String invoiceNumber);
    
    boolean existsByInvoiceNumber(String invoiceNumber);
}
```

---

## Implementing Business Logic

Now let's create service classes to handle business logic.

### Customer Service

Create `src/main/java/com/example/customerapi/service/CustomerService.java`:

```java
package com.example.customerapi.service;

import com.example.customerapi.exception.ResourceAlreadyExistsException;
import com.example.customerapi.exception.ResourceNotFoundException;
import com.example.customerapi.model.Customer;
import com.example.customerapi.repository.CustomerRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class CustomerService {

    private final CustomerRepository customerRepository;

    @Transactional(readOnly = true)
    public List<Customer> getAllCustomers() {
        log.info("Fetching all customers");
        return customerRepository.findAll();
    }

    @Transactional(readOnly = true)
    public Customer getCustomerById(Long id) {
        log.info("Fetching customer with id: {}", id);
        return customerRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Customer not found with id: " + id));
    }

    @Transactional(readOnly = true)
    public Customer getCustomerByEmail(String email) {
        log.info("Fetching customer with email: {}", email);
        return customerRepository.findByEmail(email)
                .orElseThrow(() -> new ResourceNotFoundException("Customer not found with email: " + email));
    }

    @Transactional
    public Customer createCustomer(Customer customer) {
        log.info("Creating new customer with email: {}", customer.getEmail());
        
        if (customerRepository.existsByEmail(customer.getEmail())) {
            throw new ResourceAlreadyExistsException("Customer already exists with email: " + customer.getEmail());
        }
        
        customer.setStatus(Customer.CustomerStatus.ACTIVE);
        Customer savedCustomer = customerRepository.save(customer);
        log.info("Customer created successfully with id: {}", savedCustomer.getId());
        
        return savedCustomer;
    }

    @Transactional
    public Customer updateCustomer(Long id, Customer customerDetails) {
        log.info("Updating customer with id: {}", id);
        
        Customer customer = getCustomerById(id);
        
        // Update fields
        customer.setCompanyName(customerDetails.getCompanyName());
        customer.setContactName(customerDetails.getContactName());
        customer.setPhone(customerDetails.getPhone());
        
        if (!customer.getEmail().equals(customerDetails.getEmail())) {
            if (customerRepository.existsByEmail(customerDetails.getEmail())) {
                throw new ResourceAlreadyExistsException("Email already in use: " + customerDetails.getEmail());
            }
            customer.setEmail(customerDetails.getEmail());
        }
        
        Customer updatedCustomer = customerRepository.save(customer);
        log.info("Customer updated successfully with id: {}", id);
        
        return updatedCustomer;
    }

    @Transactional
    public void deleteCustomer(Long id) {
        log.info("Deleting customer with id: {}", id);
        Customer customer = getCustomerById(id);
        customerRepository.delete(customer);
        log.info("Customer deleted successfully with id: {}", id);
    }

    @Transactional
    public Customer updateCustomerStatus(Long id, Customer.CustomerStatus status) {
        log.info("Updating customer status to {} for id: {}", status, id);
        Customer customer = getCustomerById(id);
        customer.setStatus(status);
        return customerRepository.save(customer);
    }

    @Transactional(readOnly = true)
    public List<Customer> getCustomersByStatus(Customer.CustomerStatus status) {
        log.info("Fetching customers with status: {}", status);
        return customerRepository.findByStatus(status);
    }
}
```

### Invoice Service

Create `src/main/java/com/example/customerapi/service/InvoiceService.java`:

```java
package com.example.customerapi.service;

import com.example.customerapi.exception.ResourceAlreadyExistsException;
import com.example.customerapi.exception.ResourceNotFoundException;
import com.example.customerapi.model.Invoice;
import com.example.customerapi.repository.InvoiceRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDate;
import java.util.List;

@Service
@RequiredArgsConstructor
@Slf4j
public class InvoiceService {

    private final InvoiceRepository invoiceRepository;
    private final CustomerService customerService;

    @Transactional(readOnly = true)
    public List<Invoice> getAllInvoices() {
        log.info("Fetching all invoices");
        return invoiceRepository.findAll();
    }

    @Transactional(readOnly = true)
    public Invoice getInvoiceById(Long id) {
        log.info("Fetching invoice with id: {}", id);
        return invoiceRepository.findById(id)
                .orElseThrow(() -> new ResourceNotFoundException("Invoice not found with id: " + id));
    }

    @Transactional(readOnly = true)
    public List<Invoice> getInvoicesByCustomerId(Long customerId) {
        log.info("Fetching invoices for customer id: {}", customerId);
        // Verify customer exists
        customerService.getCustomerById(customerId);
        return invoiceRepository.findByCustomerId(customerId);
    }

    @Transactional
    public Invoice createInvoice(Invoice invoice) {
        log.info("Creating new invoice for customer id: {}", invoice.getCustomerId());
        
        // Verify customer exists
        customerService.getCustomerById(invoice.getCustomerId());
        
        if (invoiceRepository.existsByInvoiceNumber(invoice.getInvoiceNumber())) {
            throw new ResourceAlreadyExistsException("Invoice already exists with number: " + invoice.getInvoiceNumber());
        }
        
        invoice.setStatus(Invoice.InvoiceStatus.PENDING);
        Invoice savedInvoice = invoiceRepository.save(invoice);
        log.info("Invoice created successfully with id: {}", savedInvoice.getId());
        
        return savedInvoice;
    }

    @Transactional
    public Invoice updateInvoiceStatus(Long id, Invoice.InvoiceStatus status) {
        log.info("Updating invoice status to {} for id: {}", status, id);
        Invoice invoice = getInvoiceById(id);
        invoice.setStatus(status);
        return invoiceRepository.save(invoice);
    }

    @Transactional
    public void deleteInvoice(Long id) {
        log.info("Deleting invoice with id: {}", id);
        Invoice invoice = getInvoiceById(id);
        invoiceRepository.delete(invoice);
        log.info("Invoice deleted successfully with id: {}", id);
    }

    @Transactional(readOnly = true)
    public List<Invoice> getOverdueInvoices() {
        log.info("Fetching overdue invoices");
        return invoiceRepository.findByStatus(Invoice.InvoiceStatus.PENDING)
                .stream()
                .filter(invoice -> invoice.getDueAt().isBefore(LocalDate.now()))
                .toList();
    }
}
```

---

## Building REST Controllers

Now let's expose our services through REST endpoints.

### Customer Controller

Create `src/main/java/com/example/customerapi/controller/CustomerController.java`:

```java
package com.example.customerapi.controller;

import com.example.customerapi.model.Customer;
import com.example.customerapi.service.CustomerService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/customers")
@RequiredArgsConstructor
public class CustomerController {

    private final CustomerService customerService;

    @GetMapping
    public ResponseEntity<List<Customer>> getAllCustomers(
            @RequestParam(required = false) Customer.CustomerStatus status) {
        if (status != null) {
            return ResponseEntity.ok(customerService.getCustomersByStatus(status));
        }
        return ResponseEntity.ok(customerService.getAllCustomers());
    }

    @GetMapping("/{id}")
    public ResponseEntity<Customer> getCustomerById(@PathVariable Long id) {
        return ResponseEntity.ok(customerService.getCustomerById(id));
    }

    @GetMapping("/email/{email}")
    public ResponseEntity<Customer> getCustomerByEmail(@PathVariable String email) {
        return ResponseEntity.ok(customerService.getCustomerByEmail(email));
    }

    @PostMapping
    public ResponseEntity<Customer> createCustomer(@Valid @RequestBody Customer customer) {
        Customer createdCustomer = customerService.createCustomer(customer);
        return ResponseEntity.status(HttpStatus.CREATED).body(createdCustomer);
    }

    @PutMapping("/{id}")
    public ResponseEntity<Customer> updateCustomer(
            @PathVariable Long id,
            @Valid @RequestBody Customer customer) {
        return ResponseEntity.ok(customerService.updateCustomer(id, customer));
    }

    @PatchMapping("/{id}/status")
    public ResponseEntity<Customer> updateCustomerStatus(
            @PathVariable Long id,
            @RequestParam Customer.CustomerStatus status) {
        return ResponseEntity.ok(customerService.updateCustomerStatus(id, status));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteCustomer(@PathVariable Long id) {
        customerService.deleteCustomer(id);
        return ResponseEntity.noContent().build();
    }
}
```

### Invoice Controller

Create `src/main/java/com/example/customerapi/controller/InvoiceController.java`:

```java
package com.example.customerapi.controller;

import com.example.customerapi.model.Invoice;
import com.example.customerapi.service.InvoiceService;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/v1/invoices")
@RequiredArgsConstructor
public class InvoiceController {

    private final InvoiceService invoiceService;

    @GetMapping
    public ResponseEntity<List<Invoice>> getAllInvoices() {
        return ResponseEntity.ok(invoiceService.getAllInvoices());
    }

    @GetMapping("/{id}")
    public ResponseEntity<Invoice> getInvoiceById(@PathVariable Long id) {
        return ResponseEntity.ok(invoiceService.getInvoiceById(id));
    }

    @GetMapping("/customer/{customerId}")
    public ResponseEntity<List<Invoice>> getInvoicesByCustomerId(@PathVariable Long customerId) {
        return ResponseEntity.ok(invoiceService.getInvoicesByCustomerId(customerId));
    }

    @GetMapping("/overdue")
    public ResponseEntity<List<Invoice>> getOverdueInvoices() {
        return ResponseEntity.ok(invoiceService.getOverdueInvoices());
    }

    @PostMapping
    public ResponseEntity<Invoice> createInvoice(@Valid @RequestBody Invoice invoice) {
        Invoice createdInvoice = invoiceService.createInvoice(invoice);
        return ResponseEntity.status(HttpStatus.CREATED).body(createdInvoice);
    }

    @PatchMapping("/{id}/status")
    public ResponseEntity<Invoice> updateInvoiceStatus(
            @PathVariable Long id,
            @RequestParam Invoice.InvoiceStatus status) {
        return ResponseEntity.ok(invoiceService.updateInvoiceStatus(id, status));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteInvoice(@PathVariable Long id) {
        invoiceService.deleteInvoice(id);
        return ResponseEntity.noContent().build();
    }
}
```

---

## Adding Validation and Error Handling

### Custom Exceptions

Create `src/main/java/com/example/customerapi/exception/ResourceNotFoundException.java`:

```java
package com.example.customerapi.exception;

public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String message) {
        super(message);
    }
}
```

Create `src/main/java/com/example/customerapi/exception/ResourceAlreadyExistsException.java`:

```java
package com.example.customerapi.exception;

public class ResourceAlreadyExistsException extends RuntimeException {
    public ResourceAlreadyExistsException(String message) {
        super(message);
    }
}
```

### Global Exception Handler

Create `src/main/java/com/example/customerapi/exception/GlobalExceptionHandler.java`:

```java
package com.example.customerapi.exception;

import lombok.extern.slf4j.Slf4j;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFoundException(ResourceNotFoundException ex) {
        log.error("Resource not found: {}", ex.getMessage());
        ErrorResponse error = new ErrorResponse(
                HttpStatus.NOT_FOUND.value(),
                ex.getMessage(),
                LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(ResourceAlreadyExistsException.class)
    public ResponseEntity<ErrorResponse> handleResourceAlreadyExistsException(ResourceAlreadyExistsException ex) {
        log.error("Resource already exists: {}", ex.getMessage());
        ErrorResponse error = new ErrorResponse(
                HttpStatus.CONFLICT.value(),
                ex.getMessage(),
                LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidationExceptions(MethodArgumentNotValidException ex) {
        log.error("Validation failed: {}", ex.getMessage());
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        
        ValidationErrorResponse response = new ValidationErrorResponse(
                HttpStatus.BAD_REQUEST.value(),
                "Validation failed",
                errors,
                LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(Exception ex) {
        log.error("Unexpected error occurred: {}", ex.getMessage(), ex);
        ErrorResponse error = new ErrorResponse(
                HttpStatus.INTERNAL_SERVER_ERROR.value(),
                "An unexpected error occurred",
                LocalDateTime.now()
        );
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }

    record ErrorResponse(int status, String message, LocalDateTime timestamp) {}
    
    record ValidationErrorResponse(int status, String message, Map<String, String> errors, LocalDateTime timestamp) {}
}
```

### Configure Application Properties

Update `src/main/resources/application.properties`:

```properties
# Application
spring.application.name=customer-api
server.port=8080

# H2 Database
spring.datasource.url=jdbc:h2:mem:customerdb
spring.datasource.driverClassName=org.h2.Driver
spring.datasource.username=sa
spring.datasource.password=

# JPA/Hibernate
spring.jpa.database-platform=org.hibernate.dialect.H2Dialect
spring.jpa.hibernate.ddl-auto=create-drop
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# H2 Console (for debugging)
spring.h2.console.enabled=true
spring.h2.console.path=/h2-console

# Logging
logging.level.com.example.customerapi=DEBUG
logging.level.org.springframework.web=INFO
logging.level.org.hibernate.SQL=DEBUG
```

---

## Testing Your API

### Start the Application

```bash
# Using Maven
./mvnw spring-boot:run

# Or using Maven Wrapper on Windows
mvnw.cmd spring-boot:run

# Or if you have Maven installed globally
mvn spring-boot:run
```

You should see output like:
```
Started CustomerApiApplication in 3.456 seconds
```

### Test Endpoints with curl

**1. Create a Customer**

```bash
curl -X POST http://localhost:8080/api/v1/customers \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john@acmecorp.com",
    "companyName": "Acme Corp",
    "contactName": "John Doe",
    "phone": "+1-555-0123"
  }'
```

**Response:**
```json
{
  "id": 1,
  "email": "john@acmecorp.com",
  "companyName": "Acme Corp",
  "contactName": "John Doe",
  "phone": "+1-555-0123",
  "status": "ACTIVE",
  "createdAt": "2026-01-11T20:30:00",
  "updatedAt": "2026-01-11T20:30:00"
}
```

**2. Get All Customers**

```bash
curl http://localhost:8080/api/v1/customers
```

**3. Get Customer by ID**

```bash
curl http://localhost:8080/api/v1/customers/1
```

**4. Update Customer**

```bash
curl -X PUT http://localhost:8080/api/v1/customers/1 \
  -H "Content-Type: application/json" \
  -d '{
    "email": "john.doe@acmecorp.com",
    "companyName": "Acme Corporation",
    "contactName": "John Doe",
    "phone": "+1-555-0199"
  }'
```

**5. Create an Invoice**

```bash
curl -X POST http://localhost:8080/api/v1/invoices \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": 1,
    "invoiceNumber": "INV-2026-001",
    "amount": 1250.00,
    "currency": "USD",
    "dueAt": "2026-02-11"
  }'
```

**6. Get Customer Invoices**

```bash
curl http://localhost:8080/api/v1/invoices/customer/1
```

**7. Update Invoice Status**

```bash
curl -X PATCH "http://localhost:8080/api/v1/invoices/1/status?status=PAID"
```

**8. Get Overdue Invoices**

```bash
curl http://localhost:8080/api/v1/invoices/overdue
```

**9. Delete Customer**

```bash
curl -X DELETE http://localhost:8080/api/v1/customers/1
```

### Postman Collection

For easier testing, import this Postman collection:

```json
{
  "info": {
    "name": "Customer Management API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "item": [
    {
      "name": "Create Customer",
      "request": {
        "method": "POST",
        "header": [{"key": "Content-Type", "value": "application/json"}],
        "body": {
          "mode": "raw",
          "raw": "{\n  \"email\": \"john@acmecorp.com\",\n  \"companyName\": \"Acme Corp\",\n  \"contactName\": \"John Doe\",\n  \"phone\": \"+1-555-0123\"\n}"
        },
        "url": {"raw": "http://localhost:8080/api/v1/customers"}
      }
    }
  ]
}
```

Save this as `customer-api.postman_collection.json` and import it into Postman.

### Access H2 Console

For database debugging, access the H2 console:

1. Open: http://localhost:8080/h2-console
2. JDBC URL: `jdbc:h2:mem:customerdb`
3. Username: `sa`
4. Password: (leave empty)
5. Click **Connect**

---

## What's Next

Congratulations! You've built a production-ready REST API with Spring Boot. Here's what we covered:

✅ Spring Boot project setup  
✅ JPA entities with validation  
✅ Repository and service layers  
✅ RESTful API endpoints  
✅ Comprehensive error handling  
✅ Testing with curl and Postman  

### Coming Up in This Series

**Post 2: Understanding OCI - Cloud Infrastructure Fundamentals**  
Learn about Oracle Cloud Infrastructure components, networking, compute instances, and IAM concepts you'll need for deployment.

**Post 3: Containerizing and Deploying Spring Boot on OCI**  
We'll containerize this application with Docker and deploy it to an OCI compute instance, making it accessible from the internet.

**Post 4: Secrets Management with HashiCorp Vault**  
Replace hardcoded credentials with Vault for secure secrets management.

---

## Complete Code Repository

The complete source code for this tutorial is available on GitHub:

📦 **[github.com/your-username/customer-api-series](https://github.com/your-username/customer-api-series)**

Each post has its own branch:
- `post-1-spring-boot-api` - This tutorial
- `post-2-oci-fundamentals` - Infrastructure setup
- `post-3-oci-deployment` - Cloud deployment
- And more...

---

## Troubleshooting

### Port 8080 Already in Use

```bash
# Find and kill the process
# On macOS/Linux:
lsof -ti:8080 | xargs kill -9

# On Windows:
netstat -ano | findstr :8080
taskkill /PID <PID> /F
```

### Lombok Not Working

Make sure you've enabled annotation processing in your IDE:

**IntelliJ IDEA:**
1. Settings → Build, Execution, Deployment → Compiler → Annotation Processors
2. Enable "Enable annotation processing"

**VS Code:**
Install the "Lombok Annotations Support" extension

### H2 Console Not Accessible

Verify `spring.h2.console.enabled=true` is in `application.properties`

---

## Key Takeaways

🔑 **Clean Architecture**: Separation of concerns with controller, service, and repository layers  
🔑 **Validation**: Bean validation ensures data integrity at the API boundary  
🔑 **Error Handling**: Centralized exception handling provides consistent error responses  
🔑 **RESTful Design**: Proper HTTP methods and status codes for intuitive API design  
🔑 **Database Abstraction**: Spring Data JPA eliminates boilerplate database code  

---

## Additional Resources

- [Spring Boot Documentation](https://spring.io/projects/spring-boot)
- [Spring Data JPA Guide](https://spring.io/guides/gs/accessing-data-jpa/)
- [REST API Best Practices](https://restfulapi.net/)
- [HTTP Status Codes Reference](https://httpstatuses.com/)

---

## Connect With Me

Found this tutorial helpful? Follow me for more:

- 🐦 Twitter: [@yourusername](https://twitter.com/yourusername)
- 💼 LinkedIn: [Your Name](https://linkedin.com/in/yourprofile)
- 🌐 Website: [yourdomain.com](https://yourdomain.com)

⭐ **Star the GitHub repo** if you found this useful!

---

*Next in series: [Post 2: Understanding OCI - Cloud Infrastructure Fundamentals](#) →*

---

## Tags

#SpringBoot #Java #RestAPI #Backend #CloudNative #OCI #Tutorial #Programming #SoftwareDevelopment #Microservices