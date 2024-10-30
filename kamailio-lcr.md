# Kamailio Least Cost Routing (LCR) Configuration Guide

## 1. Install Required Modules

First, install the LCR module:
```bash
# Debian/Ubuntu
apt-get install kamailio-lcr-modules

# CentOS/RHEL
yum install kamailio-lcr
```

## 2. Database Setup

### Create LCR Tables
```sql
-- Connect to your Kamailio database and run:
CREATE TABLE lcr_gw (
    id INT(10) UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    lcr_id SMALLINT(5) UNSIGNED NOT NULL,
    gw_name VARCHAR(128),
    ip_addr VARCHAR(50),
    hostname VARCHAR(64),
    port SMALLINT UNSIGNED,
    params VARCHAR(64),
    uri_scheme TINYINT UNSIGNED,
    transport TINYINT UNSIGNED,
    strip TINYINT UNSIGNED,
    prefix VARCHAR(16),
    tag VARCHAR(64),
    flags INT UNSIGNED DEFAULT 0,
    defunct INT UNSIGNED DEFAULT 0
);

CREATE TABLE lcr_rule (
    id INT(10) UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    lcr_id SMALLINT(5) UNSIGNED NOT NULL,
    prefix VARCHAR(16),
    from_uri VARCHAR(64),
    request_uri VARCHAR(64),
    mt_tvalue VARCHAR(128),
    stopper INT(10) UNSIGNED DEFAULT 0,
    enabled INT(10) UNSIGNED DEFAULT 1
);

CREATE TABLE lcr_rule_target (
    id INT(10) UNSIGNED PRIMARY KEY AUTO_INCREMENT,
    lcr_id SMALLINT(5) UNSIGNED NOT NULL,
    rule_id INT(10) UNSIGNED NOT NULL,
    gw_id INT(10) UNSIGNED NOT NULL,
    priority INT(10) UNSIGNED NOT NULL,
    weight INT(10) UNSIGNED DEFAULT 1
);
```

## 3. Basic Configuration

Add to your `kamailio.cfg`:
```
#!define WITH_LCR

loadmodule "lcr.so"
loadmodule "tm.so"
loadmodule "sl.so"

# LCR Module parameters
modparam("lcr", "db_url", "mysql://kamailio:kamailiorw@localhost/kamailio")
modparam("lcr", "lcr_id", 1)
modparam("lcr", "gw_uri_avp", "$avp(lcr_gw)")
modparam("lcr", "ruri_user_avp", "$avp(lcr_ruri_user)")
modparam("lcr", "from_uri_avp", "$avp(lcr_from_uri)")

# Routing configuration
request_route {
    # ... (previous routing logic) ...

    if (is_method("INVITE")) {
        if (!load_gws(1)) {
            xlog("L_ERR", "LCR processing failed\n");
            sl_send_reply("500", "Server Error");
            exit;
        }

        if (!next_gw()) {
            xlog("L_ERR", "No gateway found\n");
            sl_send_reply("503", "Service Unavailable");
            exit;
        }

        route(ROUTE_TO_GW);
    }
}

route[ROUTE_TO_GW] {
    if (!t_relay()) {
        sl_reply_error();
    }
}
```

## 4. Adding Gateways and Rules

### Add Gateways
```sql
-- Example to add a gateway
INSERT INTO lcr_gw (lcr_id, gw_name, ip_addr, port, uri_scheme, transport) 
VALUES (1, 'gateway1', '10.0.0.1', 5060, 1, 1);

INSERT INTO lcr_gw (lcr_id, gw_name, ip_addr, port, uri_scheme, transport) 
VALUES (1, 'gateway2', '10.0.0.2', 5060, 1, 1);
```

### Add Rules
```sql
-- Example to add a rule
INSERT INTO lcr_rule (lcr_id, prefix) 
VALUES (1, '1'); -- Rule for numbers starting with 1

-- Add rule targets (linking rules to gateways with priorities)
INSERT INTO lcr_rule_target (lcr_id, rule_id, gw_id, priority) 
VALUES (1, 1, 1, 1); -- First try gateway1
INSERT INTO lcr_rule_target (lcr_id, rule_id, gw_id, priority) 
VALUES (1, 1, 2, 2); -- Then try gateway2
```

## 5. Advanced Configuration

### Cost-Based Routing
```
# Add cost information to gateways
ALTER TABLE lcr_gw ADD COLUMN cost DECIMAL(10,5);

# Modify routing logic in kamailio.cfg
if (is_method("INVITE")) {
    if (!load_gws(1, "$avp(cost)")) {
        xlog("L_ERR", "LCR processing failed\n");
        sl_send_reply("500", "Server Error");
        exit;
    }
    
    # Sort by cost
    if (!next_gw("1")) {
        xlog("L_ERR", "No gateway found\n");
        sl_send_reply("503", "Service Unavailable");
        exit;
    }
}
```

### Time-Based Routing
```sql
-- Add time-based rules
INSERT INTO lcr_rule (lcr_id, prefix, mt_tvalue) 
VALUES (1, '1', '* * 1-5 * *'); -- Weekdays only
```

## 6. Maintenance and Monitoring

### Check LCR Status
```bash
# Show active gateways
kamctl lcr show_gws

# Show rules
kamctl lcr show_rules

# Check routing for specific number
kamctl lcr dump_gws <number>
```

### Common Management Tasks
```sql
-- Disable a gateway temporarily
UPDATE lcr_gw SET defunct = 1 WHERE gw_name = 'gateway1';

-- Update gateway priorities
UPDATE lcr_rule_target SET priority = 3 WHERE gw_id = 2;

-- Add failover routing
INSERT INTO lcr_rule_target (lcr_id, rule_id, gw_id, priority) 
VALUES (1, 1, 3, 3); -- Third priority gateway
```

## 7. Troubleshooting

Enable LCR debugging in kamailio.cfg:
```
modparam("lcr", "debug", 1)
```

Common issues and solutions:
1. No matching rules: Check prefix patterns
2. Gateway unreachable: Verify network connectivity
3. Routing loops: Check gateway configurations
4. Performance issues: Monitor gateway response times
