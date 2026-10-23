Flow Chart:
``` mermaid
flowchart TB
  DUT["Device Under Test"]
  NODE["Node"]
  BACKEND["Backend"]
  FB["Fluent Bit"]
  ES["Elasticsearch"]

  DUT ---  NODE
  NODE --> |"HTTP — node_id, dut_id, status"| BACKEND
  BACKEND --> |"REST — config + session_id (+ actions)"| NODE
  NODE --> |"HTTP — events (stamped session_id/dut_id/node_id)"| FB
  FB --> |"HTTP — events"| BACKEND
  FB --> |"HTTP — events"| ES
  BACKEND <--> |"HTTP — queries / results"| ES
```

Sequence Diagram:

``` mermaid
sequenceDiagram
    participant DUT as Device Under Test
    participant NODE as Node
    participant BACKEND as Backend
    participant FB as Fluent Bit
    participant ES as Elasticsearch

    Note over NODE,ES: Setup
    NODE->>BACKEND: HTTP - node_id, dut_id, status
    alt no active session for dut_id
        BACKEND->>ES: HTTP - create session doc, session_id + dut_id, status setup
    else session already active
        BACKEND->>ES: HTTP - get active session_id for dut_id
    end
    BACKEND->>ES: HTTP - create node doc, session_id + node_id, status online
    Note over NODE: online
    BACKEND->>NODE: REST - config + session_id
    NODE->>BACKEND: HTTP - status ready
    BACKEND->>ES: HTTP - update node doc, config + status ready
    Note over NODE: ready

    Note over NODE,ES: Recording
    BACKEND->>NODE: REST - start
    NODE->>BACKEND: HTTP - status running
    BACKEND->>ES: HTTP - update session, status recording + started_at
    Note over NODE: running
    loop while recording
        DUT->>NODE: network/bus traffic
        NODE->>FB: HTTP - events stamped session_id, dut_id, node_id
        FB->>BACKEND: HTTP - events
        FB->>ES: HTTP - events
        opt actions may change mid-session
            BACKEND->>ES: HTTP - upsert action doc
            BACKEND->>NODE: REST - actions
        end
    end

    Note over NODE,ES: Stop and offline analysis
    BACKEND->>NODE: REST - stop
    NODE->>BACKEND: HTTP - status stopped
    BACKEND->>ES: HTTP - count events by session_id
    ES-->>BACKEND: HTTP - count
    BACKEND->>ES: HTTP - update session, status ended + ended_at + event_count + performance
    BACKEND->>ES: HTTP - update node doc, status ready
    Note over NODE: ready
    BACKEND->>ES: HTTP - query events by session_id
    ES-->>BACKEND: HTTP - results
```

State Diagram (node states):

``` mermaid
---
config:
  theme: default
---
flowchart TB
    Online["<b>Online</b><br/>Connected, no role yet"]
    Ready["<b>Ready</b><br/>Role loaded, armed"]
    Running["<b>Running</b><br/>Performing its task"]
    Unreachable["<b>Unreachable</b><br/>Heartbeat timeout"]
    Online -->|Configured| Ready
    Ready -->|Session started| Running
    Running -->|Session stopped| Ready
    Online -.-> Unreachable
    Ready -.-> Unreachable
    Running -.-> Unreachable
    Unreachable -->|Not configured| Online
    Unreachable -->|Configured, session off| Ready
    Unreachable -->|Configured, session on| Running
```

Container Diagram:

``` mermaid
flowchart TB
    analyst(["Analyst"])
    dut["DUT<br/>Device Under Test"]

    subgraph PI["Raspberry Pi Node (x N)"]
        node["Node <br/>Capture + action"]
    end

    subgraph PC["Core"]
        fe["Frontend"]
        api["Backend"]
        eng["Analysis Engines"]
        fb["Fluent Bit"]
        es[("Elasticsearch ")]
    end

    analyst -- "Uses — HTTP" --> fe
    fe -- "Requests: sessions, nodes, actions, search, run engine — REST" --> api
    api -- "Live events — SSE push" --> fe
    api -- "Query results, engine results, session and node data — REST responses" --> fe

    dut -- "Network/bus traffic" --> node
    node -- "Actions" --> dut
    node -- "Register, status acks: node_id, dut_id, status — HTTP" --> api
    api -- "config + session_id, start, stop, actions — REST" --> node

    node -- "Events stamped session_id, dut_id, node_id — HTTP" --> fb
    fb -- "Events, live — HTTP" --> api
    fb -- "Events into wiresploit-event — HTTP" --> es

    api -- "Session, node and action docs, search queries — HTTP" --> es
    es -- "Query results — HTTP" --> api
    api -- "Run engines after stop, passes session events — in-process" --> eng
    eng -- "Findings — in-process" --> api
```

Component Diagram:
Back-end Components:    
``` mermaid
---
config:
  theme: base
---
flowchart TB
    fe["Frontend"]
    node["Node (Raspberry Pi)"]
    fb["Fluent Bit"]
    plg["Plugin Sandbox<br/>(isolated container per run)"]

    subgraph API["Backend"]
        direction TB
        subgraph P["Presentation layer"]
            rSession["Session Router"]
            rNode["Node Router"]
            rAction["Action Router"]
            rSearch["Search Router"]
            rEngine["Engine Router"]
            rLive["Live Feed Router<br/>POST /ingest · GET /stream"]
        end
        subgraph B["Business layer"]
            sSession["Session Service"]
            sNode["Node Service<br/>incl. liveness check"]
            sAction["Action Service"]
            sEngine["Engine Service<br/>installed plugins, runs, limits"]
            cache["Result Cache<br/>in-memory, per session"]
            sLive["Live Feed Service"]
        end
        subgraph PS["Persistence layer"]
            repoSession["Session Repository"]
            repoNode["Node Repository"]
            repoAction["Action Repository"]
            repoEvent["Event Repository"]
        end
        subgraph I["Integration"]
            client["Node Client"]
        end
    end

    subgraph DB["Database layer"]
        es[("Elasticsearch")]
    end

    fe -- "REST + SSE" --> P
    node -- "register, status acks, heartbeat — HTTP" --> rNode
    fb -- "events, live — HTTP POST /ingest" --> rLive
    fb -- "store events into wiresploit-event — HTTP (bypasses Backend)" --> es

    rSession --> sSession
    rNode --> sNode
    rAction --> sAction
    rSearch -- "open layer: events of a session — browse or search, read-only" --> repoEvent
    rEngine -- "list / run / results" --> sEngine
    rLive -- "put events / read events" --> sLive

    sSession -- "start / stop all nodes" --> sNode
    sSession -- "run engines after stop" --> sEngine
    sSession --> repoSession
    sSession -- "count events" --> repoEvent
    sNode --> repoNode
    sNode --> client
    sAction --> repoAction
    sAction -- "push actions" --> sNode
    sEngine -- "get / put results" --> cache
    sEngine -- "events of the session (cache miss)" --> repoEvent
    sEngine -- "events in: stdin · findings out: stdout — JSON lines" --> plg

    client -- "config + session_id, start, stop, actions — REST" --> node
    repoSession -- "HTTP" --> es
    repoNode -- "HTTP" --> es
    repoAction -- "HTTP" --> es
    repoEvent -- "search, count, read — HTTP" --> es
```
Front-end Components:
``` mermaid
---
config:
  theme: base
---
flowchart TB
    analyst(["Analyst"])

    subgraph FE["Frontend"]
        direction TB
        subgraph V["Views"]
            vLive["Live Timeline View"]
            vSessions["Sessions View"]
            vEngines["Analysis Engines View"]
            vActions["Actions View"]
            vNodes["Nodes Panel"]
        end
        subgraph UI["Shared UI components"]
            cTable["Event Table"]
            cDetails["Event Details (PSF)"]
            cPayload["Payload View (hex / ASCII)"]
        end
        subgraph ST["State"]
            stSession["Session Store"]
            stNode["Node Store"]
            stAction["Action Store"]
            stLive["Live Event Store"]
            stSearch["Search Store"]
            stEngine["Engine Store"]
        end
        subgraph AC["API Client"]
            aSession["Session API"]
            aNode["Node API"]
            aAction["Action API"]
            aSearch["Search API"]
            aEngine["Engine API"]
            aLive["Live Feed Client"]
        end
    end

    subgraph BE["Backend — Presentation layer"]
        rSession["Session Router"]
        rNode["Node Router"]
        rAction["Action Router"]
        rSearch["Search Router"]
        rEngine["Engine Router"]
        rLive["Live Feed Router"]
    end

    analyst -- "Uses" --> V

    vLive --> cTable
    vLive --> cDetails
    vLive --> cPayload
    vSessions --> cTable
    vSessions --> cDetails
    vSessions --> cPayload

    vLive -- "live events" --> stLive
    vLive -- "search the live session" --> stSearch
    vSessions -- "session list, status, counts" --> stSession
    vSessions -- "events of a session: browse or search" --> stSearch
    vEngines -- "list / run / results" --> stEngine
    vActions --> stAction
    vNodes --> stNode

    stSession --> aSession
    stNode --> aNode
    stAction --> aAction
    stSearch --> aSearch
    stEngine --> aEngine
    stLive -- "open stream" --> aLive

    aSession -- "REST" --> rSession
    aNode -- "REST" --> rNode
    aAction -- "REST" --> rAction
    aSearch -- "REST: session_id + optional text" --> rSearch
    aEngine -- "REST" --> rEngine
    aLive -- "SSE GET /stream" --> rLive

```
  
  plugin Components:
``` mermaid
---
config:
  theme: base
---
flowchart TB
    sEngine["Engine Service<br/>(Backend — Business layer)"]

    subgraph SB["Plugin Sandbox — one plugin package"]
        direction TB
        manifest["Plugin Manifest<br/>name, version, question it answers,<br/>event types / protocols wanted, limits"]

        subgraph RUN["Running container — no network · read-only filesystem · no credentials · CPU / memory / time limits"]
            direction TB
            subgraph SDK["Plugin SDK — written once, shared by every plugin"]
                runner["Runner (entry point)"]
                reader["Event Reader<br/>stdin JSON lines → Event objects"]
                contract["AnalysisEngine contract<br/>analyze(events) → findings"]
                writer["Finding Writer<br/>validates the Finding shape → stdout"]
            end
            logic["Analysis Logic<br/>plugin-specific — the only part a plugin author writes"]
        end
    end

    sEngine -- "reads before the run: which plugin, what data, what limits" --> manifest
    sEngine -- "start the run (container per run)" --> runner
    sEngine -- "events in — stdin, JSON lines" --> reader

    runner -- "1. read events" --> reader
    runner -- "2. call analyze(events)" --> contract
    runner -- "3. write findings" --> writer
    logic -- "implements" --> contract

    writer -. "findings out — stdout, JSON lines (return, not a call)" .-> sEngine
```