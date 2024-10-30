# Kamailio Billing System Implementation Guide

## 1. Database Schema Setup

### Create Billing Tables
```sql
-- Account/Customer table
CREATE TABLE billing_accounts (
    id INT PRIMARY KEY AUTO_INCREMENT,
    account_number VARCHAR(32) UNIQUE,
    username VARCHAR(64),
    balance DECIMAL(10,4) DEFAULT 0.0000,
    credit_limit DECIMAL(10,4) DEFAULT 0.0000,
    status ENUM('active', 'suspended', 'terminated') DEFAULT 'active',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Rate table
CREATE TABLE billing_rates (
    id INT PRIMARY KEY AUTO_INCREMENT,
    prefix VARCHAR(32),
    description VARCHAR(128),
    rate DECIMAL(10,4),
    connection_fee DECIMAL(10,4) DEFAULT 0.0000,
    rate_group INT,
    time_start TIME,
    time_end TIME,
    weekday_start TINYINT,
    weekday_end TINYINT,
    priority INT DEFAULT 1,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Call Detail Records (CDR)
CREATE TABLE billing_cdrs (
    id INT PRIMARY KEY AUTO_INCREMENT,
    call_id VARCHAR(128) UNIQUE,
    caller VARCHAR(64),
    callee VARCHAR(64),
    start_time TIMESTAMP,
    answer_time TIMESTAMP NULL,
    end_time TIMESTAMP NULL,
    duration INT DEFAULT 0,
    rate DECIMAL(10,4),
    cost DECIMAL(10,4),
    status ENUM('started', 'answered', 'ended', 'failed') DEFAULT 'started',
    account_id INT,
    rate_id INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES billing_accounts(id),
    FOREIGN KEY (rate_id) REFERENCES billing_rates(id)
);

-- Account Transactions
CREATE TABLE billing_transactions (
    id INT PRIMARY KEY AUTO_INCREMENT,
    account_id INT,
    amount DECIMAL(10,4),
    type ENUM('debit', 'credit'),
    description VARCHAR(255),
    reference_id VARCHAR(128),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (account_id) REFERENCES billing_accounts(id)
);

-- Create necessary indexes
CREATE INDEX idx_rates_prefix ON billing_rates(prefix);
CREATE INDEX idx_cdrs_call_id ON billing_cdrs(call_id);
CREATE INDEX idx_cdrs_account ON billing_cdrs(account_id);
CREATE INDEX idx_transactions_account ON billing_transactions(account_id);
```

## 2. Kamailio Configuration

### Main Configuration (kamailio.cfg)
```
#!KAMAILIO
#!define WITH_MYSQL
#!define WITH_AUTH
#!define WITH_BILLING

loadmodule "db_mysql.so"
loadmodule "htable.so"
loadmodule "dialog.so"
loadmodule "rtimer.so"
loadmodule "ctl.so"
loadmodule "sqlops.so"

# Module Parameters
modparam("htable", "htable", "active_calls=>size=8;autoexpire=7200")
modparam("dialog", "db_mode", 1)
modparam("rtimer", "timer", "name=billing;interval=60;mode=1;")

# Database Connection
modparam("sqlops", "sqlcon", "cb=>mysql://kamailio:password@localhost/kamailio")

# Custom Functions
## Check Account Balance
route[CHECK_BALANCE] {
    sql_query("cb", "SELECT balance, credit_limit, status FROM billing_accounts \
              WHERE username = '$fU' AND status = 'active'", "ra");
    
    if ($dbr(ra=>rows) > 0) {
        $var(balance) = $dbr(ra=>[0,0]);
        $var(credit_limit) = $dbr(ra=>[0,1]);
        
        if ($var(balance) + $var(credit_limit) <= 0) {
            xlog("L_WARN", "Insufficient balance for user $fU\n");
            sl_send_reply("402", "Payment Required");
            exit;
        }
        return(1);
    }
    return(-1);
}

## Get Call Rate
route[GET_RATE] {
    $var(prefix) = $(rU{s.substr,0,4});
    sql_query("cb", "SELECT id, rate, connection_fee FROM billing_rates \
              WHERE prefix = '$var(prefix)' AND \
              TIME(NOW()) BETWEEN time_start AND time_end AND \
              WEEKDAY(NOW()) BETWEEN weekday_start AND weekday_end \
              ORDER BY priority DESC LIMIT 1", "ra");
    
    if ($dbr(ra=>rows) > 0) {
        $var(rate_id) = $dbr(ra=>[0,0]);
        $var(rate) = $dbr(ra=>[0,1]);
        $var(connection_fee) = $dbr(ra=>[0,2]);
        return(1);
    }
    return(-1);
}

## Start Call Billing
route[START_BILLING] {
    if (!route(GET_RATE)) {
        xlog("L_ERR", "No rate found for prefix $var(prefix)\n");
        sl_send_reply("402", "Payment Required");
        exit;
    }
    
    sql_query("cb", "INSERT INTO billing_cdrs (call_id, caller, callee, start_time, \
              rate, account_id, rate_id) VALUES ('$ci', '$fU', '$rU', NOW(), \
              $var(rate), (SELECT id FROM billing_accounts WHERE username='$fU'), \
              $var(rate_id))");
              
    $sht(active_calls=>$ci) = $var(rate);
}

## End Call Billing
route[END_BILLING] {
    if ($sht(active_calls=>$ci)) {
        sql_query("cb", "UPDATE billing_cdrs SET end_time = NOW(), \
                  duration = UNIX_TIMESTAMP(NOW()) - UNIX_TIMESTAMP(start_time), \
                  status = 'ended' WHERE call_id = '$ci'");
                  
        sql_query("cb", "SELECT duration, rate FROM billing_cdrs WHERE call_id = '$ci'", "ra");
        if ($dbr(ra=>rows) > 0) {
            $var(duration) = $dbr(ra=>[0,0]);
            $var(rate) = $dbr(ra=>[0,1]);
            $var(cost) = ($var(duration) / 60) * $var(rate) + $var(connection_fee);
            
            sql_query("cb", "UPDATE billing_cdrs SET cost = $var(cost) WHERE call_id = '$ci'");
            sql_query("cb", "UPDATE billing_accounts SET balance = balance - $var(cost) \
                      WHERE username = '$fU'");
            
            # Record transaction
            sql_query("cb", "INSERT INTO billing_transactions (account_id, amount, type, \
                      description, reference_id) VALUES ((SELECT id FROM billing_accounts \
                      WHERE username='$fU'), $var(cost), 'debit', 'Call charge', '$ci')");
        }
        
        $sht(active_calls=>$ci) = $null;
    }
}

# Main Request Route
request_route {
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483", "Too Many Hops");
        exit;
    }

    if (is_method("INVITE") && !has_totag()) {
        if (!route(CHECK_BALANCE)) {
            sl_send_reply("403", "Forbidden");
            exit;
        }
        
        route(START_BILLING);
        route(RELAY);
    }
    
    if (is_method("BYE")) {
        route(END_BILLING);
        route(RELAY);
    }
}
```

## 3. Management Scripts

### Add Account
```bash
#!/bin/bash
# save as add_account.sh

MYSQL_USER="kamailio"
MYSQL_PASS="password"
MYSQL_DB="kamailio"

add_account() {
    mysql -u "$MYSQL_USER" -p"$MYSQL_PASS" "$MYSQL_DB" <<EOF
    INSERT INTO billing_accounts (account_number, username, balance, credit_limit) 
    VALUES ('$1', '$2', $3, $4);
EOF
}

# Usage
# ./add_account.sh "ACC001" "user@domain.com" 100.00 50.00
```

### Add Rate
```bash
#!/bin/bash
# save as add_rate.sh

add_rate() {
    mysql -u "$MYSQL_USER" -p"$MYSQL_PASS" "$MYSQL_DB" <<EOF
    INSERT INTO billing_rates (prefix, description, rate, connection_fee) 
    VALUES ('$1', '$2', $3, $4);
EOF
}

# Usage
# ./add_rate.sh "1" "USA" 0.01 0.05
```

## 4. Reporting Tools

### Generate CDR Report
```sql
-- Daily Usage Report
SELECT 
    DATE(start_time) as call_date,
    COUNT(*) as total_calls,
    SUM(duration)/60 as total_minutes,
    SUM(cost) as total_cost
FROM billing_cdrs
WHERE status = 'ended'
GROUP BY DATE(start_time)
ORDER BY call_date DESC;

-- User Usage Report
SELECT 
    ba.username,
    COUNT(*) as total_calls,
    SUM(bc.duration)/60 as total_minutes,
    SUM(bc.cost) as total_cost,
    ba.balance as current_balance
FROM billing_cdrs bc
JOIN billing_accounts ba ON bc.account_id = ba.id
WHERE bc.status = 'ended'
GROUP BY ba.username;
```

## 5. Maintenance Tasks

### Regular Maintenance Queries
```sql
-- Clean up old CDRs
DELETE FROM billing_cdrs 
WHERE end_time < DATE_SUB(NOW(), INTERVAL 90 DAY);

-- Suspend accounts with negative balance
UPDATE billing_accounts 
SET status = 'suspended' 
WHERE balance + credit_limit <= 0;

-- Generate monthly invoices
INSERT INTO billing_invoices (account_id, amount, period_start, period_end)
SELECT 
    account_id,
    SUM(cost) as total_amount,
    DATE_FORMAT(NOW() - INTERVAL 1 MONTH, '%Y-%m-01'),
    LAST_DAY(NOW() - INTERVAL 1 MONTH)
FROM billing_cdrs
WHERE status = 'ended'
AND start_time BETWEEN DATE_FORMAT(NOW() - INTERVAL 1 MONTH, '%Y-%m-01')
AND LAST_DAY(NOW() - INTERVAL 1 MONTH)
GROUP BY account_id;
```

## 6. Monitoring and Alerts

### Set up monitoring queries
```sql
-- Monitor for suspicious activity
SELECT caller, COUNT(*) as call_count
FROM billing_cdrs
WHERE start_time > DATE_SUB(NOW(), INTERVAL 1 HOUR)
GROUP BY caller
HAVING call_count > 100;

-- Monitor for failed payments
SELECT ba.username, ba.balance, ba.credit_limit
FROM billing_accounts ba
WHERE ba.balance + ba.credit_limit <= 10
AND ba.status = 'active';
```

## 7. API Integration Example

```python
from flask import Flask, jsonify
import mysql.connector

app = Flask(__name__)

@app.route('/api/balance/<username>')
def get_balance(username):
    conn = mysql.connector.connect(
        host="localhost",
        user="kamailio",
        password="password",
        database="kamailio"
    )
    cursor = conn.cursor()
    
    cursor.execute("""
        SELECT balance, credit_limit 
        FROM billing_accounts 
        WHERE username = %s
    """, (username,))
    
    result = cursor.fetchone()
    
    if result:
        return jsonify({
            'username': username,
            'balance': float(result[0]),
            'credit_limit': float(result[1])
        })
    
    return jsonify({'error': 'User not found'}), 404

if __name__ == '__main__':
    app.run(port=5000)
```

