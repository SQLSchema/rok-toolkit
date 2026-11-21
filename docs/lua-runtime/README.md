<sub>[RoK Toolkit](../../README.md) › [Docs](../README.md) › The Lua Runtime Method</sub>

<img src="../assets/headers/lua-runtime.svg" width="100%" alt="">

# The Lua Runtime Method

Unlike [headless](../headless/README.md), a desktop tool attaches to the normally running client, reads its state, and invokes its code.

```mermaid
%%{init: {"theme":"base","themeVariables":{"primaryColor":"#000000","primaryTextColor":"#e8e8e8","primaryBorderColor":"#7a7a7a","secondaryColor":"#000000","tertiaryColor":"#000000","lineColor":"#7a7a7a","textColor":"#7a7a7a","titleColor":"#e8e8e8","edgeLabelBackground":"#000000","clusterBkg":"#000000","clusterBorder":"#7a7a7a","actorBkg":"#000000","actorBorder":"#7a7a7a","actorTextColor":"#e8e8e8","actorLineColor":"#7a7a7a","signalColor":"#7a7a7a","signalTextColor":"#7a7a7a","noteBkgColor":"#000000","noteBorderColor":"#7a7a7a","noteTextColor":"#e8e8e8","transitionColor":"#7a7a7a","transitionLabelColor":"#7a7a7a","stateLabelColor":"#e8e8e8","stateBkg":"#000000"},"flowchart":{"wrappingWidth":520}}}%%
%% source: "Unlike headless, a desktop tool attaches to the normally running client, reads its state, and invokes its code."
flowchart LR
    T["Desktop tool"] --> A["attaches to the normally running client"]
    T --> R["reads its state"]
    T --> I["invokes its code"]
```

| Layer | Page |
| --- | --- |
| Attachment | [Attach and the Execution Hook](01-attach.md): process discovery, elevation, attachment, single-thread rule |
| Reading | [Reading the Client](02-client-reads.md): read constraints, catalogue, raw table reads, install files |
| Acting | [Invoking the Client](03-client-calls.md): client functions, server view, message wrapping |
| Session | [Governors and Reloads](04-governors.md): in-client switching, reload hold |

## Boundary

Measured boundary: the tool does not log in, speak the protocol, or present client identity. It uses the running client’s process, code, and session.

---

<div align="center">

[← The RoK-Data Store](../headless/game-systems/07-rokdata-store.md) &nbsp;·&nbsp; [↑ Documentation Map](../README.md) &nbsp;·&nbsp; [Attach and the Execution Hook →](01-attach.md)

</div>
