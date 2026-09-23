# FortiGate NOC Dashboard for Grafana

A single dashboard covering SD-WAN link health, interface throughput, firewall policies,
and UTM security events (IPS, Web Filter, SSL Inspection) — 50 panels across 8 rows.

Metrics come from Prometheus, security events from Loki. Both are required: the FortiGate
REST API exposes no metrics at all for IPS, Web Filter or SSL Inspection, because those
subsystems emit log events rather than counters.

![rows](Live Status · SD-WAN / ISP Quality · Traffic & Interfaces · Threat Activity · Web Filter · SSL Inspection · Top Talkers & Policies · System Health)

---

## What you need

| Component | Purpose |
|---|---|
| [prometheus-community/fortigate_exporter](https://github.com/prometheus-community/fortigate_exporter) | SD-WAN health, interfaces, policies, VPN, certificates, system |
| Loki | Stores FortiGate syslog |
| Grafana Alloy | Ships logs to Loki (Promtail reached end of life in March 2026) |
| rsyslog | Receives syslog from the firewall |

---

## 1. Metrics pipeline

Create a read-only REST API admin on the FortiGate. The `netgrp` read permission is not
optional — without it the SD-WAN metrics are silently missing, which is the most common
reason people see empty ISP panels.

```
config system accprofile
    edit "prom-monitor"
        set authgrp read
        set fwgrp custom
        set loggrp custom
        set netgrp custom
        set sysgrp custom
        set vpngrp read
        config netgrp-permission
            set cfg read
            set route-cfg read
        end
        config fwgrp-permission
            set policy read
            set others read
        end
        config sysgrp-permission
            set cfg read
        end
    next
end
```

Then **System → Administrators → Create New → REST API Admin**, bind the profile, set
Trusted Hosts to your monitoring server, and copy the token.

Exporter auth file:

```yaml
"https://firewall.example.com":
  token: YOUR_TOKEN
  probes:
    exclude:
      - BGP           # drop these only if the feature is unconfigured;
      - OSPF          # otherwise the probe reports a failure every scrape
      - Log/Fortianalyzer
```

Prometheus job — note the multi-target `/probe` pattern:

```yaml
  - job_name: "fortigate"
    metrics_path: /probe
    scrape_interval: 60s
    scrape_timeout: 30s
    static_configs:
      - targets: ["https://firewall.example.com"]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
        regex: '(?:.+)(?::\/\/)([^:]*).*'
      - target_label: __address__
        replacement: '127.0.0.1:9710'
```

A 60s interval with a 30s timeout is deliberate. One scrape fans out into many API calls,
and the default 15s global interval will time out on smaller appliances.

---

## 2. Log pipeline

On the FortiGate:

```
config log syslogd setting
    set status enable
    set server "10.0.0.50"
    set port 514
    set mode udp
    set facility local7
    set format default
end

config log syslogd filter
    set severity information
    set forward-traffic enable
    set local-traffic disable
end
```

`format default` matters — the dashboard parses FortiGate's `key=value` format with
`| logfmt`. CEF or CSV will break every log panel.

rsyslog receiver (`/etc/rsyslog.d/10-fortigate.conf`):

```
template(name="FortigateRaw" type="string" string="%rawmsg-after-pri%\n")

if ($fromhost-ip == '10.0.0.1') then {
    action(type="omfile"
           file="/var/log/fortigate/fortigate.log"
           template="FortigateRaw"
           fileOwner="syslog"
           fileGroup="adm"
           fileCreateMode="0640")
    stop
}
```

Two details worth copying: `%rawmsg-after-pri%` keeps the `date=` field that rsyslog's
default template strips, and the trailing `stop` prevents every firewall log from also
landing in `/var/log/syslog`.

Alloy config (`/etc/alloy/config.alloy`):

```alloy
local.file_match "fortigate" {
  path_targets = [{
    __path__ = "/var/log/fortigate/fortigate.log",
    job      = "fortigate",
  }]
}

loki.source.file "fortigate" {
  targets    = local.file_match.fortigate.targets
  forward_to = [loki.process.fortigate.receiver]
}

loki.process "fortigate" {
  stage.logfmt {
    mapping = { "type" = "", "subtype" = "", "level" = "", "action" = "" }
  }
  stage.labels {
    values = { type = "", subtype = "", level = "", action = "" }
  }
  forward_to = [loki.write.default.receiver]
}

loki.write "default" {
  endpoint { url = "http://127.0.0.1:3100/loki/api/v1/push" }
}
```

Only four low-cardinality fields become labels. Everything else — `srcip`, `hostname`,
`url`, `catdesc`, `attack`, `sni` — is parsed at query time. Promoting those to labels
would give you a stream explosion and a very unhappy Loki.

Add `alloy` to the `adm` group so it can read the log file.

---

## 3. Import

Grafana → **Dashboards → New → Import** → upload the JSON → pick your Prometheus and
Loki data sources.

Variables: **FortiGate** selects the device, **Interface** filters the throughput panels
(worth narrowing to your active ports), **Filters** is an ad-hoc Loki filter for
drilling into a single source IP or hostname.

---

## Notes

**Volume.** Traffic logs dominate by roughly 30:1 over UTM events. If disk is tight, set
`forward-traffic disable` on the firewall — the IPS, Web Filter and SSL panels keep
working, only Top Talkers goes quiet.

**IPS profiles.** The built-in monitor-only sensors detect without blocking, so the
"IPS Action" panel will show `detected` rather than `blocked`. That is the profile
behaving as designed, not a dashboard bug.

**Geomaps.** Country lookup uses Grafana's built-in gazetteer against FortiGate's
`srccountry` / `dstcountry` fields. Internal RFC1918 traffic is reported as `Reserved`
and simply doesn't plot.

**Query cost.** The `topk` panels use `$__range`, so they recompute across the whole
selected window. On a busy firewall a 7-day range is expensive — narrow the range rather
than the refresh interval.
