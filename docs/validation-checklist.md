# Build record and validation checklist

[Project overview](../README.md) · [Deployment guide](deployment-guide.md)

**Deployment status: Not yet recorded.** Check a box only after observing the result in your own lab.

## Environment record

| Field | Actual value |
| --- | --- |
| Build date | Pending |
| Hypervisor and host architecture | Pending |
| Ubuntu version and IP | Pending |
| Windows version, hostname, and IP | Pending |
| Elasticsearch version | Pending |
| Kibana version | Pending |
| Fleet Server agent version | Pending |
| Windows agent version | Pending |
| Sysmon version | Pending |

## Acceptance checks

- [ ] Both VMs communicate over the private network.
- [ ] All four Elastic components use the chosen matching version.
- [ ] Elasticsearch responds to an authenticated HTTPS request with CA validation.
- [ ] Kibana enrollment and login succeed.
- [ ] Fleet Server is Healthy.
- [ ] Windows reaches Ubuntu on TCP 8220 and 9200.
- [ ] Windows agent is Healthy under Windows-Lab-Policy.
- [ ] Sysmon event 1 is generated locally and searchable in Discover.
- [ ] PowerShell event 4104 is generated locally and searchable in Discover.
- [ ] After reboot, services recover and newly generated events arrive.
- [ ] Screenshots and logs have been checked for exposed secrets before committing.

## Evidence to capture

| Evidence | Expected content | Result |
| --- | --- | --- |
| Package versions | Matching installed versions | Pending |
| Cluster health | Available primary shards; investigate red health | Pending |
| Fleet agents | Ubuntu Fleet Server and Windows endpoint Healthy | Pending |
| Sysmon search | Event 1 for harmless notepad process creation | Pending |
| PowerShell search | Event 4104 for the validation command | Pending |
| Reboot check | Fresh events after restart | Pending |

## Deviations and unresolved issues

Record changed versions, configuration differences, errors, resolutions, and any failed checkpoint here. Use redacted evidence and timestamps. Do not put credentials in this record.
