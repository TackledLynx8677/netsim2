# NetSim — a Packet-Tracer-style network simulator (Flask)

A from-scratch network topology + packet simulator: build a topology by
dragging routers, switches, and PCs onto a canvas, cable them together,
configure each device through an IOS-flavored CLI, and watch a ping
actually get encapsulated, ARP-resolved, switched, and routed hop by hop
— animated live on the canvas.

## Quick start

```bash
pip install -r requirements.txt
python app.py
```

Then open **http://localhost:5050** in a browser. Click **Load demo
topology** for a ready-made two-site network (two LANs behind two routers
on a WAN link) that works out of the box, or start from an empty canvas.

No database, no build step, no JS framework — just Flask + vanilla JS.
The whole topology lives in server memory (`netsim/store.py`), so
restarting the server resets it (this is a single-session teaching tool,
like opening one `.pkt` file — not a multi-user service).

## What's actually simulated

This isn't a static diagram with a fake "packet" sliding along a line. A
ping is driven through a real, from-scratch simulation:

- **Encapsulation** — an ICMP Echo Request is built with real IP headers
  (src/dst, TTL) and, hop by hop, a real Ethernet frame (src/dst MAC,
  ethertype) is wrapped around it. You can inspect every header at every
  hop in the event log.
- **ARP** — if a device doesn't have a cached MAC for the next hop, it
  broadcasts an ARP request that actually **floods through switches** in
  its broadcast domain (routers and PCs are boundaries; switches are
  transparent), and the owning host **unicasts a reply back** along the
  learned path.
- **Switching** — every switch has a real CAM/MAC-address table. It
  learns the source MAC of every frame it sees on the ingress port, and
  either forwards out the single known port or floods, exactly like a
  real switch.
- **Routing** — routers keep a real routing table (`show ip route`):
  directly-**C**onnected routes appear automatically once an interface is
  addressed and `no shutdown`; you can add **S**tatic routes by hand; or
  turn on simplified **R**IP and hit "Converge RIP" to run distance-vector
  convergence (split-horizon, hop-count metric, max 15 hops) across every
  RIP-enabled router in the topology.
- **TTL** — each router hop decrements TTL and a packet that hits 0 dies
  with a simulated "TTL expired" event, same as a real traceroute would
  show.
- **The reply** — once the Echo Request is delivered, an Echo Reply is
  generated and driven back through the same machinery (which may now hit
  warm ARP caches, so the return trip is often shorter — also realistic).

Every step of all of this is recorded and streamed back to the browser as
an ordered list of events, which the frontend plays back as a sequential
animation (a glowing dot tracing each link) plus a scrolling, teletype
event log with full header detail — similar to Packet Tracer's Simulation
Mode.

## The CLI

Click a device, open its **Console** tab, and you get a simplified but
real IOS-style shell with the classic mode hierarchy and abbreviation
support (`en`, `conf t`, `int g0/0`, `sh ip rou`, `no shut`...):

```
Router> enable
Router# configure terminal
Router(config)# interface GigabitEthernet0/0
Router(config-if)# ip address 10.0.1.1 255.255.255.0
Router(config-if)# no shutdown
Router(config-if)# exit
Router(config)# ip route 10.0.2.0 255.255.255.0 192.168.100.2
Router(config)# router rip
Router(config-router)# network 10.0.1.0
Router(config-router)# end
Router# show ip route
Router# show ip interface brief
Router# show arp
Router# ping 10.0.2.10
```

Switches get `show mac-address-table`; PCs get a small simplified shell
(`ip address <ip> <mask> [gateway]`, `ipconfig`/`show ip`, `ping`,
`arp -a`) since real hosts don't run IOS.

## Architecture

```
app.py                  Flask routes (REST API only, no server-rendered state)
netsim/
  models.py              Topology graph: Device (Router/Switch/PC), Interface, Link, Route
                          + BFS helpers that flood ARP through switches and find
                          the point-to-point L2 path for unicast forwarding
  packet.py               Packet: per-layer headers (app/transport/network/datalink)
  engine.py                The simulation core: first_hop_decision (route-or-ARP
                            choice), resolve_arp (flood + learn + reply), _deliver
                            (hop-by-hop loop with TTL), simulate_echo (full ping)
  routing.py                Simplified RIP: neighbor discovery over the L2 graph +
                             Bellman-Ford-style distance-vector convergence
  cli.py                     The IOS-like command interpreter (mode state machine)
  store.py                   The single global Topology instance + the demo builder
static/
  css/style.css             Design system (schematic/console visual language)
  js/canvas.js               SVG topology renderer: drag/drop, link drawing, packet FX
  js/cli.js                    Terminal panel
  js/simulate.js                Ping panel: calls the engine, plays back the animation
  js/app.js                     API client + toolbar/tabs/device-detail wiring
templates/index.html          Page shell
```

## Notable simplifications (by design, for a teaching tool)

- Single global topology (no multi-user sessions, no persistence to disk —
  add `to_dict()`/`from_dict()` wiring if you want save/load).
- No VLANs, no STP (topologies are assumed loop-free — the switch BFS
  doesn't need loop-prevention because there's nothing to prevent).
- No IPv6, no TCP handshake modeling (ICMP echo only — TCP/UDP header
  fields exist in the Packet model but aren't driven through a state
  machine).
- RIP only (no OSPF) — RIP's distance-vector logic maps cleanly onto the
  L2-broadcast-domain BFS already built for ARP, which kept the "third
  routing protocol" reachable within scope; OSPF's link-state flooding
  and DR/BDR election would be a natural next step on the same graph.
- The `ip route` next-hop is recursively resolved against connected
  routes each time it's used rather than cached, so it always reflects
  the current interface state (matches real IOS behavior for
  next-hop-only static routes).
