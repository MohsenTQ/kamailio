# Kamailio Setup and Basic Configuration Guide

## 1. Installation

### Debian/Ubuntu
```bash
apt-get update
apt-get install kamailio kamailio-mysql-modules kamailio-websocket-modules
```

### CentOS/RHEL
```bash
yum install epel-release
yum install kamailio kamailio-mysql kamailio-websocket
```

## 2. Basic Configuration

The main configuration files are located in `/etc/kamailio/`:
- `kamailio.cfg` - Main configuration file
- `kamctlrc` - Kamailio control tool configuration

### Essential Configuration Steps

1. Edit `/etc/kamailio/kamctlrc`:
```ini
# Uncomment and set these parameters
SIP_DOMAIN=your-domain.com
DBENGINE=MYSQL
DBHOST=localhost
DBNAME=kamailio
DBRWUSER=kamailio
DBRWPW=kamailiorw
```

2. Create database and tables:
```bash
kamdbctl create
```

3. Basic `kamailio.cfg` template:
```
#!KAMAILIO
#!define WITH_MYSQL
#!define WITH_AUTH
#!define WITH_USRLOCDB

# Main parameters
listen=udp:*:5060
listen=tcp:*:5060

# Modules section
loadmodule "tm.so"
loadmodule "sl.so"
loadmodule "rr.so"
loadmodule "pv.so"
loadmodule "maxfwd.so"
loadmodule "usrloc.so"
loadmodule "registrar.so"
loadmodule "mysql.so"
loadmodule "auth.so"
loadmodule "auth_db.so"

# Module configurations
modparam("usrloc", "db_url", "mysql://kamailio:kamailiorw@localhost/kamailio")
modparam("auth_db", "calculate_ha1", 1)
modparam("auth_db", "password_column", "password")
modparam("auth_db", "db_url", "mysql://kamailio:kamailiorw@localhost/kamailio")

# Routing Logic
request_route {
    # Max forwards check
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483", "Too Many Hops");
        exit;
    }

    # Record route for all requests
    if (is_method("INVITE|SUBSCRIBE"))
        record_route();

    # Handle registrations
    if (is_method("REGISTER")) {
        if (!auth_check("$fd", "subscriber", "1")) {
            auth_challenge("$fd", "0");
            exit;
        }
        
        if (!save("location"))
            sl_reply_error();
            
        exit;
    }

    # Handle presence
    if (is_method("SUBSCRIBE|NOTIFY"))
        route(PRESENCE);

    # Authentication
    if (!auth_check("$fd", "subscriber", "1")) {
        auth_challenge("$fd", "0");
        exit;
    }

    # Route to destination
    if (!lookup("location")) {
        sl_send_reply("404", "User Not Found");
        exit;
    }
    
    route(RELAY);
}

route[RELAY] {
    if (!t_relay()) {
        sl_reply_error();
    }
}
```

## 3. Common Use Cases

### Basic SIP Proxy
- Routes SIP traffic between endpoints
- Handles registration and authentication
- Maintains user location database

### Load Balancer
```
# Add to kamailio.cfg
loadmodule "dispatcher.so"
modparam("dispatcher", "list_file", "/etc/kamailio/dispatcher.list")

# Example dispatcher.list content
1 sip:10.0.0.1:5060 0
1 sip:10.0.0.2:5060 0
```

### WebSocket Support
```
listen=ws:*:8080
loadmodule "websocket.so"
```

## 4. Management and Monitoring

### Basic Commands
```bash
# Start Kamailio
systemctl start kamailio

# Check status
kamctl online
kamctl stats

# User management
kamctl add username password
kamctl rm username

# Show registered users
kamctl ul show
```

### Debugging
```bash
# Enable debug mode
kamctl fifo debug 4

# View logs
tail -f /var/log/kamailio.log
```

## 5. Security Considerations

1. Always change default passwords
2. Use TLS for secure communications
3. Implement proper access controls
4. Regular security updates
5. Monitor for suspicious activity

## 6. Performance Tuning

Key parameters in kamailio.cfg:
```
# TCP Connection Settings
tcp_connection_lifetime=3605
tcp_max_connections=2048

# Worker Processes
children=8

# Memory Settings
pkg_mem=8
```
