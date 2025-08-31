---
layout: post
title: "Enriching Sysmon with Velociraptor"
date: 2025-08-31
---

The difference between a good detection and a great one often comes down to having the right context at the right time. With [Velociraptor](https://docs.velociraptor.app/) we can enrich logs with additional data before they are sent to the SIEM. These enriched logs allow analysts to better triage detections, and for detection engineers to write new and improved rules.

Sysmon is a powerful observability tool commonly used by defenders to monitor critical telemetry not present in Windows by default. Sysmon allows for tracking key events such as process creation, network connections, DNS requests, file creations, and more. However, Sysmon development has stagnated in recent years, the last meaningful Windows contribution that added a new feature was in June 2023. Using Velociraptor as our log collector, we can breathe new life into Sysmon and tailor telemetry to individual use cases.

For example, we can add [TLSH hashes](https://blog.ecapuano.com/p/the-role-of-fuzzy-hashes-in-security) and authenticode signatures onto process creations, and commandlines onto network connection events. Using Velociraptor we can easily extend any event to add new and interesting fields.

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

The enriched events we recieve are far superior to the default Sysmon logs. They supply analysts with the context they need to close alerts, and open up new possibilities for detections. We can easily apply similar enrichments to other events, such as adding commandline information to Sysmon network connection events.

## Conclusion

While Sysmon's development may have slowed, its core value remains strong. By leveraging Velociraptor's powerful query language, we can enhance Sysmon telemetry with new data. The enrichment strategies discussed here, from TLSH hashes to process call chains, represent just the beginning of what's possible. As threats evolve, so must the telemetry we monitor. Velociraptor gives us the tools to do exactly that, regardless of Sysmon's development timeline. 