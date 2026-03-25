# AWS RDS MySQL Setup Guide for Free-Tier Style Development

This guide collects the full implementation notes for:

1. Choosing the right options when creating a MySQL database in AWS RDS
2. Understanding what the other creation options mean
3. Configuring VPC and security group inbound rules
4. Connecting from DataGrip
5. Connecting from a Spring Boot app in IntelliJ
6. Understanding common connection and startup issues

---

## 1. Goal of this setup

This setup is designed for a common development workflow:

- use **AWS RDS MySQL**
- keep cost as low as possible
- connect from your **local laptop**
- use **DataGrip** for manual DB access
- use **Spring Boot** in IntelliJ for application development

For this use case, the best general direction is:

- **MySQL**
- **Full configuration**
- **Free tier** when available
- **Publicly accessible = Yes**
- **Security group inbound rule for MySQL on port 3306**
- restrict access to **your IP only**

---

## 2. Create database in AWS RDS

Go to:

**AWS Console -> RDS -> Databases -> Create database**

---

## 3. Engine options

### Recommended choice
- **Engine type**: `MySQL`

### Why
Choose MySQL if:
- your application already uses MySQL
- your Spring Boot project uses MySQL JDBC
- you want easy compatibility with DataGrip, IntelliJ, Hibernate, and common MySQL tools

### Other engine options explained

#### Aurora (MySQL Compatible)
- AWS-managed database engine
- compatible with MySQL
- designed for better cloud-native scaling and availability
- usually more advanced than needed for simple personal dev work

Use it when:
- you specifically want AWS Aurora features
- you are building a more production-oriented system

#### Aurora (PostgreSQL Compatible)
- Aurora engine compatible with PostgreSQL
- useful if your project is based on PostgreSQL but you want Aurora

#### PostgreSQL
- strong open-source relational database
- very popular for backend systems and advanced SQL features

Use it when:
- your application is designed around PostgreSQL
- you need PostgreSQL-specific features

#### MariaDB
- related to MySQL
- similar in many cases, but not identical

Use it when:
- your application explicitly uses MariaDB

#### Oracle / Microsoft SQL Server / IBM Db2
- enterprise-oriented database engines
- usually more expensive and more specialized
- generally not the best fit for a low-cost learning setup

---

## 4. Choose a database creation method

You will usually see:

- **Easy create**
- **Full configuration**

### Recommended choice
- **Full configuration**

### Why
Use **Full configuration** if you want to learn and control:
- template
- connectivity
- public access
- VPC
- subnet group
- security group
- backup settings
- instance size
- storage options

### Easy create
Use **Easy create** if:
- you want the fastest path
- you do not care about understanding each option yet

### Recommendation for learning
For practice and future real-world understanding, **Full configuration** is better.

---

## 5. Template selection

You will usually see:
- **Production**
- **Dev/Test**
- **Free tier**

### Recommended choice
- **Free tier**

### Why
Choose **Free tier** when:
- you want the lowest-cost setup
- you are learning RDS
- your traffic is very small
- you only need a lightweight environment for Spring Boot and DataGrip

### Other options explained

#### Dev/Test
Use this when:
- Free tier is unavailable in your account or region
- you want a cheaper non-production setup
- you do not need production-level defaults

#### Production
Use this when:
- you want stronger defaults for availability and resilience
- you are building a real production system
- you accept the higher cost

### Recommendation for your use case
For local development + DataGrip + Spring Boot testing, **Free tier** is usually the best option.

---

## 6. Recommended settings during creation

Below is a practical recommended setup.

### 6.1 Engine version
- Use the default stable MySQL version shown by AWS unless your project requires a specific version

### 6.2 DB instance identifier
Example:

```text
database-1
```

This is the RDS instance name, not the database/schema name inside MySQL.

### 6.3 Master username
Example:

```text
admin
```

### 6.4 Credentials management
Recommended:
- **Self managed**

This lets you control the password directly.

You can use AWS Secrets Manager later in more advanced or production setups.

### 6.5 DB instance class
Recommended:
- choose the smallest free-tier eligible or lowest-cost instance shown in the AWS console

Typical goal:
- the cheapest option that is still enough for testing and development

### 6.6 Storage
Recommended:
- General Purpose SSD
- small allocated storage for development

### 6.7 Multi-AZ deployment
Recommended for low-cost development:
- **No**

Why:
- Multi-AZ improves availability
- but increases cost
- usually unnecessary for local development practice

### 6.8 Initial database name
You may create an initial database name such as:

```text
demo_db
```

This is the database/schema your Spring Boot application can connect to.

### 6.9 Public access
Recommended for your use case:
- **Yes**

Why:
- your laptop, DataGrip, and IntelliJ need direct access from outside the AWS VPC

If you choose **No**, your local machine usually cannot connect directly unless you use:
- an EC2 bastion host
- SSH tunnel
- VPN
- an application running inside the same VPC

### 6.10 Port
For MySQL:

```text
3306
```

---

## 7. Connectivity and VPC choices

### Recommended choices
- **Network type**: `IPv4`
- **VPC**: default VPC is fine for learning
- **DB subnet group**: default is fine for a simple dev setup
- **Security group**: attach one you can edit
- **Public access**: `Yes`
- **Database port**: `3306`

### Why default VPC is okay for learning
If your goal is:
- understand RDS
- connect from DataGrip
- connect from Spring Boot

then the default VPC is usually enough.

In production environments, teams often use:
- custom VPCs
- private subnets
- tighter route control
- app-to-DB private connectivity only

---

## 8. Security group inbound configuration

This is the most common reason DataGrip times out.

After the DB is created, go to:

**RDS -> Databases -> your DB -> Connectivity & security -> click the attached security group**

Then open **Inbound rules**.

### What often exists by default
A default security group rule may look like:
- **Type**: All traffic
- **Source**: the same security group (`sg-xxxx`)

This only allows resources **inside the same security group** to connect.

It does **not** allow your laptop to connect.

### Rule you need for local access
Add a new inbound rule:

```text
Type: MySQL/Aurora
Protocol: TCP
Port range: 3306
Source: My IP
```

### Best practice
Use:
- **My IP**
- or your exact public IP like `x.x.x.x/32`

Do **not** leave this open to:

```text
0.0.0.0/0
```

unless you are doing a temporary quick test.

### Example
A correct local-dev inbound rule:

```text
Type: MySQL/Aurora
Protocol: TCP
Port range: 3306
Source: 123.45.67.89/32
```

### Why this matters
Even if:
- the endpoint is correct
- the password is correct
- the database is publicly accessible

you still cannot connect if the security group blocks inbound port `3306`.

---

## 9. What you need from the RDS details page

After creation, copy these values:

- **Endpoint**
  - example:

  ```text
  database-1.xxxxx.ap-southeast-1.rds.amazonaws.com
  ```

- **Port**
  - usually:

  ```text
  3306
  ```

- **Master username**
  - example:

  ```text
  admin
  ```

- **Database name**
  - example:

  ```text
  demo_db
  ```

---

## 10. Test connectivity from terminal first

Before using DataGrip, test connection from your terminal.

### Test TCP reachability

```bash
nc -vz database-1.xxxxx.ap-southeast-1.rds.amazonaws.com 3306
```

If it succeeds, your laptop can reach the RDS port.

### Test with MySQL CLI

```bash
mysql -h database-1.xxxxx.ap-southeast-1.rds.amazonaws.com -P 3306 -u admin -p
```

Then enter your password.

### If it times out
Usually one of these is wrong:
- DB is not in **Available** state
- public access is off
- inbound security group rule is missing
- wrong endpoint
- local network/firewall blocks port `3306`

---

## 11. Connect from DataGrip

Create a new **MySQL** data source in DataGrip.

### Fill in:
- **Host**: RDS endpoint only
- **Port**: `3306`
- **User**: your RDS username
- **Password**: your RDS password
- **Database**: optional, such as `demo_db`

### Correct host

```text
database-1.xxxxx.ap-southeast-1.rds.amazonaws.com
```

### Wrong host

```text
https://database-1.xxxxx.ap-southeast-1.rds.amazonaws.com
```

Do not include `http://` or `https://`.

### SSL
For the first connection test:
- try **SSL disabled**

After the connection works, you can configure SSL later if needed.

### If DataGrip shows `Operation timed out`
That usually means:
- it cannot reach the server over the network
- the problem is usually not the MySQL version

Most common reasons:
- security group inbound rule missing
- DB is not publicly accessible
- wrong endpoint or port
- local network blocks `3306`

### About the version warning
If DataGrip cannot reach the server, it also cannot detect the MySQL version.  
So timeout errors should be fixed before worrying about version selection.

---

## 12. Spring Boot setup in IntelliJ

Use `application.properties` like this:

```properties
spring.application.name=demo

spring.datasource.url=jdbc:mysql://YOUR_RDS_ENDPOINT:3306/demo_db
spring.datasource.username=admin
spring.datasource.password=YOUR_PASSWORD

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

### Example

```properties
spring.application.name=demo

spring.datasource.url=jdbc:mysql://database-1.xxxxx.ap-southeast-1.rds.amazonaws.com:3306/demo_db
spring.datasource.username=admin
spring.datasource.password=your_real_password

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

### About Hibernate dialect
You usually do **not** need this:

```properties
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQLDialect
```

Hibernate can usually infer the dialect automatically.

---

## 13. Maven dependencies for Spring Boot

Make sure you have at least:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<dependency>
    <groupId>com.mysql</groupId>
    <artifactId>mysql-connector-j</artifactId>
    <scope>runtime</scope>
</dependency>
```

### Why `spring-boot-starter-web` matters
If your app should run as a normal web application and stay alive, this dependency is usually needed.

Without it, your application may start, initialize the database, then exit.

---

## 14. Main class example

```java
@SpringBootApplication
public class DemoApplication {
    public static void main(String[] args) {
        SpringApplication.run(DemoApplication.class, args);
    }
}
```

---

## 15. How to understand Spring Boot logs

If your logs show things like:

- JDBC URL printed
- `Database driver: MySQL Connector/J`
- `Database version: ...`
- `Initialized JPA EntityManagerFactory`
- `Started DemoApplication`

then the DB connection is already successful.

### If the app then shuts down
If you later see:
- `Closing JPA EntityManagerFactory`
- `HikariPool - Shutdown initiated`
- `HikariPool - Shutdown completed`

that usually means:
- the app connected successfully
- then the application exited normally

This is usually an **application lifecycle issue**, not an RDS connectivity issue.

### Common reasons Spring Boot exits
- missing `spring-boot-starter-web`
- app is a command-line app, not a web app
- code closes the context
- the process was manually stopped

---

## 16. Common problems and how to interpret them

### Problem: DataGrip says `Operation timed out`
Meaning:
- your machine cannot reach the RDS server

Most likely causes:
- public access is disabled
- security group inbound rule is missing
- wrong endpoint
- wrong port
- local network blocks 3306

### Problem: DataGrip asks for MySQL version
Meaning:
- it cannot auto-detect while the server is unreachable

Fix network/connectivity first.

### Problem: Spring Boot logs show DB metadata and JPA initialized
Meaning:
- the DB connection already works

### Problem: Spring Boot starts then exits
Meaning:
- usually a Spring app lifecycle/config problem, not an RDS connection problem

---

## 17. Final recommended setup for your use case

### Create DB
- Engine: `MySQL`
- Creation method: `Full configuration`
- Template: `Free tier`
- Public access: `Yes`
- Port: `3306`
- Smallest eligible instance
- Initial DB name: `demo_db`

### Security group inbound
```text
Type: MySQL/Aurora
Protocol: TCP
Port range: 3306
Source: My IP
```

### DataGrip
- MySQL data source
- Host = RDS endpoint
- Port = 3306
- User/password = RDS credentials

### Spring Boot
```properties
spring.datasource.url=jdbc:mysql://YOUR_RDS_ENDPOINT:3306/demo_db
spring.datasource.username=admin
spring.datasource.password=YOUR_PASSWORD
```

---

## 18. Quick copy section

### DataGrip settings
```text
Host: database-1.xxxxx.ap-southeast-1.rds.amazonaws.com
Port: 3306
User: admin
Password: your_password
Database: demo_db
```

### Spring Boot settings
```properties
spring.application.name=demo

spring.datasource.url=jdbc:mysql://database-1.xxxxx.ap-southeast-1.rds.amazonaws.com:3306/demo_db
spring.datasource.username=admin
spring.datasource.password=your_password

spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

### Security group inbound rule
```text
Type: MySQL/Aurora
Protocol: TCP
Port range: 3306
Source: My IP
```

### Terminal test commands
```bash
nc -vz database-1.xxxxx.ap-southeast-1.rds.amazonaws.com 3306
mysql -h database-1.xxxxx.ap-southeast-1.rds.amazonaws.com -P 3306 -u admin -p
```

---

## 19. Final conclusion

For a low-cost local development workflow using AWS RDS MySQL, DataGrip, and Spring Boot:

- choose **MySQL**
- use **Full configuration**
- choose **Free tier** when available
- set **Public access = Yes**
- add security group inbound rule for **MySQL/Aurora TCP 3306**
- restrict access to **My IP**
- use the RDS **endpoint + port 3306** in both DataGrip and Spring Boot

If DataGrip times out, the problem is usually networking or security group rules.

If Spring Boot shows MySQL metadata and JPA initialization, then database connectivity is already working.
