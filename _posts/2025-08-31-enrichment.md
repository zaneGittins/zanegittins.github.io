---
layout: post
title: "Enriching Sysmon with Velociraptor"
date: 2025-08-31
---

The cornerstone of effective security monitoring is having the right context at the right time. Using [Velociraptor](https://docs.velociraptor.app/) we can enrich logs before they are sent to the SIEM. The additional context is available to analysts immediately, and enables detection engineers to write new and improved rules.

Sysmon is a powerful observability tool commonly used by defenders to monitor critical telemetry not present in Windows by default. Sysmon tracks key events needed to trace threat actor activity such as process creation, network connections, DNS requests, file creations, and more. However, Sysmon development has stagnated in recent years, the last new feature was added in June of 2023. Using Velociraptor as our log collector, we can breathe new life into Sysmon by adding valuable context to an already powerful log source.

For example, we can add [TLSH hashes](https://blog.ecapuano.com/p/the-role-of-fuzzy-hashes-in-security) and authenticode signatures to process creations, and staple on commandlines to network connection events. Using Velociraptor's powerful query language we can easily extend events to add valuable context.

Even for those who have EDR in place, Velociraptor remains a compelling piece of the observability pipeline. EDRs often transparently sample events, occasionally dropping important telemetry. Users also have little to no control over the log schema, and are therefore stuck with a cookie cutter approach to security logging. Using Velociraptor we can gather and enrich event logs, and even tap into lower level telemetry sources such as [ETW](https://docs.velociraptor.app/docs/gui/debugging/vql/plugins/etw/). Velociraptor provides us with a deep level of customization through its query language (VQL), allowing us to gather almost any log source, and enrich those logs in realtime.

## Example

I've previously released an example of enriching Sysmon process creations to the [artifact exchange](https://docs.velociraptor.app/exchange/artifacts/pages/windows.eventlogs.sysmonprocessenriched/).


In this artifact I add the authenticode signature to process creations, telling us if this is a signed and trusted binary, along with signature information. Checking the authenticode signature of a process is often one of the first steps defenders take, and now that information is readily available instead of requiring follow-up action. This also adds new detection opportunities, we can now write rules based on if a binary is trusted.

```sql
LET get_auth_cache(Image) = SELECT authenticode(filename=Image) AS Authenticode
    FROM scope()
```

I also add [TLSH](https://tlsh.org/) fuzzy hashes, opening up new detection opportunities. TLSH enables similarity matching by calculating the distance between hashes, making it possible to identify malware variants even when attackers have modified the original code. These can be leveraged in proactive hunting for known threats.

```sql
LET get_tlsh_cache(Image) = SELECT tlsh_hash(path=Image) AS TLSH
    FROM scope()
```

Lastly, I tack on the full process chain by leveraging Velociraptor's process tracker. This saves analysts valuable time by showing them the full process ancestry immediately, rather than the analyst manually making multiple queries to walk up the process chain.

```sql
 join(
array=process_tracker_callchain(
    id=EventData.ProcessId).Data.Name,
sep="->") AS CallChain
```

Viewing sample events in our SIEM, we can see all the new fields which were added to our process creations.

<div class="centered-image">
  <img src="/assets/images/enriched_events.png" alt="Enriched Sysmon Events">
  <p>Enriched Sysmon events</p>
</div>

The enriched events we receive are far superior to default Sysmon logs. They supply analysts with the context they need, and open up new possibilities for detections. We can easily apply similar enrichments to other events, such as adding commandline information to Sysmon network connection events. The VQL used to enrich events is straightforward and intuitive, and the same ideas could easily be applied to other Sysmon events and Windows log sources.

## Performance

For those managing large fleets of endpoints you may worry about the performance impact of adding these additional fields. After all, we are collecting authenticode signatures and TLSH hashes in realtime on each process execution. In my testing, the performance impact has been negligible due to intelligent caching mechanisms. We cache both the authenticode signature and TLSH hash using the standard Sysmon hash field as a lookup key. This approach ensures that expensive operations are performed only once per unique executable. When the same binary launches multiple times, subsequent events leverage cached results rather than consuming CPU cycles. 

## Conclusion

While Sysmon's development may have slowed, its core value remains strong. By leveraging Velociraptor's powerful query language, we can enhance Sysmon telemetry with new data. The enrichment strategies discussed here, from TLSH hashes to process call chains, represent just the beginning of what's possible. As threats evolve, so must the telemetry we monitor. Velociraptor gives us the tools to do exactly that, regardless of Sysmon's development timeline.