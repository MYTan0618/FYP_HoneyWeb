# FYP_HoneyWeb
# Honeypot & ELK Stack - Quick Reference Guide

## 📋 Table of Contents
- [Starting System](#starting-system)
- [Stopping System](#stopping-system)
- [Checking Status](#checking-status)
- [Access Points](#access-points)
- [Testing & Monitoring](#testing--monitoring)
- [Telegram Alerts](#telegram-alerts)
- [Common Tasks](#common-tasks)
- [Troubleshooting](#troubleshooting)

---

## 🚀 Starting System

### Start Everything
```bash
# Start SNARE/TANNER honeypot
cd ~/honeypot
docker-compose up -d

# Start ELK Stack + Telegram Alerter
cd ~/honeypot/elk
docker-compose up -d

# Verify all services are running
docker ps
```

### Start Individual Components
```bash
# Start only honeypot
cd ~/honeypot && docker-compose up -d snare tanner

# Start only ELK stack
cd ~/honeypot/elk && docker-compose up -d elasticsearch kibana logstash

# Start only Telegram alerter
cd ~/honeypot/elk && docker-compose up -d telegram-alerter
```

### Expected Startup Time
- **Elasticsearch**: 30-60 seconds
- **Kibana**: 1-2 minutes
- **Logstash**: 15-30 seconds
- **SNARE/TANNER**: 10-15 seconds
- **Telegram Alerter**: 5-10 seconds

---

## 🛑 Stopping System

### Stop Everything
```bash
# Stop ELK Stack
cd ~/honeypot/elk
docker-compose down

# Stop Honeypot
cd ~/honeypot
docker-compose down
```

### Stop Individual Components
```bash
# Stop specific service
cd ~/honeypot/elk
docker-compose stop logstash

# Or
cd ~/honeypot
docker-compose stop snare
```

---

## ✅ Checking Status

### Quick Status Check
```bash
# Check all containers
docker ps

# Check specific containers
docker ps | grep -E "snare|tanner|elasticsearch|logstash|kibana|telegram"
```

### Detailed Health Check
```bash
# Elasticsearch health
curl -u fyp:fyp http://localhost:9200/_cluster/health?pretty

# Check if Logstash pipeline is running
docker logs logstash 2>&1 | grep "Pipeline started"

# Check if honeypot is responding
curl -I http://localhost:8080

# Check attack count
curl -s -u fyp:fyp "http://localhost:9200/honeypot-attacks-*/_count" | jq

# Check Telegram alerter status
docker logs telegram_alerter --tail 10
```

### Service Status Script
```bash
cat > ~/honeypot/check_status.sh << 'STATUSEOF'
#!/bin/bash

echo "================================"
echo "   HONEYPOT SYSTEM STATUS"
echo "================================"
echo ""

echo "📦 Container Status:"
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}" | grep -E "NAME|snare|tanner|elasticsearch|logstash|kibana|telegram"

echo ""
echo "🔍 Service Health:"

# Elasticsearch
if curl -s -u fyp:fyp http://localhost:9200/_cluster/health | grep -q "green\|yellow"; then
    echo "✓ Elasticsearch: Healthy"
else
    echo "✗ Elasticsearch: Down"
fi

# Kibana
if curl -s http://localhost:5601/api/status | grep -q "available"; then
    echo "✓ Kibana: Healthy"
else
    echo "✗ Kibana: Down or Starting"
fi

# Logstash
if docker logs logstash 2>&1 | grep -q "Pipeline started"; then
    echo "✓ Logstash: Pipeline Running"
else
    echo "✗ Logstash: Pipeline Not Started"
fi

# SNARE
if curl -s -I http://localhost:8080 | grep -q "200\|301\|302"; then
    echo "✓ SNARE: Responding"
else
    echo "✗ SNARE: Not Responding"
fi

# Telegram Alerter
if docker logs telegram_alerter 2>&1 | tail -5 | grep -q "Connected to Elasticsearch"; then
    echo "✓ Telegram Alerter: Connected"
else
    echo "✗ Telegram Alerter: Not Connected"
fi

echo ""
echo "📊 Attack Statistics:"
ATTACKS=$(curl -s -u fyp:fyp "http://localhost:9200/honeypot-attacks-*/_count" 2>/dev/null | grep -o '"count":[0-9]*' | grep -o '[0-9]*')
echo "Total Attacks Detected: ${ATTACKS:-0}"

echo ""
echo "🔗 Access URLs:"
echo "  • Honeypot: http://localhost:8080"
echo "  • Kibana: http://localhost:5601"
echo "  • Elasticsearch: http://localhost:9200"
echo ""
STATUSEOF

chmod +x ~/honeypot/check_status.sh
```

Run with: `~/honeypot/check_status.sh`

---

## 🌐 Access Points

| Service | URL | Credentials | Purpose |
|---------|-----|-------------|---------|
| **SNARE Honeypot** | http://localhost:8080 | None | Target for attacks |
| **Kibana Dashboard** | http://localhost:5601 | None | Visualize attacks |
| **Elasticsearch API** | http://localhost:9200 | user: `fyp`<br>pass: `fyp` | Query attack data |
| **TANNER API** | http://localhost:8090 | None | Honeypot brain |
| **Telegram** | Telegram App | Your bot | Real-time alerts |

---

## 🧪 Testing & Monitoring

### Quick Attack Test
```bash
# Single SQL injection test
curl "http://localhost:8080/login?user=admin'+OR+'1'='1"

# Wait for processing
sleep 10

# Check if detected
curl -s -u fyp:fyp "http://localhost:9200/honeypot-attacks-*/_search?size=1&sort=@timestamp:desc" | jq '.hits.hits[]._source | {type: .attack.type, severity: .event.severity}'
```

### Run Attack Campaigns
```bash
# Make executable (first time only)
chmod +x ~/honeypot/attack_campaigns.sh

# Run campaign
~/honeypot/attack_campaigns.sh

# Campaign options:
# 1. SQL Injection Campaign (15+ attacks)
# 2. XSS Campaign (10+ attacks)
# 3. File Inclusion Campaign (LFI/RFI)
# 4. Command Injection Campaign
# 5. Directory Traversal Campaign
# 6. APT Simulation (Multi-stage)
# 7. WAF Bypass Techniques
# 8. Run All Campaigns
# 9. Rapid Fire Test (Quick validation)
```

### Monitor Attacks in Real-Time

**Option 1: Watch Logstash Output**
```bash
docker logs -f logstash | grep -i "attack"
```

**Option 2: Watch Elasticsearch**
```bash
watch -n 5 'curl -s -u fyp:fyp "http://localhost:9200/honeypot-attacks-*/_count" | jq'
```

**Option 3: Watch Telegram Alerter**
```bash
docker logs -f telegram_alerter
```

**Option 4: Real-time Dashboard Script**
```bash
cat > ~/honeypot/monitor_attacks.sh << 'MONITOREOF'
#!/bin/bash

while true; do
    clear
    echo "================================"
    echo "   LIVE ATTACK MONITORING"
    echo "================================"
    echo ""
    
    # Total count
    TOTAL=$(curl -s -u fyp:fyp "http://localhost:9200/honeypot-attacks-*/_count" 2>/dev/null | grep -o '"count":[0-9]*' | grep -o '[0-9]*')
    echo "📊 Total Attacks: ${TOTAL:-0}"
    
    echo ""
    echo "🔴 Recent Attacks (Last 10):"
    curl -s -u fyp:fyp "http://localhost:9200/honeypot-attacks-*/_search?size=10&sort=@timestamp:desc" 2>/dev/null | \
        jq -r '.hits.hits[]._source | "\(.event.severity | ascii_upcase) | \(.attack.type) | \(.source.ip) | \(.url.path)"' 2>/dev/null | \
        head -10
    
    echo ""
    echo "Refreshing in 5 seconds... (Ctrl+C to exit)"
    sleep 5
done
MONITOREOF

chmod +x ~/honeypot/monitor_attacks.sh
```

Run with: `~/honeypot/monitor_attacks.sh`

---

## 📱 Telegram Alerts

### Alert Levels

| Emoji | Severity | Risk Score | Examples |
|-------|----------|------------|----------|
| 🔴 | **CRITICAL** | 90-99 | SQL Injection, Command Injection, Webshells |
| 🟠 | **HIGH** | 80-89 | XSS, LFI, Directory Traversal |
| 🟡 | **MEDIUM** | 60-79 | Scanner Detection, Admin Path Access |

### Configure Alerts
```bash
# Edit Telegram settings
nano ~/honeypot/telegram/telegram_alerter.py

# Update these values:
TELEGRAM_BOT_TOKEN = "YOUR_BOT_TOKEN"
TELEGRAM_CHAT_ID = "YOUR_CHAT_ID"
CHECK_INTERVAL = 10  # Check every 10 seconds
MAX_ALERTS_PER_MINUTE = 10  # Rate limiting
```

### Restart Alerter After Changes
```bash
cd ~/honeypot/elk
docker-compose restart telegram-alerter
docker logs -f telegram-alerter
```

### Test Telegram Bot
```bash
# Send test message
curl -X POST "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/sendMessage" \
  -d "chat_id=<YOUR_CHAT_ID>&text=Test from honeypot system"
```

---

## 🔧 Common Tasks

### View Recent Attacks
```bash
# Last 20 attacks
curl -s -u fyp:fyp "http://localhost:9200/honeypot-attacks-*/_search?size=20&sort=@timestamp:desc&pretty" | less

# Last 10 with specific fields
curl -s -u fyp:fyp "http://localhost:9200/honeypot-attacks-*/_search?size=10&sort=@timestamp:desc" | \
  jq '.hits.hits[]._source | {
    time: ."@timestamp",
    type: .attack.type,
    severity: .event.severity,
    ip: .source.ip,
    path: .url.path
  }'
```

### Search by Attack Type
```bash
# SQL Injection attacks
curl -s -u fyp:fyp -X POST "http://localhost:9200/honeypot-attacks-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{"query": {"match": {"attack.type": "SQL Injection"}}, "size": 10}'

# Critical severity attacks
curl -s -u fyp:fyp -X POST "http://localhost:9200/honeypot-attacks-*/_search?pretty" \
  -H 'Content-Type: application/json' \
  -d '{"query": {"match": {"event.severity": "critical"}}, "size": 10}'
```

### Get Attack Statistics
```bash
# Attack type breakdown
curl -s -u fyp:fyp -X POST "http://localhost:9200/honeypot-attacks-*/_search?size=0&pretty" \
  -H 'Content-Type: application/json' \
  -d '{
    "aggs": {
      "attack_types": {
        "terms": {"field": "attack.type.keyword", "size": 20}
      },
      "severity": {
        "terms": {"field": "event.severity.keyword"}
      },
      "top_ips": {
        "terms": {"field": "source.ip.keyword", "size": 10}
      }
    }
  }'
```

### View Container Logs
```bash
# Honeypot logs
docker logs -f snare
docker logs -f tanner

# ELK logs
docker logs -f elasticsearch
docker logs -f logstash
docker logs -f kibana

# Telegram alerter
docker logs -f telegram_alerter

# View last 50 lines
docker logs --tail 50 logstash

# View logs with timestamps
docker logs -f --timestamps logstash
```

### Restart Components
```bash
# Restart Logstash (after config change)
cd ~/honeypot/elk
docker-compose restart logstash

# Restart Telegram alerter
docker-compose restart telegram-alerter

# Restart honeypot
cd ~/honeypot
docker-compose restart snare tanner

# Restart everything
cd ~/honeypot/elk && docker-compose restart
cd ~/honeypot && docker-compose restart
```

### Clean Up Old Data
```bash
# List indices
curl -s -u fyp:fyp "http://localhost:9200/_cat/indices/honeypot-*?v&s=index"

# Delete old index (e.g., from 7 days ago)
curl -X DELETE -u fyp:fyp "http://localhost:9200/honeypot-attacks-2026.01.01"

# Delete indices older than 7 days (manual)
for i in {1..7}; do
  OLD_DATE=$(date -d "$i days ago" +%Y.%m.%d)
  curl -X DELETE -u fyp:fyp "http://localhost:9200/honeypot-attacks-$OLD_DATE" 2>/dev/null
done
```

### Backup Configuration
```bash
# Create backup
cd ~
zip -r honeypot-backup-$(date +%Y%m%d).zip honeypot \
  -x "honeypot/elk/data/*" \
  -x "honeypot/tanner/docker/log/*.log"

# Backup location
ls -lh ~/honeypot-backup-*.zip
```

### Update System
```bash
# Pull latest images
cd ~/honeypot
docker-compose pull

cd ~/honeypot/elk
docker-compose pull

# Restart with new images
docker-compose down
docker-compose up -d
```

---

## 🔍 Troubleshooting

### Problem: No Attacks Detected

**Check 1: Is honeypot accessible?**
```bash
curl -v http://localhost:8080
```

**Check 2: Are logs being created?**
```bash
ls -lh ~/honeypot/tanner/docker/log/
tail -20 ~/honeypot/tanner/docker/log/tanner_report.json
```

**Check 3: Is Logstash processing?**
```bash
docker logs logstash 2>&1 | grep "Pipeline started"
docker logs logstash --tail 50 | grep -i "attack"
```

**Check 4: Test with known attack**
```bash
curl "http://localhost:8080/test?id=1'+OR+'1'='1"
sleep 10
curl -s -u fyp:fyp "http://localhost:9200/honeypot-attacks-*/_count" | jq
```

### Problem: Telegram Not Alerting

**Check 1: Is alerter running?**
```bash
docker ps | grep telegram
docker logs telegram_alerter --tail 20
```

**Check 2: Are there new attacks?**
```bash
# Look for "Found X NEW attacks" in logs
docker logs telegram_alerter | grep "Found"
```

**Check 3: Test Telegram connectivity**
```bash
# Replace with your actual values
curl -X POST "https://api.telegram.org/bot<YOUR_TOKEN>/sendMessage" \
  -d "chat_id=<YOUR_CHAT_ID>&text=Test message"
```

**Solution: Restart alerter**
```bash
cd ~/honeypot/elk
docker-compose restart telegram-alerter
docker logs -f telegram-alerter
```

### Problem: Port Already in Use

**Check what's using the port:**
```bash
sudo lsof -i :8080  # SNARE
sudo lsof -i :5601  # Kibana
sudo lsof -i :9200  # Elasticsearch
```

**Kill process using port:**
```bash
sudo kill -9 $(sudo lsof -t -i:8080)
```

**Or stop system Elasticsearch (if installed):**
```bash
sudo systemctl stop elasticsearch
sudo systemctl stop kibana
sudo systemctl disable elasticsearch
sudo systemctl disable kibana
```

### Problem: Logstash Configuration Error

**Check for errors:**
```bash
docker logs logstash 2>&1 | grep -i "error\|exception"
```

**Common errors:**
- **"Unknown setting"**: Typo in config
- **"Expected one of"**: Syntax error, missing brace

**Test config:**
```bash
# Validate pipeline config
cat ~/honeypot/elk/logstash/pipeline/honeypot-pipeline.conf
```

**Restore from backup:**
```bash
cp ~/honeypot/elk/logstash/pipeline/honeypot-pipeline.conf.backup \
   ~/honeypot/elk/logstash/pipeline/honeypot-pipeline.conf
```

### Problem: High Memory Usage

**Check memory:**
```bash
free -h
docker stats --no-stream
```

**Reduce Elasticsearch heap:**
```bash
# Edit docker-compose.yml
nano ~/honeypot/elk/docker-compose.yml

# Change:
ES_JAVA_OPTS: "-Xms1g -Xmx1g"  # Was 2g

# Restart
docker-compose restart elasticsearch
```

### Problem: Kibana Not Loading

**Check if Elasticsearch is ready:**
```bash
curl -u fyp:fyp http://localhost:9200/_cluster/health
```

**Wait for Kibana (takes 1-2 minutes):**
```bash
docker logs -f kibana
# Look for: "Kibana is now available"
```

**Force restart:**
```bash
cd ~/honeypot/elk
docker-compose restart kibana
```

### Emergency Reset

**Complete reset (DELETES ALL DATA):**
```bash
# Stop everything
cd ~/honeypot/elk && docker-compose down -v
cd ~/honeypot && docker-compose down -v

# Remove all containers
docker rm -f $(docker ps -aq)

# Remove volumes (WARNING: Deletes all data)
docker volume prune -f

# Restart
cd ~/honeypot && docker-compose up -d
cd ~/honeypot/elk && docker-compose up -d
```

---

## 📊 Kibana Dashboard Setup

### First Time Setup

1. **Access Kibana**: http://localhost:5601

2. **Create Index Pattern:**
   - Go to: Management → Stack Management → Index Patterns
   - Click "Create index pattern"
   - Pattern: `honeypot-attacks-*`
   - Time field: `@timestamp`
   - Click "Create"

3. **View Data:**
   - Go to: Analytics → Discover
   - Select `honeypot-attacks-*`
   - See real-time attacks

4. **Create Visualizations:**
   - Pie Chart: Attack types distribution
   - Bar Chart: Attacks over time
   - Data Table: Top attacker IPs
   - Metric: Critical attacks count
   - Map: Geographic attack sources

5. **Save Dashboard:**
   - Go to: Analytics → Dashboard
   - Click "Create dashboard"
   - Add your visualizations
   - Save as "Honeypot Attacks Overview"

---

## 🔐 Security Notes

⚠️ **Important:**
- **Never expose directly to internet** without proper protection
- **Change default passwords** in production
- **Keep bot token secret** - don't commit to repositories
- **Monitor resource usage** - prevent DoS on your system
- **Regular backups** - save configurations and important data
- **Legal compliance** - ensure you have authorization to deploy

---

## 📞 Quick Commands Reference
```bash
# ============ START/STOP ============
cd ~/honeypot && docker-compose up -d
cd ~/honeypot/elk && docker-compose up -d
cd ~/honeypot/elk && docker-compose down
cd ~/honeypot && docker-compose down

# ============ STATUS ============
docker ps
~/honeypot/check_status.sh
docker logs telegram_alerter --tail 20

# ============ TESTING ============
~/honeypot/attack_campaigns.sh
curl "http://localhost:8080/test?id=1'+OR+'1'='1"

# ============ MONITORING ============
~/honeypot/monitor_attacks.sh
docker logs -f logstash | grep attack
curl -s -u fyp:fyp "http://localhost:9200/honeypot-attacks-*/_count" | jq

# ============ RESTART ============
cd ~/honeypot/elk && docker-compose restart logstash
cd ~/honeypot/elk && docker-compose restart telegram-alerter

# ============ LOGS ============
docker logs -f snare
docker logs -f tanner
docker logs -f logstash
docker logs -f telegram_alerter
```

---

## 📚 Additional Resources

- **Full Manual**: See `README.md` for comprehensive documentation
- **Logstash Config**: `~/honeypot/elk/logstash/pipeline/honeypot-pipeline.conf`
- **Telegram Script**: `~/honeypot/telegram/telegram_alerter.py`
- **Attack Scripts**: `~/honeypot/attack_campaigns.sh`

---

**Last Updated**: 2026-01-05  
**System Version**: 1.0
