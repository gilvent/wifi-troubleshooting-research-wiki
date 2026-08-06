---
title: WiFi Troubleshooting Data Collection — Industry Mechanisms
tags: [telemetry, data-collection, aiops, cloud-wlan, sensors, standards, deep-research]
sources:
  - https://www.juniper.net/documentation/us/en/software/mist/mist-wireless/shared-content/topics/concept/service-level-expectations.html
  - https://www.mist.com/documentation/dynamic-packet-captures/
  - https://www.mist.com/documentation/ap-connectivity-to-the-mist-cloud/
  - https://www.juniper.net/documentation/us/en/software/mist/mist-aiops/topics/concept/marvis-client-overview.html
  - https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Troubleshooting_and_Support/Troubleshooting/Packet_Capture_Overview
  - https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Meraki_Health_Overview
  - https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/catalyst-center-assurance/2-3-7/b_cisco_catalyst_assurance_2_3_7_ug/b_cisco_catalyst_assurance_2_3_6_ug_chapter_01110.html
  - https://arubanetworking.hpe.com/techdocs/central/2.5.7/content/nms/clients/clients-live-events.htm
  - https://docs.nilesecure.com/nile-slas
  - https://7signal.com/platform/sensors/
  - https://wyebot.com/platform/faq/
  - https://help.capenetworks.com/en/articles/3295755-faq-s
  - https://docs.thousandeyes.com/product-documentation/global-vantage-points/endpoint-agents/how-endpoint-agents-work/how-does-the-endpoint-agent-work.md
  - https://www.wi-fi.org/system/files/Wi-Fi_CERTIFIED_Data_Elements_Technology_Overview_202401.pdf
  - https://www.broadband-forum.org/pdfs/tr-369-1-3-0.pdf
  - https://github.com/plume-design/opensync
  - https://github.com/Telecominfraproject/wlan-cloud-ucentralgw/blob/main/PROTOCOL.md
  - https://github.com/prplfoundation/prplMesh/blob/master/documentation/flows/agent/ap_metrics.md
  - https://blogs.cisco.com/networking/why-the-802-11k-and-neighbor-report-are-important
updated: 2026-07-14
---

# WiFi Troubleshooting Data Collection — Industry Mechanisms

*Survey of how commercial AI/cloud WiFi-troubleshooting systems and open telemetry standards actually collect their data: what is measured, from which vantage point, over what transport, and — critically for this wiki's research direction — when collection escalates.*

> Sources: vendor documentation and datasheets (Juniper Mist, Cisco Meraki, Cisco Catalyst Center, HPE Aruba Central/UXI, Extreme CloudIQ, Nile, 7SIGNAL, Wyebot, ThousandEyes), open specs and reference implementations (Wi-Fi Alliance Data Elements Jan-2024 overview, Broadband Forum TR-369/TR-181/TR-157, OpenSync `src/sm`, TIP OpenWiFi `PROTOCOL.md`, prplMesh `ap_metrics.md`), and [the research direction](edge-diagnostic-agent-research-direction.md) §3.1, §5, §9 for framing. Compiled 2026-07-14 via three parallel web-research passes; per-claim URLs inline. Unverifiable claims are flagged in §7.

## 1. The five collection mechanisms

Industry systems combine five distinct mechanisms. Almost every product is a particular mix of these, not a novel sixth.

| # | Mechanism | Vantage point | Airtime cost | Example |
|---|---|---|---|---|
| 1 | **Infrastructure event/stat streaming** — AP/controller continuously exports connection events + RF counters | AP serving radio, controller | ~zero (side effect of serving) | Mist ap-stats every 2 s; Meraki Health |
| 2 | **Dedicated scanning radio** — extra radio on the AP does passive scans, spectrum, WIPS | AP, non-serving radio | zero on serving channel | Mist 3rd/4th radio; Meraki Air Marshal radio; Nile 3rd radio |
| 3 | **Triggered deep capture** — packet capture armed by anomaly detection | AP (serving radio for OFDMA correctness) | storage/uplink, not airtime | Mist dynamic PCAP; Meraki Proactive PCAP; Cisco iCAP |
| 4 | **Synthetic testing** — a real or emulated client runs active tests (assoc/auth/DHCP/DNS/HTTP/VoIP) | Dedicated sensor, AP-as-client, or agent on endpoint | **real airtime + association slot** | 7SIGNAL Sapphire Eye; Aruba UXI; Marvis Minis; Nile sensors |
| 5 | **Client-side telemetry** — SDK/agent on the actual user device reads OS radio state | The client itself | ~zero (passive) or low (agent probes) | Mist Marvis Client + Zebra; ThousandEyes Endpoint Agent; 7SIGNAL Mobile Eye |

A sixth quasi-mechanism rides inside 802.11 itself: **802.11k beacon reports** let the AP ask an associated client to scan and report what *it* sees — client-perspective RF data with no installed agent ([Cisco](https://blogs.cisco.com/networking/why-the-802-11k-and-neighbor-report-are-important); EasyMesh wraps the same primitive in its Beacon Metrics Query, [prplMesh #399](https://github.com/prplfoundation/prplMesh/issues/399)). This is the only agentless path to the client's view, and it costs the client an on-air scan.

## 2. What is collected (the common data model)

Across vendors and standards the collected entities converge on the same shape — Radio → BSS → associated STA, plus neighbor scans and events:

- **Client connection state machine** — per-client association → authentication → DHCP → DNS/ARP steps with per-step timing and failure reason. This is the backbone of every AIOps product: Mist tracks "100+ states per device" feeding its SLE classifiers ([Mist SLE](https://www.juniper.net/documentation/us/en/software/mist/mist-wireless/shared-content/topics/concept/service-level-expectations.html)); Meraki Health scores the same four onboarding steps ([Meraki Health](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Operate_and_Maintain/Monitoring_and_Reporting/Meraki_Health_Overview)); Catalyst Center scores onboarding per phase (assoc/AAA/DHCP).
- **Roaming events** — roam success rate, roam latency (Mist targets ~200 ms for 802.11r vs ~2 s full roam), old-AP-dissociation-to-first-packet-on-new-AP delta (Meraki).
- **RF counters** — per-STA RSSI/SNR as seen by the AP, PHY rates, retries/errors; per-radio channel utilization split into own-BSS / other-WiFi / non-WiFi (exactly the `Radio.Transmit`/`ReceiveSelf`/`ReceiveOther` split in [Wi-Fi Data Elements](https://www.wi-fi.org/system/files/Wi-Fi_CERTIFIED_Data_Elements_Technology_Overview_202401.pdf)).
- **Neighbor/channel scans** — off-channel or scanning-radio surveys of neighboring BSSs and per-channel utilization.
- **Packet captures** — management/EAP/DHCP frames around failures (see §4); Mist explicitly never collects payloads, "only transmission and connection data" ([Mist pcap doc](https://www.juniper.net/documentation/us/en/software/mist/mist-wireless/shared-content/topics/topic-map/packet-captures.html)).
- **Application/flow telemetry** — on-AP DPI (Aruba AppRF, Meraki L7), NetFlow/IPFIX; used to link "the call broke" to the flow that broke.
- **Synthetic-test KPIs** — time-to-connect per step, ping/HTTP/throughput, VoIP MOS/jitter/loss (7SIGNAL layers 2–7 tests; UXI DHCP/DNS/portal/app timings).
- **Client-perspective data** (only where a client SDK exists) — device-measured RSSI, adapter/driver version, WiFi↔cellular handoffs, and **the client's own roam decisions** — Mist frames the Marvis Client explicitly as visibility into "client roaming algorithms since roaming is a client decision" ([Marvis Client](https://www.juniper.net/documentation/us/en/software/mist/mist-aiops/topics/concept/marvis-client-overview.html)).

## 3. Transport: how data reaches the analysis backend

- **Persistent TLS channel on 443 is the norm** for enterprise WLAN: Mist uses HTTPS to `ep-terminator.mistsys.net` with ap-stats every 2 s ([Mist cloud connectivity](https://www.mist.com/documentation/ap-connectivity-to-the-mist-cloud/)); Aruba Central uses a WebSocket; Meraki a proprietary AES-256 tunnel carrying protobufs, ~1 kbps/device idle management traffic ([Meraki](https://documentation.meraki.com/General_Administration/Other_Topics/Management_traffic_for_Cisco_Meraki_devices)); Extreme uses DTLS-secured CAPWAP.
- **MQTT is the de facto bus for ISP/home CPE**: OpenSync pushes protobuf stat reports over MQTT via its Queue Manager ([OpenSync whitepaper](https://discover.plume.com/rs/776-GZM-466/images/Plume-Opensync_for_the_connected_home.pdf)); Wyebot sensors upload metadata over MQTT/TLS ([Wyebot FAQ](https://wyebot.com/platform/faq/)); TR-369/USP lists MQTT, WebSocket, STOMP, CoAP as standard bindings ([TR-369](https://www.broadband-forum.org/pdfs/tr-369-1-3-0.pdf)).
- **On-prem/cloud splits exist where privacy demands it**: Catalyst Center collects on-prem via gRPC/gNMI model-driven telemetry from the 9800 WLC, then exports de-identified data over HTTPS to the Cisco AI cloud, opt-in ([Cisco admin guide](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/catalyst-center/3-2-x/admin-guide/cisco-catalyst-center-admin-guide-3-2-x/configure-external-services/t_configure_cisco_ai_network_analytics_data_collection.html)).
- **Every mature stack ends up hybrid push+pull**: periodic streaming for cheap stats, subscription/notify for events, on-demand query for deep data. USP replaced TR-069 polling with Notify subscriptions but kept Get; uCentral has periodic `state`/`healthcheck` messages plus a `telemetry` command that starts a 0–60 s-interval stream which **auto-stops after 30 minutes** ([uCentral PROTOCOL.md](https://github.com/Telecominfraproject/wlan-cloud-ucentralgw/blob/main/PROTOCOL.md)); EasyMesh has on-demand metric queries *and* interval reporting *and* threshold-triggered push, configured by a Metric Reporting Policy TLV ([prplMesh ap_metrics](https://github.com/prplfoundation/prplMesh/blob/master/documentation/flows/agent/ap_metrics.md)).
- **Telemetry buffering during WAN outage is a consistent documentation gap** — vendors document that the data plane survives cloud loss, none publish back-fill behavior.

## 4. Escalation: triggered deep collection (the load-bearing section)

The industry already runs a **two-tier sensing architecture**: cheap always-on telemetry plus expensive deep capture that is *gated*. Three vendors automate the escalation:

- **Mist dynamic PCAP** — any client connection failure "automatically triggers a short-term dynamic packet capture" stored in the cloud; trigger list includes DHCP timeout/denied, RADIUS failures, association failure, and notably **11r FBT failure** and OKC auth failure; retained 7 days ([dynamic pcap](https://www.mist.com/documentation/dynamic-packet-captures/), [trigger list](https://www.juniper.net/documentation/us/en/software/mist/mist-wireless/shared-content/topics/topic-map/packet-captures.html)).
- **Meraki Proactive PCAP** — fires on connection failure **or roaming issue**; rate-limited to the first failure per 5-minute interval; 7-day retention ([Meraki pcap](https://documentation.meraki.com/Platform_Management/Dashboard_Administration/Troubleshooting_and_Support/Troubleshooting/Packet_Capture_Overview)).
- **Cisco Intelligent Capture (iCAP) Anomaly Capture** — always-on per-AP capture of management frames for clients with onboarding issues, on a direct AP→Catalyst-Center gRPC channel (TCP 32626) bypassing the WLC; repeated identical anomalies are **aggregated at the AP into one event before sending**; hard caps: 100 packets per onboarding event, 1 GB rolling full-capture, 14-day retention ([iCAP](https://www.cisco.com/c/en/us/td/docs/cloud-systems-management/network-automation-and-management/catalyst-center-assurance/2-3-7/b_cisco_catalyst_assurance_2_3_7_ug/b_cisco_catalyst_assurance_2_3_6_ug_chapter_01110.html)).

Aruba's AI Assist auto-collects logs+pcaps on infrastructure events (port/tunnel flaps) and pre-populates TAC cases; UXI hardware sensors run automated follow-up triage tests when a synthetic test fails ([UXI](https://help.capenetworks.com/en/articles/11504575-how-uxi-helps-identify-and-troubleshoot-network-and-application-issues)). Extreme and Nile document no automatic capture — deep collection is manual/on-demand.

The rate limits, dedup, and byte caps above are **hardcoded cost-control policies**: evidence that vendors treat deep sensing as budgeted, but the budget logic is static (first-failure-per-5-min, fixed packet caps), not a learned or information-driven policy.

## 5. Sensor vs. infrastructure vs. client-agent trade-offs

- Dedicated sensors (7SIGNAL Sapphire Eye, Wyebot, UXI, Nile wall-plug) run true end-to-end tests from a fixed vantage point — association, 4-way handshake, DHCP, DNS, HTTP, VoIP MOS — on fixed cadences (UXI ~3 min/network; Nile every minute backing a financially-guaranteed SLA, [Nile SLAs](https://docs.nilesecure.com/nile-slas)). They consume real airtime and an association slot; **no vendor publishes a quantified airtime-overhead figure for its own active tests**. The closest admissions: 7SIGNAL markets its sensor's *Ethernet* backhaul as "zero impact to the Wi-Fi environment" ([7SIGNAL](https://7signal.com/platform/sensors/)), and Cape/UXI concedes its sensor's own capture buffer depresses its measured throughput ([UXI FAQ](https://help.capenetworks.com/en/articles/3295755-faq-s)).
- Wyebot's answer to measurement interference is hardware: 3 radios — one active-testing as a client, two purely passive capturing 24/7 with 24-hour pcap retention on the sensor itself ([Wyebot](https://wyebot.com/blogs/complete-packet-capture-support/)). Radio separation removes self-corruption of the measurement but not contention with production traffic on the tested channel.
- Sensors acting "as a client" reveal only how *the sensor's chipset* would roam. Only software on the real device (Marvis Client, Zebra Wireless Insights, 7SIGNAL Mobile Eye, ThousandEyes Endpoint Agent) sees genuine roam decisions — and this category is mostly **passive** (reads OS/driver state, near-zero airtime; Marvis Client quantifies cost as ~2–3%/hr battery, [FAQ](https://www.juniper.net/documentation/us/en/software/mist/mist-aiops/topics/concept/marvis-client-faq.html)). ThousandEyes documents its endpoint collection precisely: continuous WiFi state (RSSI, BSSID, retransmission rate, roaming events) plus per-minute ICMP/TCP probes to gateway/proxy/VPN, all over HTTPS 443 ([ThousandEyes](https://docs.thousandeyes.com/product-documentation/global-vantage-points/endpoint-agents/how-endpoint-agents-work/how-does-the-endpoint-agent-work.md)).
- Client agents remain rare in enterprise WLAN — Mist is the only WLAN vendor with a first-class client SDK; everyone else infers client experience from the infrastructure side, or borrows the client's radio via 802.11k beacon reports.

## 6. Relevance to the edge-diagnostic-agent research direction

Mapping this survey onto [the research direction](edge-diagnostic-agent-research-direction.md):

- **The two-tier structure already exists** (§4) — always-on cheap telemetry + gated deep capture is standard practice. What does not exist anywhere in the survey is a *policy* deciding escalation: triggers are hardcoded event lists (Mist's failure enum, Meraki's 5-min dedup). This is direct evidence for the gap claimed in §9 of the direction: "commercial systems do not treat monitoring cost as a decision variable at all."
- **Retention windows quantify RQ1's irreversible information loss**: Mist pcaps 7 days / telemetry 60-day rolling; Meraki client RSSI **2 hours**, proactive pcaps 7 days; Cisco anomaly pcaps 1.05 GB FIFO. A post-hoc query outside these windows is unanswerable by construction — the industry has already committed to retention policies, just not learned ones.
- **The cost model's structure (direction §5) is corroborated by hardware**: vendors physically separate sensing from serving (scanning radios, Wyebot's radio split, 7SIGNAL's Ethernet backhaul) precisely because passive monitoring on a serving radio and on a dedicated radio have categorically different costs — the capability-on-hardware pairing argument.
- **Nobody publishes the airtime cost of their own active testing** (§5). The direction's non-participating-observer requirement (§6) and its inflicted-harm axis have no commercial precedent to reuse; that measurement has to be built.
- **802.11k beacon reports are the in-protocol version of RQ3's dilemma**: the AP spends client airtime (an on-air scan) to buy client-perspective evidence — a per-probe cost/information trade the standards expose but no surveyed system optimizes. EasyMesh's Metric Reporting Policy TLV (interval + RCPI/utilization thresholds) is the closest standardized knob an escalation policy could actuate.
- **Precedents for edge-side data reduction** (Cisco's AP-side anomaly aggregation, uCentral's auto-stopping telemetry stream, USP's subscription model) are the mechanism layer the direction's §5 explicitly defers — they exist off the shelf; the policy that drives them is the open ground.

## 7. Flagged as unverified

Claims the research passes could not confirm from primary sources: OpenSync default MQTT report intervals (OVSDB schema not inspected); exact EasyMesh TLV layouts (from prplMesh docs, not the WFA base spec); 7SIGNAL test-scheduling intervals and Sapphire Eye upload protocol; NetBeez upload protocol/cadence (specs PDF unreadable); Marvis Client upload cadence; Mist AP↔cloud internal wire protocol; any vendor's per-AP telemetry byte volume except Meraki's ~1 kbps; Nile's "1000+ data points" (marketing, no primary source); telemetry buffering during WAN outage for all vendors.

## Related pages

- [Home WiFi Roaming Scenario](home-wifi-roaming-scenario.md) — the testbed whose telemetry table (hostapd events, station dumps, monitor capture) is the DIY analog of mechanisms 1–3 here.
- [802.11k — Radio Resource Measurement](80211k-radio-resource-measurement.md) — the beacon-report primitive that turns clients into sensors.
- [802.11v — Wireless Network Management](80211v-wireless-network-management.md) — BTM as the steering actuator vendors wrap policies around.
- [Common WiFi Roaming Issues](2026-06-08-wifi-roaming-common-issues.md) — the fault classes this telemetry must discriminate.
