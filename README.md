# pgAudit Installation Guide for macOS Apple Silicon

This repository provides a comprehensive guide and tools for installing the PostgreSQL Audit Extension (pgAudit) on macOS systems with Apple Silicon processors.

## About This Project

While installing pgAudit on Linux systems is well-documented, macOS users—particularly those with Apple Silicon (M1/M2/M3) processors—face unique challenges due to SDK compatibility issues, architecture differences, and library path problems.

This guide aims to solve these issues with:

1. Detailed documentation
2. Ready-to-use scripts
3. Best practices for configuration

## Key Features

- **Automatic SDK Detection**: Finds and uses the appropriate macOS SDK
- **Architecture Compatibility**: Handles the ARM64 vs x86_64 compilation challenges
- **Library Resolution**: Manages library search paths correctly
- **Safe Configuration**: Includes rollback capability if installation fails
- **Comprehensive Verification**: Tools to test and validate your installation

## View the Guide

Read the complete guide on GitHub Pages:
[pgAudit macOS Installation Guide](https://gatescrispy.github.io/pgaudit-macos-silicon-guide/)

## What's Included

- **build_pgaudit_macos.sh**: Script to compile and install pgAudit
- **apply_pgaudit_config.sh**: Script to configure PostgreSQL
- **pgaudit_config.conf**: Sample configuration file
- **test_pgaudit.sql**: SQL script for testing
- **check_audit_logs.sh**: Script for analyzing audit logs

## Technical Background

This work was developed after attempting to contribute these scripts to the official pgAudit repository. While the maintainers preferred not to include OS-specific installation scripts in the main project, they encouraged sharing this as a separate resource.

The article provides a deep dive into the technical challenges and solutions involved in getting pgAudit working correctly on macOS Apple Silicon, particularly addressing issues with:

- SDK version mismatches
- Architecture compatibility issues
- Library search path complications

## License

This documentation and all scripts are released under the [MIT License](LICENSE).

## Contributing

Contributions are welcome! If you have improvements or fixes, please submit a pull request.

---

*Created by [Gatescrispy](https://github.com/Gatescrispy)*