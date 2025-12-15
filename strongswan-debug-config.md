# StrongSwan Debug Configuration

This document outlines the steps to enable detailed logging for the StrongSwan `charon` daemon via syslog. This configuration is intended for debugging connection and authentication issues.

## Configuration

### 1\. Edit Configuration File

Open the StrongSwan configuration file:

```bash
sudo vim /etc/strongswan.conf
```

### 2\. Modify Logging Levels

Update the `charon` block to include the `syslog` section as shown below.

```conf
charon {
    load_modular = yes
    plugins {
        include strongswan.d/charon/*.conf
    }
    
    # Enable debug logging
    syslog {
        daemon {
            # Level 2 = Control flow (Debug)
            # Level 3 = Data analysis (Verbose)
            default = 2
        }
    }
}

include strongswan.d/*.conf
```

> [!NOTE]
> It is recommended to revert the default log level to `0` or `1` after troubleshooting is complete to conserve disk space.

## Execution

Restart the StrongSwan service to apply changes and view the logs in real-time.

```bash
# Restart service
sudo systemctl restart strongswan

# Follow log output
sudo journalctl -u strongswan -f
```