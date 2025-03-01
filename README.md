# Azure Functions - Storage and Database Cleanup

## Overview
This project contains two Azure Functions that handle automated cleanup tasks:
1. **Blob Storage Cleanup**: Removes archived blobs older than a specified retention period
2. **Database Cleanup**: Deletes old records from Logs and AuditTrail tables


## Functions Configuration

### 1. Blob Storage Cleanup Function

#### Environment Variables
```bash
STORAGE_ACCOUNT_NAME="storadatainazure"    # Storage account name
CONTAINER_NAME="conatinatertest"           # Container to clean up
RETENTION_DAYS="90"                        # Retention period in days
```

#### Required Permissions
The function's managed identity needs these RBAC roles:
- `Storage Blob Data Contributor`
- `Storage Blob Data Reader`

#### Function Logic
- Connects to specified storage account using managed identity
- Lists blobs in the target container
- Deletes archived blobs older than retention period

### 2. Database Cleanup Function

#### Environment Variables
```bash
SQL_SERVER="mysqlserverdemo100.database.windows.net"  # SQL Server address
SQL_DATABASE="mydbfunction"                           # Database name
RETENTION_DAYS="90"                                  # Retention period in days
```

#### Required Permissions
The function's managed identity needs these SQL roles:
- `db_datareader`
- `db_datawriter`

#### Function Logic
- Connects to SQL Database using managed identity
- Deletes old records from:
  - Logs table
  - AuditTrail table
- Uses parameterized queries for safety

## Deployment

### 1. Enable Managed Identity
```bash
# Enable managed identity for your function app
az functionapp identity assign \
    --name YourFunctionAppName \
    --resource-group YourResourceGroup
```

### 2. Storage Account  and Database Permissions
Use system assign to grant necessary permissions to your function app.

![image](https://github.com/user-attachments/assets/268fd4a1-a59b-4829-ae07-bd83e33a8800)


### 3-1. Database Permissions and Schema
Connect to your SQL Database and run:
Initial Setup and Permissions

```
-- Replace <function-app-name> with your function app's name
CREATE USER [powerfullappfunction] FROM EXTERNAL PROVIDER;

-- Grant necessary permissions
ALTER ROLE db_datareader ADD MEMBER [powerfullappfunction];
ALTER ROLE db_datawriter ADD MEMBER [powerfullappfunction];

-- Grant explicit DELETE permissions for specific tables
-- First, create example tables if they don't exist
CREATE TABLE Logs (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    LogMessage NVARCHAR(MAX),
    CreatedDate DATETIME DEFAULT GETDATE()
);

CREATE TABLE AuditTrail (
    Id INT IDENTITY(1,1) PRIMARY KEY,
    AuditMessage NVARCHAR(MAX),
    Timestamp DATETIME DEFAULT GETDATE()
);

-- Grant DELETE permissions
GRANT DELETE ON dbo.Logs TO [powerfullappfunction];
GRANT DELETE ON dbo.AuditTrail TO [powerfullappfunction];
```

### 3-2.Verify Permissions

Use these queries to verify the permissions are set correctly:

```
-- Check user existence and roles
SELECT dp.name as principal_name,
       dp.type_desc as principal_type,
       o.name as role_name
FROM sys.database_principals dp
LEFT JOIN sys.database_role_members rm ON dp.principal_id = rm.member_principal_id
LEFT JOIN sys.database_principals o ON rm.role_principal_id = o.principal_id
WHERE dp.name = 'powerfullappfunction';

-- Check table permissions
SELECT pr.principal_id, pr.name, pr.type_desc,
       o.name as object_name,
       p.permission_name,
       p.state_desc
FROM sys.database_permissions p
JOIN sys.objects o ON p.major_id = o.object_id
JOIN sys.database_principals pr ON p.grantee_principal_id = pr.principal_id
WHERE pr.name = 'powerfullappfunction'
AND o.name IN ('Logs', 'AuditTrail');
```

![image](https://github.com/user-attachments/assets/fe0f3369-e6ba-4d14-a7a9-63ce4ba19166)

### 4. Configure Environment Variables

![image](https://github.com/user-attachments/assets/c13d6627-12c2-4976-a49b-76b3d586974d)

## Monitoring

### Function Logs
Both functions provide detailed logging:
- Connection attempts
- Operation counts
- Error details
- Completion summaries

### Metrics to Monitor
1. Storage Cleanup:
   - Number of blobs processed
   - Number of blobs deleted
   - Error count
   - Execution duration

2. Database Cleanup:
   - Number of records deleted
   - Query execution time
   - Error count
   - Execution duration

## Testing

# Blob Storage Cleanup function

![image](https://github.com/user-attachments/assets/8c5c12f7-3e3d-4e0c-bb91-5285de9ebff9)

# Database Cleanup function

![image](https://github.com/user-attachments/assets/7742c4d9-188a-4505-b944-1c252ad5a847)


## Troubleshooting

### Common Issues

1. **Storage Access Denied**
   - Verify managed identity is enabled
   - Check RBAC role assignments
   - Confirm storage account network access settings

2. **Database Connection Failed**
   - Verify SQL Server firewall rules
   - Check managed identity SQL permissions
   - Confirm ODBC driver installation

3. **Function Timeout**
   - Adjust function timeout setting
   - Review execution duration logs
   - Consider batch processing for large datasets

### Verification Steps

1. **Storage Function**:
   - Check function logs for successful execution
   - Verify blob count changes
   - Confirm deleted blobs match criteria

2. **Database Function**:
   - Review SQL Server audit logs
   - Verify record count changes
   - Check transaction logs

## Security Considerations

1. **Authentication**
   - Using managed identity for all authentication
   - No stored credentials
   - Regular token rotation

2. **Data Access**
   - Minimum required permissions
   - Scoped to specific resources
   - Parameterized SQL queries

3. **Network Security**
   - Azure service integration
   - Configurable firewall rules
   - Encrypted connections

## Contributing
1. Fork the repository
2. Create a feature branch
3. Commit your changes
4. Push to the branch
5. Create a Pull Request

