# Splunk SPL Detection Queries

All search queries used during the SSH brute-force detection lab.

## Baseline Queries

### All events from Linux logs index

```
index=linux_logs
```

### Failed authentication events

```
index=linux_logs "Failed password"
```

### Successful authentication events

```
index=linux_logs "Accepted password"
```

## Detection Queries

### Top attacker source IPs

```
index=linux_logs "Failed password" 
| rex "from (?<src_ip>\d+\.\d+\.\d+\.\d+)" 
| stats count by src_ip 
| sort -count
```

### Top targeted users

```
index=linux_logs "Failed password" 
| rex "for (?:invalid user )?(?<user>\S+) from" 
| stats count by user 
| sort -count
```

### Authentication summary by user (failed vs accepted)

```
index=linux_logs ("Failed password" OR "Accepted password") 
| rex "(?<status>Failed|Accepted) password" 
| rex "for (?:invalid user )?(?<user>\S+) from" 
| stats count by user status 
| sort -count
```

### Attack timeline (failed logins per minute)

```
index=linux_logs "Failed password" 
| timechart span=1m count
```

### Attack duration (start, end, total minutes)

```
index=linux_logs "Failed password" 
| stats earliest(_time) as start latest(_time) as end 
| eval duration_seconds=end-start 
| eval duration_minutes=round(duration_seconds/60, 2) 
| eval start_time=strftime(start, "%H:%M:%S") 
| eval end_time=strftime(end, "%H:%M:%S") 
| table start_time end_time duration_minutes
```

### Compromise detection (successful logins from attacker IP)

```
index=linux_logs "Accepted password" 
| rex "from (?<src_ip>\d+\.\d+\.\d+\.\d+)" 
| stats count by src_ip user
```

## Threshold Concept (Future Alert)

A threshold-based alert could be configured to trigger when:

- More than 50 failed logins from a single source IP within 5 minutes, OR
- More than 5 distinct usernames attempted from the same source IP within 10 minutes, OR
- Any successful login from an IP that previously generated failed login events