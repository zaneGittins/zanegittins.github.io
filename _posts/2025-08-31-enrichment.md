---
layout: post
title: "Enriching Sysmon with Velociraptor"
date: 2025-08-31
---

The difference between a good detection and a great one often comes down to having the right context at the right time. With [Velociraptor](https://docs.velociraptor.app/) we can enrich logs with additional data before they are sent to the SIEM. These enriched logs allow analysts to better triage detections, and for detection engineers to write new and improved rules.

Sysmon is a powerful observability tool commonly used by defenders to monitor critical telemetry not present in Windows by default. Sysmon allows for tracking key events such as process creation, network connections, DNS requests, file creations, and more. However, Sysmon development has stagnated in recent years, the last new feature was added in June 2023. Using Velociraptor as our log collector, we can breathe new life into Sysmon and tailor telemetry to individual use cases.

For example, we can add [TLSH hashes](https://blog.ecapuano.com/p/the-role-of-fuzzy-hashes-in-security) and authenticode signatures onto process creations, and commandlines onto network connection events. Using Velociraptor's powerful query language we can easily extend any event to add new and interesting fields. 

Even for those who have EDR in place, Velociraptor remains a compelling piece of the observability pipeline. EDRs often transparently sample events, and users have little to no control over the log schema. Using Velociraptor we can gather and enrich event logs, and even tap into lower level telemetry sources such as [ETW](https://docs.velociraptor.app/docs/gui/debugging/vql/plugins/etw/).

## Example

I've previously released an example of enriching Sysmon process creations to the [artifact exchange](https://docs.velociraptor.app/exchange/artifacts/pages/windows.eventlogs.sysmonprocessenriched/).


First, I added the authenticode signature to process creations, telling us if this is a signed and trusted binary, along with signature information. Checking the authenticode signature of a process is often one of the first steps defenders take, and now that information is readily available instead of requiring follow-up action.

```sql
LET get_auth_cache(Image) = SELECT authenticode(filename=Image) AS Authenticode
    FROM scope()
```

Additionally, I add TLSH hashes, opening up new detection opportunities. Any time you receive a new malware detection, you can add it to a corpus of fuzzy hashes, comparing the process creation hashes against this list to find similar executables running in your environment.

```sql
LET get_tlsh_cache(Image) = SELECT tlsh_hash(path=Image) AS TLSH
    FROM scope()
```

Lastly, I also add on the full process chain by leveraging Velociraptor's process tracker. This saves analysts valuable time by showing them the full process ancestry immediately, rather than the analyst manually making multiple queries to walk up the process chain.

```sql
 join(
array=process_tracker_callchain(
    id=EventData.ProcessId).Data.Name,
sep="->") AS CallChain
```

Viewing sample events in our SIEM we can view all the new fields which were added to our process creations.

<div class="centered-image">
  <img src="/assets/images/enriched_events.png" alt="Enriched Sysmon Events">
  <p>Enriched Sysmon events</p>
</div>

The enriched events we receive are far superior to the default Sysmon logs. They supply analysts with the context they need to close alerts, and open up new possibilities for detections. We can easily apply similar enrichments to other events, such as adding commandline information to Sysmon network connection events. None of the VQL used here is overly complex, my hope is that readers use this as a template to create their own artifacts to enrich other Sysmon events and Windows log sources.

## Performance

For those managing large fleets of endpoints you may worry about the performance impact of adding these additional fields. After all, we are collecting authenticode signatures and TLSH hashes in realtime on each process execution. In my testing, the performance impact has been negligible due to intelligent caching mechanisms. We cache both the authenticode signature and TLSH hash using the standard Sysmon hash field as a lookup key. This approach ensures that expensive operations are performed only once per unique executable. When the same binary launches multiple times, subsequent events leverage cached results rather than consuming CPU cycles.

## Getting Started

For those new to Velociraptor I wanted to share these brief steps to start collecting the enriched process creation events mentioned above.

1. First, import the [Process Creation enriched](https://docs.velociraptor.app/exchange/artifacts/pages/windows.eventlogs.sysmonprocessenriched/) sample artifact by either running `Server.Import.ArtifactExchange` or manually copying the artifact to your server.

2. Add the artifact to the client monitoring table, ensuring you have the process tracker enabled.

<div class="centered-image">
  <img src="/assets/images/monitoring.png" alt="Monitoring table">
  <p>Monitoring table</p>
</div>

That's it! Enriched sysmon events should now begin streaming into your server. You can forward these to a SIEM by using a server monitoring artifact such as [Elastic.Events.Upload](https://docs.velociraptor.app/artifact_references/pages/elastic.events.upload/). 

## Conclusion

While Sysmon's development may have slowed, its core value remains strong. By leveraging Velociraptor's powerful query language, we can enhance Sysmon telemetry with new data. The enrichment strategies discussed here, from TLSH hashes to process call chains, represent just the beginning of what's possible. As threats evolve, so must the telemetry we monitor. Velociraptor gives us the tools to do exactly that, regardless of Sysmon's development timeline.