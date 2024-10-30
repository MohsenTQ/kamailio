# Kamailio and FreePBX Custom Headers Integration Guide

## 1. Kamailio Configuration

### Basic Configuration (kamailio.cfg)
```
#!KAMAILIO
#!define WITH_MYSQL
#!define WITH_NAT

loadmodule "textops.so"
loadmodule "siputils.so"
loadmodule "sl.so"
loadmodule "tm.so"
loadmodule "rr.so"
loadmodule "pv.so"
loadmodule "htable.so"

# FreePBX connection parameters
#!define FREEPBX_IP "192.168.1.100"
#!define FREEPBX_PORT "5060"

# Custom headers we want to handle
#!define CUSTOM_HEADER_1 "X-Custom-Route"
#!define CUSTOM_HEADER_2 "X-Account-ID"
#!define CUSTOM_HEADER_3 "X-Service-Type"

# Database connection
modparam("htable", "htable", "headers=>size=8;autoexpire=3600")

# Main request routing
request_route {
    # Basic request checks
    if (!mf_process_maxfwd_header("10")) {
        sl_send_reply("483", "Too Many Hops");
        exit;
    }

    if (is_method("INVITE")) {
        route(HANDLE_INVITE);
    }
    
    if (is_method("BYE")) {
        route(HANDLE_BYE);
    }
}

# INVITE handling
route[HANDLE_INVITE] {
    # Store original RURI
    $var(original_ruri) = $ru;
    
    # Check for custom routing header
    if (is_present_hf("$var(CUSTOM_HEADER_1)")) {
        $var(route_value) = $(hdr(CUSTOM_HEADER_1));
        xlog("L_INFO", "Custom route header found: $var(route_value)\n");
        
        # Store in hash table for response handling
        $sht(headers=>$ci) = $var(route_value);
    }
    
    # Add additional custom headers if needed
    append_hf("X-Original-URI: $var(original_ruri)\r\n");
    append_hf("X-Caller-ID: $fU\r\n");
    
    # You can also modify existing headers
    if (is_present_hf("X-Account-ID")) {
        $var(account) = $(hdr(X-Account-ID));
        remove_hf("X-Account-ID");
        append_hf("X-Account-ID: MODIFIED-$var(account)\r\n");
    }
    
    # Redirect to FreePBX
    $ru = "sip:" + $rU + "@" + FREEPBX_IP + ":" + FREEPBX_PORT;
    
    route(RELAY);
}

# Response handling
onreply_route[HANDLE_RESPONSE] {
    if (is_present_hf("X-FreePBX-Route")) {
        $var(pbx_route) = $(hdr(X-FreePBX-Route));
        xlog("L_INFO", "FreePBX route received: $var(pbx_route)\n");
    }
}

# BYE handling
route[HANDLE_BYE] {
    # Clean up stored headers
    $sht(headers=>$ci) = $null;
    route(RELAY);
}

# Relay route
route[RELAY] {
    if (!t_relay()) {
        sl_reply_error();
    }
}
```

## 2. FreePBX Configuration

### 1. Custom Extensions Configuration
```php
// File: /etc/asterisk/extensions_custom.conf

[from-kamailio-custom]
exten => _X.,1,NoOp(Incoming call from Kamailio with custom headers)
same => n,Set(CHANNEL(hangup_handler_push)=hangup-handler,s,1)
same => n,Set(CUSTOM_ROUTE=${SIP_HEADER(X-Custom-Route)})
same => n,Set(ACCOUNT_ID=${SIP_HEADER(X-Account-ID)})
same => n,Set(SERVICE_TYPE=${SIP_HEADER(X-Service-Type)})
same => n,GotoIf($["${CUSTOM_ROUTE}" = ""]?no-route,1:has-route,1)

exten => has-route,1,NoOp(Custom route: ${CUSTOM_ROUTE})
same => n,Set(CALLERID(name)=${CUSTOM_ROUTE}-${CALLERID(name)})
same => n,Goto(${CUSTOM_ROUTE},${EXTEN},1)

exten => no-route,1,NoOp(No custom route defined)
same => n,Goto(from-internal,${EXTEN},1)

[hangup-handler]
exten => s,1,NoOp(Call cleanup)
same => n,Set(CUSTOM_HEADER="X-FreePBX-Route: ${CUSTOM_ROUTE}")
same => n,SIPAddHeader(${CUSTOM_HEADER})
same => n,Return
```

### 2. SIP Configuration
```ini
; File: /etc/asterisk/sip_custom.conf

[kamailio-trunk]
type=friend
host=KAMAILIO_IP
context=from-kamailio-custom
insecure=port,invite
qualify=yes
nat=force_rport,comedia
directmedia=no
; Allow custom headers
allowtransfer=yes
; Customize based on your needs
disallow=all
allow=ulaw
allow=alaw
```

### 3. Dialplan Functions
```php
// File: /etc/asterisk/extensions_custom.conf

[custom-routes]
; Example route for handling specific account types
exten => account-route,1,NoOp(Account specific routing)
same => n,Set(CALLERID(name)=Account-${ACCOUNT_ID})
same => n,Set(__ACCOUNT_TYPE=${ACCOUNT_ID})
same => n,Goto(account-handling,${EXTEN},1)

; Example route for service-based routing
exten => service-route,1,NoOp(Service specific routing)
same => n,Set(CALLERID(name)=Service-${SERVICE_TYPE})
same => n,Set(__SERVICE_TYPE=${SERVICE_TYPE})
same => n,Goto(service-handling,${EXTEN},1)
```

## 3. Testing and Debugging

### 1. Test SIP INVITE with Custom Headers
```bash
# Using sipsak
sipsak -v -s sip:test@yourdomain.com \
  -H "X-Custom-Route: account-route" \
  -H "X-Account-ID: 12345" \
  -H "X-Service-Type: premium"

# Using SIPp
cat > test.xml <<EOF
<?xml version="1.0" encoding="ISO-8859-1"?>
<scenario name="Custom Headers Test">
  <send retrans="500">
    <![CDATA[
      INVITE sip:[service]@[remote_ip]:[remote_port] SIP/2.0
      Via: SIP/2.0/[transport] [local_ip]:[local_port];branch=[branch]
      From: sipp <sip:sipp@[local_ip]:[local_port]>;tag=[call_number]
      To: [service] <sip:[service]@[remote_ip]:[remote_port]>
      Call-ID: [call_id]
      CSeq: 1 INVITE
      Contact: sip:sipp@[local_ip]:[local_port]
      X-Custom-Route: account-route
      X-Account-ID: 12345
      X-Service-Type: premium
      Content-Type: application/sdp
      Content-Length: [len]

      v=0
      o=user1 53655765 2353687637 IN IP[local_ip_type] [local_ip]
      s=-
      c=IN IP[media_ip_type] [media_ip]
      t=0 0
      m=audio [media_port] RTP/AVP 0
      a=rtpmap:0 PCMU/8000
    ]]>
  </send>
</scenario>
EOF

sipp -sf test.xml KAMAILIO_IP:5060
```

### 2. Debug Commands

#### Kamailio Debugging
```bash
# Enable debug mode
kamctl fifo debug 4

# Check active calls with custom headers
kamctl htable show headers

# Monitor SIP traffic
ngrep -d any -W byline port 5060
```

#### FreePBX/Asterisk Debugging
```bash
# Connect to Asterisk CLI
asterisk -rvvv

# Enable SIP debug
sip set debug on

# Watch specific channel
core show channel ${CHANNEL}

# Monitor custom variables
dialplan show variables
```

## 4. Common Use Cases

### 1. Route Based on Account Type
```
# In Kamailio (kamailio.cfg)
if (is_method("INVITE")) {
    sql_query("cb", "SELECT account_type FROM accounts WHERE id='$fU'", "ra");
    if ($dbr(ra=>rows) > 0) {
        append_hf("X-Custom-Route: account-route\r\n");
        append_hf("X-Account-ID: $(dbr(ra=>[0,0]))\r\n");
    }
}

# In FreePBX (extensions_custom.conf)
[account-handling]
exten => _X.,1,NoOp(Account type: ${ACCOUNT_TYPE})
same => n,GotoIf($["${ACCOUNT_TYPE}" = "premium"]?premium,1:standard,1)

exten => premium,1,NoOp(Premium routing)
same => n,Goto(premium-services,${EXTEN},1)

exten => standard,1,NoOp(Standard routing)
same => n,Goto(standard-services,${EXTEN},1)
```

### 2. Service-Based Routing
```
# In Kamailio (kamailio.cfg)
if (is_method("INVITE") && $(ru{uri.param,service})) {
    append_hf("X-Custom-Route: service-route\r\n");
    append_hf("X-Service-Type: $(ru{uri.param,service})\r\n");
}

# In FreePBX (extensions_custom.conf)
[service-handling]
exten => _X.,1,NoOp(Service type: ${SERVICE_TYPE})
same => n,GotoIf($["${SERVICE_TYPE}" = "support"]?support,1:sales,1)

exten => support,1,NoOp(Support routing)
same => n,Goto(support-queue,${EXTEN},1)

exten => sales,1,NoOp(Sales routing)
same => n,Goto(sales-queue,${EXTEN},1)
```

## 5. Error Handling

```
# In Kamailio (kamailio.cfg)
failure_route[HANDLE_FAILURE] {
    if (t_check_status("408")) {
        xlog("L_ERROR", "Request timeout for call $ci\n");
        append_hf("X-Error-Cause: Request-Timeout\r\n");
    }
    if (t_check_status("403")) {
        xlog("L_ERROR", "Forbidden for call $ci\n");
        append_hf("X-Error-Cause: Forbidden\r\n");
    }
}

# In FreePBX (extensions_custom.conf)
[error-handling]
exten => handle-error,1,NoOp(Error handling: ${ERROR_CAUSE})
same => n,Set(CUSTOM_HEADER="X-Error-Response: ${ERROR_CAUSE}")
same => n,SIPAddHeader(${CUSTOM_HEADER})
same => n,Hangup()
```
