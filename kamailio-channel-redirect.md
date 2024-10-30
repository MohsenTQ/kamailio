# Kamailio Channel Name Redirection to Freebox

## 1. Database Setup

First, create a table to store channel mapping:
```sql
CREATE TABLE channel_mapping (
    id INT(10) UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    original_name VARCHAR(128) NOT NULL,
    display_name VARCHAR(128) NOT NULL,
    freebox_ip VARCHAR(64) NOT NULL,
    freebox_port INT UNSIGNED DEFAULT 5060,
    enabled TINYINT(1) DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Create indexes for better performance
CREATE INDEX idx_original_name ON channel_mapping(original_name);
CREATE INDEX idx_enabled ON channel_mapping(enabled);
```

## 2. Kamailio Configuration

Add this to your `kamailio.cfg`:
```
#!KAMAILIO
#!define WITH_MYSQL
#!define WITH_NAT

loadmodule "db_mysql.so"
loadmodule "uac.so"
loadmodule "textops.so"
loadmodule "siputils.so"
loadmodule "htable.so"

# Module Parameters
modparam("htable", "htable", "channelCache=>size=8;autoexpire=3600")

# Database Connection
modparam("db_mysql", "db_url", "mysql://kamailio:password@localhost/kamailio")

# Custom Parameters
modparam("uac", "restore_mode", "auto")

# Helper Functions
route[LOAD_CHANNEL_INFO] {
    # Query channel mapping from database
    sql_query("cb", "SELECT display_name, freebox_ip, freebox_port FROM channel_mapping \
              WHERE original_name = $rU AND enabled = 1", "ra");
    
    if ($dbr(ra=>rows) > 0) {
        $var(display_name) = $dbr(ra=>[0,0]);
        $var(freebox_ip) = $dbr(ra=>[0,1]);
        $var(freebox_port) = $dbr(ra=>[0,2]);
        return(1);
    }
    return(-1);
}

# Main Request Route
request_route {
    # Initial request processing
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483", "Too Many Hops");
        exit;
    }

    if (is_method("INVITE")) {
        if (route(LOAD_CHANNEL_INFO)) {
            # Store original RURI
            $var(original_ruri) = $ru;
            
            # Modify Request-URI for Freebox
            $ru = "sip:" + $rU + "@" + $var(freebox_ip) + ":" + $var(freebox_port);
            
            # Preserve original channel name in display name
            uac_replace_from("$var(display_name)");
            
            # Add custom header for tracking
            append_hf("X-Original-Channel: $var(original_ruri)\r\n");
            
            # Cache the mapping
            $sht(channelCache=>$ci) = $var(display_name);
            
            xlog("L_INFO", "Redirecting channel $rU to Freebox at $ru\n");
            
            # Forward to Freebox
            route(RELAY);
            exit;
        }
    }

    # Handle responses
    if (is_method("BYE") && $sht(channelCache=>$ci)) {
        xlog("L_INFO", "Cleaning up channel mapping for call $ci\n");
        $sht(channelCache=>$ci) = $null;
    }
}

# Relay Route
route[RELAY] {
    if (!t_relay()) {
        sl_reply_error();
    }
}
```

## 3. Channel Management Scripts

### Add Channel Mapping
```bash
#!/bin/bash
# save as add_channel.sh

MYSQL_USER="kamailio"
MYSQL_PASS="password"
MYSQL_DB="kamailio"

add_channel() {
    mysql -u "$MYSQL_USER" -p"$MYSQL_PASS" "$MYSQL_DB" <<EOF
    INSERT INTO channel_mapping (original_name, display_name, freebox_ip, freebox_port) 
    VALUES ('$1', '$2', '$3', $4);
EOF
}

# Usage
# ./add_channel.sh "BBC1" "BBC One" "192.168.1.1" 5060
```

### Update Channel Mapping
```bash
#!/bin/bash
# save as update_channel.sh

MYSQL_USER="kamailio"
MYSQL_PASS="password"
MYSQL_DB="kamailio"

update_channel() {
    mysql -u "$MYSQL_USER" -p"$MYSQL_PASS" "$MYSQL_DB" <<EOF
    UPDATE channel_mapping 
    SET display_name='$2', freebox_ip='$3', freebox_port=$4 
    WHERE original_name='$1';
EOF
}

# Usage
# ./update_channel.sh "BBC1" "BBC One HD" "192.168.1.1" 5060
```

## 4. Example Usage

### Adding Channel Mappings
```sql
-- Add channel mappings
INSERT INTO channel_mapping (original_name, display_name, freebox_ip, freebox_port) 
VALUES 
('BBC1', 'BBC One HD', '192.168.1.1', 5060),
('ITV1', 'ITV 1 HD', '192.168.1.1', 5060),
('CH4', 'Channel 4 HD', '192.168.1.1', 5060);
```

### Testing the Setup
```bash
# Send test SIP INVITE
sipsak -v -s sip:BBC1@yourdomain.com

# Check logs
tail -f /var/log/kamailio.log

# View active mappings
kamctl htable show channelCache
```

## 5. Monitoring and Maintenance

### Check Channel Status
```sql
-- View all active channels
SELECT original_name, display_name, freebox_ip 
FROM channel_mapping 
WHERE enabled = 1;

-- Check channel usage
SELECT original_name, COUNT(*) as usage_count 
FROM channel_mapping_log 
GROUP BY original_name;
```

### Maintenance Tasks
```sql
-- Disable unused channels
UPDATE channel_mapping 
SET enabled = 0 
WHERE last_used < DATE_SUB(NOW(), INTERVAL 30 DAY);

-- Update Freebox IP for multiple channels
UPDATE channel_mapping 
SET freebox_ip = '192.168.1.2' 
WHERE freebox_ip = '192.168.1.1';
```

## 6. Troubleshooting

Common issues and solutions:

1. Channel not redirecting
   - Check if channel exists in database
   - Verify Freebox IP and port
   - Check Kamailio logs for errors

2. Display name not showing
   - Verify UAC module is loaded
   - Check From header manipulation
   - Verify SIP client supports display names

3. Performance issues
   - Monitor cache hit ratio
   - Optimize database indexes
   - Adjust cache size if needed
