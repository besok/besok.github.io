+++
title = 'Proprioception for the Smart Factory: A Reactive Graph for Industrial Documentation'
date = 2026-08-20T00:00:00+01:00
draft = false
tags = ['smart-factory', 'manufacturing', 'documentation', 'graph', 'reasoning']
+++

# Proprioception for the Smart Factory: A Reactive Graph for Industrial Documentation

## Intro

In biology and robotics, [proprioception](https://en.wikipedia.org/wiki/Proprioception) is the intrinsic sensing mechanism that allows a system to continuously track its own state, position, and physical boundaries.

It relies on a distributed network of [mechanoreceptors](https://en.wikipedia.org/wiki/Mechanoreceptor) or, in machines, encoders and telemetry, to monitor attributes like joint angles, load tension, and spatial limits, enabling strict coordination and immediate error detection.

In this article, I want to reason about what is, in my opinion, the most vital part of the factory: documentation. The article places more open questions rather brings answers to them. The things in the articles are rather obvious but intricate. I believe it is not novel and a lot of companies successfully solved it internally but I have not met the industrial solution yet.

## The main problem

The core problem is simple. A typical smart factory relies on a multi-tiered stack of hardware and software systems:

- [ERP](https://en.wikipedia.org/wiki/Enterprise_resource_planning) (Enterprise Resource Planning)
- [MES](https://en.wikipedia.org/wiki/Manufacturing_execution_system) (Manufacturing Execution System)
- Shopfloor & Edge Systems ([PLCs](https://en.wikipedia.org/wiki/Programmable_logic_controller), [Robotics](https://en.wikipedia.org/wiki/Industrial_robot), [SCADA](https://en.wikipedia.org/wiki/SCADA), [HMIs](https://en.wikipedia.org/wiki/User_interface))

Every system in this stack comes with a documentation layer, a set of technical requirements it must fulfill.

Conversely, whenever we update a system, we end up pushing new requirements or tweaking existing ones.

In practice, these documentation layers don't exist in isolation; they are tightly interconnected. You can think of the entire plant as a massive graph, mapping high-level business logic down to leaf nodes: the physical hardware and running software. A change to a single requirement at the top can cascade into a series of mandatory changes all the way down.

When factories were "dumb," changes happened slowly. You could manage this graph manually, updating code, reconfiguring machines, and tweaking docs by hand and still keep everything mostly in sync.

Smart factories break this balance. As software takes over operational logic, teams naturally adopt [Continuous Delivery](https://en.wikipedia.org/wiki/Continuous_delivery), shipping rapid, incremental changes to production. At the same time, the software stack grows far more complex. Managing the dependency graph between specifications and runtime code by hand becomes impossible.

Which brings us to the core engineering problem: How do we build a unified documentation layer that links software and hardware to their underlying requirements, while automatically enforcing changes in downstream dependencies whenever a source document changes?

In short: how do we build a central nervous system for factory documentation?

Do existing systems solve this already? Kind of, partially, at least on paper.

- **PLM / ALM Platforms:** Heavy enterprise platforms like Siemens Teamcenter/Polarion or PTC Windchill try to bridge this gap. But in practice, the process is incredibly tedious, and these systems rarely extend cleanly down to the shopfloor (PLCs, SCADA, edge controllers). I've frequently observed a completely neglected attitude toward keeping them linked to all parts of the real, running systems.
- **Asset Administration Shell (AAS):** In theory, AAS gives every physical asset and software module a standardized digital wrapper for specs and documentation. In practice, however, these submodels usually become digital dumping grounds for static PDFs and JSON files.
- **Infrastructure-as-Code (IaC) & GitOps:** In pure cloud software, IaC and GitOps solve a very similar problem: detecting and reconciling the drift between declared state and running state. But physical shopfloor hardware, with its hard real-time loops and safety constraints, doesn't fit neatly into a standard declarative GitOps model. Ladder diagrams, the proprietary parts of the code are hard to manage with git.

The core gap is how both current tooling and engineers themselves view documentation: as a **pre-facto design requirement or a post-facto audit artifact**. It is treated as something that exists on paper or in a wiki, but completely disappears at runtime.

## Why it is hard in practice:

- **Heterogeneity of the leaves.** A factory graph has CAD files, PLC ladder logic, SCADA tags, PDFs, safety certifications, other machinery-specific formats and systems, not to mention cloud-based or pure software systems on top of that. For every one of them, there needs to be an adapter.
- **Ownership and incentives.** Even if the tool exists, someone has to actually maintain the links, which means the engineering process itself needs an extra step: linking the system, not just building it. One important factor here is how lightweight the process needs to be. It should stay transparent, quick, and easy, nothing that adds friction to the engineer's actual work.
- **Standards exist but stop at the metadata layer.** Typically the process stops at the borders of a system, not bothering to go further.
- **The links to the leafs are semantic not syntactic.** We have to either enforce it through the process of creation during work with leafs or manually establish it.
- **Safety/compliance culture actively resists dynamism.** Anything touching safety bounds or certified configurations comes with a regulatory/audit trail that doesn't want to be "just another node in a graph." So full automation isn't even desirable everywhere.

## Design:

### Core

The core is a graph-based system. Nodes are pulled from multiple heterogeneous sources, like some human-authored, some machine-generated:

- Documents (specs, PDFs, certifications, manuals)
- Notes/Records (change logs, tickets, commit messages)
- Structured data pulled directly from industrial datastores or databases (ERP records, MES parameters, SCADA tags)

The open question which comes first is granularity: **do we decompose every document into homogeneous, atomic records, or do we keep documents intact and anchor sections in them?**

My choice is the latter. I see it as the unit of the graph should be a **reference, not a copy**. The source of truth stays the original document; the graph just gives that fragment an address and a place to hang edges off of.

This has a few consequences worth being explicit about:

- **It's cheap to start.** You're not blocked on digitizing a decade of PDFs before the system does anything useful, you tag as you touch things, which matches how engineers actually work: incrementally, driven by whatever they're currently changing.
- **It preserves context.** A decomposed "fact" loses the surrounding qualifiers, exceptions, and conditions that usually live in the same paragraph, but a reference keeps the fact anchored to its full context instead of asserting a clean, and possibly wrong, extraction of it.
- **The manual work required is not new work.** This is worth stressing, because it's the crux of the adoption problem: someone tagging `[qq245]` when they touch a requirement is not meaningfully more effort than what disciplined engineers are already supposed to be doing, namely referencing the spec they changed something against. The system doesn't ask for a new discipline, it asks for the existing discipline to leave a trace instead of living only in someone's head or a commit message nobody reads later.

**Should the core be historical?**

A related question worth naming explicitly: should the graph only reflect current state, or should it also carry history like what a node used to point to, what an edge used to be, before it changed?

My take: yes, eventually but this is a genuinely harder engineering problem than it first appears, and I'd deliberately defer it rather than try to solve it in v1. There's also a practical reason to sequence it this way: **live state is the part with the immediate payoff**, and it's the part that proves whether the whole approach is viable at all.

### Integration

If Core defines _what_ a node looks like, the Integration Layer defines _how_ it gets there. Every system in the stack needs its own adapter, because each speaks a different native format and none of them were designed with this graph in mind.

The adapter's job is narrow and mechanical: **scan the target system, find breadcrumbs, and report them back.**

A breadcrumb is just a marker like `[qq245]`. It can be dropped into a comment field, a tag description, a ladder logic rung annotation, wherever the target system allows free text. The adapter doesn't need to understand what `[qq245]` means; it just needs to recognize the pattern and extract it, along with enough context (file, location, timestamp, author if available) to make the reference addressable back in Core.

**It is relatively easy for the general software than the industrial systems.**

The Ladder logic, proprietary PLC project files, CAD binaries, vendor-locked SCADA configs or any other closed formats, sometimes without any documented way to embed free text at all, and sometimes only readable through a vendor's own tooling. For those, "find the breadcrumb" isn't a parsing problem, it's an _access_ problem, meaning you first have to figure out whether the format even has a place to put a marker, and whether you can extract it without the vendor's SDK.

This means the Integration layer isn't purely a technical layer, it's a **process question as much as a tooling question**. Leaf types need to be classified upfront by how tractable they are, roughly:

- **Native text** (configs, markdown, comments, tickets): breadcrumb + adapter is close to free.
- **Structured but closed** (PLC project files, SCADA tag databases): often has _some_ text field to hijack for a breadcrumb, but requires a purpose-built adapter per vendor/format, and sometimes per version.
- **Opaque/binary** (CAD geometry, compiled ladder logic, firmware blobs): may have no usable text field at all. Here the breadcrumb often can't live _inside_ the artifact; it has to live _next to_ it like a sidecar file, a filename convention, an entry in whatever PLM/version-control wrapper the file already sits in.

For each class, you need both **an adapter _and_ a process adaptation** like a convention the engineers working in that system agree to follow, since the tooling alone can't manufacture a place to put a marker that the format doesn't support.

Several observations:

- **The system must be alive, not a snapshot. It should scan the changes(or rescan everything).**
- **The process must stay lightweight.**
- **Manual revision is required to eliminate false positives.**

### Meta

Core and the Integration, taken together, give you a set of relatively small, isolated subgraphs, loosely centered around each system.

The open problem at this layer is different from anything Core or Integration had to solve: **how do we connect these subgraphs to each other?**

A requirement in the Robotic Cell graph and a parameter in the MES graph might genuinely describe the same constraint, but nothing in either system's own breadcrumb trail says so because the link lives _between_ systems, in someone's head, not inside either system's native format.

This is, at least initially, a manual process. Someone who understands both the MES-side requirement and the machinery-side parameter has to assert that they're the same thing, or that one constrains the other. Unlike the breadcrumbs in Core, which get left almost incidentally as a byproduct of normal engineering work, these cross-system bonds require someone to deliberately step outside their own system and reach into another one. That's a fundamentally different kind of effort, and it's worth naming honestly.

**Can LLMs help here?**

Possibly, and arguably more plausibly. At the Meta layer, we are matching between subgraphs that are each already somewhat structured and named, which is a more tractable input for similarity matching or embedding-based suggestion than raw proprietary binaries.

But the same principle from before still applies without exception: any artificially-suggested cross-system bond is a candidate, not a fact.

**The payoff: these bonds are long-living and slowly-changing.**

This is what makes the manual cost at this layer worth paying, and it's a meaningfully different cost/benefit shape than Core's breadcrumbs. A breadcrumb inside a single system changes often. A cross-system bond describes a structural relationship between two systems' _concepts_, not their current values, and that kind of relationship is far more stable.

### Query

Everything built so far exists to answer one question well: _if I change this, what else is affected?_ The Query Engine is where that question actually gets asked, and it's arguably the layer worth investing in first, ahead of the more speculative parts of the stack.

**The core operations is traversal not lookup.**

A conventional search returns documents that match a query. This needs something closer to `rdeps()` in a build system, or forward-citation search in an academic database: given a node then return everything reachable from it, at whatever depth, across however many system boundaries it happens to cross. The output isn't "the document that mentions X," it's "every node, in every system, that this change could ripple into."

**Why invest here first, practically.**

A minimal version of this doesn't need Meta cross-system ML bonding to already be mature, or Core's historical layer to exist at all. It can run against whatever fragment of the graph exists today, however incomplete, and still be useful for the subset of dependencies that _are_ captured. That's valuable for a second reason beyond adoption: a query tool that visibly returns incomplete or thin results is itself a forcing function. It makes gaps in the graph visible and legible to the people who could close them, rather than the gaps sitting silently unnoticed in an unqueried, unused store. In effect, good tooling here doesn't just consume the graph.

## Final Thoughts

I started writing this out of frustration, trying to formulate what a real solution to this problem would even look like, rather than just naming the gap and moving on.

Having worked through it, I don't think it's easy. But I do think it's doable, and the payoff, if it works, is substantial.

The main pitfall isn't technical (well technical challenge here is big as well, I have to admit that) but the balance. It dictates the nature of the system. It is not directly a product but rather a thing operating in background and it is not clear how and when it gives fruits.

Every design choice in this piece has a failure mode on either side of it, and both sides are real:

- Push too far toward completeness, and the system becomes heavier than the problem it was meant to solve.
- Push too far toward lightness, and it becomes a toy: a nice-looking graph visualization with a handful of hand-picked demo links, technically impressive, operationally irrelevant.

Whether a factory has proprioception or not isn't really a question about graph databases or ML matching. It's a question about whether that balance can be sustained by the people actually doing the work, day after day, long after the initial motivation for building it has faded. That's the part I don't have a clean answer to yet and probably the part worth returning to.

_Disclaimer: styling and error handling throughout this article were cleaned up with the help of AI._
