---
layout: default
title: pgAudit Installation Guide for macOS Apple Silicon
description: Solving pgAudit compilation and installation challenges on macOS with M-series processors
---

# Installing pgAudit on macOS Apple Silicon

*By: [Gatescrispy](https://github.com/Gatescrispy) | Last Updated: April 10, 2025*

## Introduction

PostgreSQL's audit logging extension (pgAudit) provides detailed session and object-level audit logging essential for security compliance and database monitoring. While installing pgAudit on Linux systems is well-documented, macOS users—particularly those with Apple Silicon processors—face unique challenges due to SDK compatibility issues and architecture differences.

Having encountered these issues firsthand during implementation of audit logging requirements for a financial services project, I've developed a comprehensive solution that addresses the specific challenges of the macOS/Apple Silicon environment.

## Key Challenges Identified

1. **SDK Version Mismatch**: PostgreSQL binaries from the EnterpriseDB installer are compiled against specific macOS SDK versions, creating conflicts with the SDK available on the user's system.

   ```
   ld: warning: object file was built for newer macOS version (15.0) than being linked (11.0)
   ```

2. **Architecture Incompatibilities**: When compiling for arm64 (Apple Silicon), various errors arise due to architecture mismatch with PostgreSQL's universal binary compilation.

   ```
   ld: warning: ignoring file 'pgaudit.o': found architecture 'arm64', required architecture 'x86_64'
   ```

3. **Library Path Issues**: Search paths for required libraries often don't align with macOS system configuration.

   ```
   ld: warning: search path '/opt/local/Current_v15/lib' not found
   ```

## Solution Overview

This guide offers a complete solution with:

1. A build script that automatically detects the available SDK and sets appropriate compilation flags
2. Configuration scripts with robust error handling and rollback capabilities
3. Test tools to verify the installation is working correctly

## Prerequisites

- PostgreSQL 16 installed via the EnterpriseDB installer
- Xcode Command Line Tools
- Git

## Step 1: Build Script Implementation

Our custom build script handles the complex compilation environment specific to macOS:

```bash
#!/bin/bash
# Script to compile pgAudit for PostgreSQL 16 on macOS (Apple Silicon)

set -e

# Verify if the directory already exists
if [ ! -d "/tmp/pgaudit" ]; then
  # Clone the pgAudit repository
  echo "Cloning pgAudit repository..."
  mkdir -p /tmp/pgaudit
  cd /tmp/pgaudit
  git clone https://github.com/pgaudit/pgaudit.git .
  git checkout REL_16_STABLE
else
  cd /tmp/pgaudit
  git checkout REL_16_STABLE
fi

# Get PostgreSQL configuration information
PG_CONFIG=/Library/PostgreSQL/16/bin/pg_config
PG_INCLUDEDIR=$($PG_CONFIG --includedir)
PG_PKGLIBDIR=$($PG_CONFIG --pkglibdir)
PG_SHAREDIR=$($PG_CONFIG --sharedir)
PG_CFLAGS=$($PG_CONFIG --cflags)
PG_CPPFLAGS=$($PG_CONFIG --cppflags)
PG_LDFLAGS=$($PG_CONFIG --ldflags)

# Configure to use the available SDK
# Find the latest available SDK
SDK_PATH=$(ls -d /Library/Developer/CommandLineTools/SDKs/MacOSX*.sdk | sort -V | tail -n 1)

echo "Compiling pgAudit with the following options:"
echo "PG_INCLUDEDIR: $PG_INCLUDEDIR"
echo "PG_PKGLIBDIR: $PG_PKGLIBDIR"
echo "SDK_PATH: $SDK_PATH"

# Compile pgAudit
echo "Compiling pgAudit..."
make clean || true
make USE_PGXS=1 PG_CONFIG=$PG_CONFIG CFLAGS="$PG_CFLAGS -isysroot $SDK_PATH" CPPFLAGS="$PG_CPPFLAGS -isysroot $SDK_PATH" LDFLAGS="$PG_LDFLAGS -isysroot $SDK_PATH"

# Install pgAudit
echo "Installing pgAudit..."
sudo make install USE_PGXS=1 PG_CONFIG=$PG_CONFIG

# Verify if the module was installed correctly
if [ -f "$PG_PKGLIBDIR/pgaudit.so" ] || [ -f "$PG_PKGLIBDIR/pgaudit.dylib" ]; then
  echo "pgAudit module successfully installed in $PG_PKGLIBDIR"
else
  echo "Error: pgAudit module not installed!"
  exit 1
fi

# Create the control file if it doesn't already exist
CONTROL_FILE="$PG_SHAREDIR/extension/pgaudit.control"
if [ ! -f "$CONTROL_FILE" ]; then
  echo "Creating pgaudit.control file..."
  sudo bash -c "cat > $CONTROL_FILE << EOF
# pgAudit extension
comment = 'PostgreSQL Audit Extension'
default_version = '16.0'
module_pathname = '\$libdir/pgaudit'
relocatable = false
trusted = true
EOF"
fi

# Verify if the control file was installed correctly
if [ -f "$CONTROL_FILE" ]; then
  echo "pgaudit.control file successfully installed in $PG_SHAREDIR/extension"
else
  echo "Error: pgaudit.control file not installed!"
  exit 1
fi

# Check SQL files
SQL_DIR="$PG_SHAREDIR/extension"
if [ -f "$SQL_DIR/pgaudit--16.0.sql" ] || [ -f "$SQL_DIR/pgaudit--16.1.sql" ]; then
  echo "pgAudit SQL files found in $SQL_DIR"
else
  echo "Warning: No pgAudit SQL files found in $SQL_DIR"
  
  # Create basic SQL files if needed
  if [ ! -f "$SQL_DIR/pgaudit--16.0.sql" ]; then
    echo "Creating a basic SQL file for pgAudit 16.0..."
    sudo bash -c "cat > $SQL_DIR/pgaudit--16.0.sql << EOF
-- complains if script is sourced in psql, since it's not inside a transaction
\\echo Use \"CREATE EXTENSION pgaudit\" to load this file. \\quit

-- Empty SQL file for pgAudit 16.0
EOF"
  fi
fi

echo "pgAudit installation complete!"
```

### The Core Solution: SDK Detection and Flag Configuration

The key insight in this script is using dynamic SDK detection combined with appropriate compiler and linker flags:

```bash
# Find the latest available SDK
SDK_PATH=$(ls -d /Library/Developer/CommandLineTools/SDKs/MacOSX*.sdk | sort -V | tail -n 1)

# Apply SDK path to all compilation flags
make USE_PGXS=1 PG_CONFIG=$PG_CONFIG CFLAGS="$PG_CFLAGS -isysroot $SDK_PATH" CPPFLAGS="$PG_CPPFLAGS -isysroot $SDK_PATH" LDFLAGS="$PG_LDFLAGS -isysroot $SDK_PATH"
```

This approach ensures that:
1. We use the proper SDK available on the system
2. We maintain compatibility with PostgreSQL's compilation environment
3. We handle architecture differences correctly

## Step 2: Configuration and PostgreSQL Integration

After compilation, we need to configure PostgreSQL to load pgAudit properly. Our configuration script handles this with proper error checking and rollback capability:

```bash
#!/bin/bash
# Script to apply pgAudit configuration and restart PostgreSQL

set -e

# Check for sudo privileges
if [ "$EUID" -ne 0 ]; then
  echo "This script must be run with sudo"
  exit 1
fi

# Paths
PG_DATA="/Library/PostgreSQL/16/data"
PG_CONFIG_FILE="$PG_DATA/postgresql.conf"
PGAUDIT_CONFIG="pgaudit_config.conf"
BACKUP_SUFFIX=".$(date +%Y%m%d%H%M%S).bak"

# Verify pgAudit is installed
if [ ! -f "/Library/PostgreSQL/16/lib/postgresql/pgaudit.dylib" ] && [ ! -f "/Library/PostgreSQL/16/lib/postgresql/pgaudit.so" ]; then
  echo "Error: pgAudit is not installed. Please run the build script first."
  exit 1
fi

# Backup the configuration file
echo "Backing up PostgreSQL configuration..."
cp "$PG_CONFIG_FILE" "$PG_CONFIG_FILE$BACKUP_SUFFIX"
echo "Backup created: $PG_CONFIG_FILE$BACKUP_SUFFIX"

# Add pgAudit configurations
echo "Adding pgAudit configurations..."
cat "$PGAUDIT_CONFIG" >> "$PG_CONFIG_FILE"
echo "pgAudit configuration added to $PG_CONFIG_FILE"

# Create log directory if needed
PG_LOG_DIR="$PG_DATA/pg_log"
if [ ! -d "$PG_LOG_DIR" ]; then
  echo "Creating log directory..."
  mkdir -p "$PG_LOG_DIR"
  chown postgres:postgres "$PG_LOG_DIR"
  chmod 700 "$PG_LOG_DIR"
fi

# Restart PostgreSQL
echo "Restarting PostgreSQL..."
/Library/PostgreSQL/16/bin/pg_ctl restart -D "$PG_DATA" -m fast

# Check if PostgreSQL started successfully
sleep 2
if /Library/PostgreSQL/16/bin/pg_isready -q; then
  echo "PostgreSQL successfully restarted with pgAudit configuration!"
  echo "You can check audit logs in $PG_LOG_DIR"
else
  echo "Error: PostgreSQL failed to start correctly."
  echo "Restoring previous configuration..."
  cp "$PG_CONFIG_FILE$BACKUP_SUFFIX" "$PG_CONFIG_FILE"
  /Library/PostgreSQL/16/bin/pg_ctl restart -D "$PG_DATA" -m fast
  echo "Configuration restored and PostgreSQL restarted."
  exit 1
fi
```

The pgAudit configuration itself focuses on a balance between comprehensive auditing and performance:

```
# Configuration for pgAudit

# Load pgAudit
shared_preload_libraries = 'pgaudit'

# Audit configuration
pgaudit.log = 'ddl, write'       # Audit DDL and write operations (INSERT, UPDATE, DELETE)
pgaudit.log_catalog = on         # Audit catalog objects
pgaudit.log_parameter = on       # Include query parameters
pgaudit.log_statement_once = on  # Log the statement text only once
pgaudit.log_level = 'log'        # Log level
pgaudit.log_relation = on        # Log all relations referenced in queries

# Logging configuration
logging_collector = on                # Enable log collection
log_directory = 'pg_log'              # Log directory
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log' # Log filename format
log_line_prefix = '%m [%p] %q%u@%d ' # Log line prefix format
log_statement = 'none'                # Disable standard query logging (pgAudit will handle it)
```

## Step 3: Testing and Verification

To verify the installation, we provide a comprehensive test SQL script:

```sql
-- Script to test pgAudit
-- Execute with: psql -U postgres -f test_pgaudit.sql

-- Check if pgaudit extension is available
SELECT * FROM pg_available_extensions WHERE name = 'pgaudit';

-- Create a test database
DROP DATABASE IF EXISTS audit_test;
CREATE DATABASE audit_test;

\connect audit_test

-- Enable pgaudit extension
CREATE EXTENSION IF NOT EXISTS pgaudit;

-- Create a test user
DROP ROLE IF EXISTS audit_test_user;
CREATE ROLE audit_test_user WITH LOGIN PASSWORD 'test123';

-- Create a test schema and table
CREATE SCHEMA audit_schema;
CREATE TABLE audit_schema.test_table (
    id SERIAL PRIMARY KEY,
    name TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert test data
INSERT INTO audit_schema.test_table (name) VALUES ('Test Record 1');
INSERT INTO audit_schema.test_table (name) VALUES ('Test Record 2');

-- Update data
UPDATE audit_schema.test_table SET name = 'Updated Record' WHERE id = 1;

-- Delete data
DELETE FROM audit_schema.test_table WHERE id = 2;

-- Grant privileges
GRANT USAGE ON SCHEMA audit_schema TO audit_test_user;
GRANT SELECT, INSERT, UPDATE, DELETE ON audit_schema.test_table TO audit_test_user;

-- Message to verify logs
\echo 'Audit test complete. Check logs in the pg_log directory.'
```

And a log analysis script to verify the audit trail:

```bash
#!/bin/bash
# Script to verify pgAudit audit logs

# PostgreSQL log path
PG_LOG_DIR="/Library/PostgreSQL/16/data/pg_log"

# Check if log directory exists
if [ ! -d "$PG_LOG_DIR" ]; then
  echo "Error: Log directory $PG_LOG_DIR does not exist."
  exit 1
fi

# Find the most recent log file
LATEST_LOG=$(ls -t "$PG_LOG_DIR"/postgresql-*.log 2>/dev/null | head -1)

if [ -z "$LATEST_LOG" ]; then
  echo "Error: No log files found in $PG_LOG_DIR"
  exit 1
fi

echo "Examining log file: $LATEST_LOG"
echo "----------------------------------------"

# Display audit entries (pgaudit)
echo "pgAudit entries:"
echo "----------------------------------------"
grep -i "AUDIT:" "$LATEST_LOG" | tail -n 20

# Statistics
AUDIT_COUNT=$(grep -i "AUDIT:" "$LATEST_LOG" | wc -l)
echo ""
echo "Audit statistics:"
echo "----------------------------------------"
echo "Total audit entries: $AUDIT_COUNT"
echo ""

# Show different types of audited operations
echo "Types of audited operations:"
echo "----------------------------------------"
grep -i "AUDIT:" "$LATEST_LOG" | grep -o "STATEMENT: [A-Z]\+" | sort | uniq -c | sort -nr

echo ""
echo "Audit entries by user:"
echo "----------------------------------------"
grep -i "AUDIT:" "$LATEST_LOG" | grep -o "USER: [a-z_]\+" | sort | uniq -c | sort -nr
```

## Performance Considerations and Best Practices

When implementing pgAudit in production environments, be aware of these performance implications:

1. **Selective Auditing**: Avoid auditing high-volume transactions by configuring `pgaudit.log` properly. For OLTP systems, consider auditing only DDL and critical data changes.

2. **Log Management**: The audit logs can grow substantially. Implement a log rotation strategy and consider offloading logs to a dedicated storage solution.

3. **System Resources**: Monitor CPU and I/O impact after enabling pgAudit, particularly on systems with high transaction volume.

4. **Object-Level Auditing**: For fine-grained control, consider using object-level auditing via the `pgaudit.role` setting rather than session-level auditing for everything.

## Advanced Configuration: Object Audit Role

To implement object-level auditing, create a dedicated audit role:

```sql
-- Create audit role
CREATE ROLE pgaudit_role;

-- Grant access to specific tables you want to audit
GRANT SELECT, INSERT, UPDATE, DELETE ON sensitive_table TO pgaudit_role;

-- Enable object auditing
ALTER SYSTEM SET pgaudit.role = 'pgaudit_role';
```

This approach allows for more precise control over what gets audited, reducing overhead and focusing logs on what matters.

## Conclusion and Applications

The pgAudit extension is essential for meeting compliance requirements in regulated environments, including:

- **Financial services**: Meeting SOX, PCI-DSS requirements
- **Healthcare**: HIPAA compliance for PHI access logs
- **Government**: FedRAMP, FISMA, and other regulatory frameworks

By solving the macOS Apple Silicon compatibility issues, this guide enables developers to test and implement proper audit logging locally before deploying to production environments.

## Technical Debt and Future Improvements

While these scripts provide a working solution, there are potential areas for improvement:

1. **Platform Detection**: Enhance the build script to detect processor architecture automatically and adjust flags accordingly
2. **Package Integration**: Explore creating a Homebrew formula for easier installation
3. **Version Compatibility**: Extend testing to cover PostgreSQL 13-17

## References and Resources

1. [Official pgAudit Documentation](https://github.com/pgaudit/pgaudit)
2. [PostgreSQL Documentation on Logging](https://www.postgresql.org/docs/16/runtime-config-logging.html)
3. [Apple Developer Documentation on SDK Compatibility](https://developer.apple.com/documentation/)

---

*Did you find this guide helpful? Have suggestions for improvements? [Open an issue](https://github.com/Gatescrispy/pgaudit-macos-silicon-guide/issues) or submit a PR!*