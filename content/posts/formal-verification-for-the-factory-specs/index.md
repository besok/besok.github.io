+++
title = 'Formal Verification for the Factory Specs'
date = 2026-11-08T00:00:00+01:00
draft = false
tags = ['formal-verification', 'automation', 'robotics', 'plc', 'dsl', 'specifications']
+++

# Formal Verification for the Factory Specs

**Summary.** Technical specifications for robot-cell functions live in text documents and are re-implemented by hand in the design data, the MES, the PLC, and the robot code. This causes ambiguous readings, uncovered edge cases, contradicting rules, and out-of-sync limits. I propose writing the rules once in a typed, executable rule language, analyzing them with a solver, evaluating or generating them in each system, and checking the rule version at runtime. No existing tool covers the whole chain, so the approach needs a custom framework, and a proof of concept (PoC) has to validate it first.


I keep returning to one problem in industrial automation: keeping the technical specifications shared by different systems consistent. After some time being in the industry, I believe this is the main composite glue allowing the systems functioning stable and robust.


## Context

A factory line with a set of robotic cells and other machinery (grippers, mount tables, flip gears, etc.). Every cell has up to 5 robot arms. The line is intended to create some product step by step. Every robot arm has a set of functions that it can execute. Every function is a high-level action that composes several simple actions. Examples:

- Place a board
- Place a beam
- Inject a staple line
- Blow insulation

Every function has a contract:

- Set of incoming parameters (size of the board, where to place, material, etc.)
- Set of constraints and rules (size of the board, where to place it, etc.)

The workflow has four steps:

1. Design data. We need to create a model of what we want to expect at the end.
2. Bill of materials (BOM) and bill of process (BOP) data. We need to match the existing materials and break the design data into the process of creation, considering the machinery.
3. We need to analyze the BOM/BOP data and transform it into the set of functions to perform a step in the BOP.
4. We need to deliver the functions to the machinery state with a programmable logic controller (PLC) or robot language and execute them.

Almost every step can run in a different system, because each step has different users:

- Design data is crafted in the specialized CAD/Modeling system. It needs to comply with the external constraints, rules or other expectations. Here, the design engineers or architect own the area.
- The BOM and BOP are created in an MES (manufacturing execution system) or similar. This area is owned by process and industrial engineers.
- The model that creates the functions can be a part of the MES but also can be an independent system. Nevertheless, the software engineers are responsible for the given module.
- The functions are delivered to the cell, where the PLC or robot controller picks them up. The robotics engineers are working with it.

The format of the contract is mostly textual (for instance, a PDF or Markdown document). It is stored and used by engineers during the implementation as a reference. No special validations are imposed over the document. All updates are delivered by hands.


## Issues

### Reading ambiguity of the rules

The unrestricted format leaves some room for free interpretation. For example: `The order of placing battens. The battens should be sorted by x-axis descending and the vertical batten should be placed first.` Does it mean, the priority the orientation or x position or in terms of the equal x-position only? Should we place the vertical batten first even if we have a horizontal batten with a smaller x position??

### Non-exhaustive rules

The rule can cover the cases partially and some edge cases leaves for the implementing systems. For example: `The collisions while stapling battens. The parameter CollisionPresence must be True when an obstacle lies in front of or beside the staple position within a 20 cm radius.` The issue here is the special case of collision behind (the batten) can arise for the first staple when the staple is closer to the edge than the given limit and the batten has another batten abutting it. The logical meaning of the rule here is when we staple the tool grips the batten and goes lower the level of the batten to staple. With the special case we risk damaging either another batten or the gripper itself.

### Logical inconsistency for the rule set

Rules can contradict each other, because nothing formally validates them. Sometimes it can be harmless, when the rules just make part of them redundant as in the given example:

```
- The beam width should be no more than 80 mm and no less than 60 mm
- Only 5 beams can be combined into a group of beams
- The accumulated beam group should be no more than 240 mm and no less than 180 mm
```

Here the second rule is redundant. The max width given can include max 4 beams. We can omit this rule harmlessly. The pitfall can arise when we decide to revise and update the third rule and extend the max limit to higher than `400 mm`. Then the third limit becomes useless as we are restricted by the second rule.

But at some cases it can be dramatic. Let's take a look into the following example:

```
- Every vertical batten should have the horizontal battens abut from top and bottom.
- The first vertical batten edge should be placed into the origin of the plane, meaning 0, over x,y pos.
- Every horizontal batten must be placed within the 10–10,000 mm range (for the nearest to smaller x edge.)
```

The system cannot run under these rules: together they are contradictory.

### The desynchronization update

If we update the limits we have to guarantee that all systems update all parameters accordingly. If one system still uses the old limit (1700 mm) and another the current one (1750 mm), the systems are out of sync and may behave inconsistently.


## Logical solution

### Process-wise description

We transform the rules into the formally verified DSL with a set of tools to compile, validate and generate the specs for the given layer. Also it allows us to generate a textual version of the specs. We revise and incorporate the systems with the generated module in a way that we keep the internal data structures and logic but use the generated module to test/validate the cases. Or we can use it in the runtime as well. When we need to update the limits, the framework should allow us to regenerate the logic.

### Issues addressed

- Reading ambiguity of the rules. Should be reduced dramatically, since we can compile it into executable code. Not to eliminate though.
- Non-exhaustive rules. Technically, with the general context we can impose the primitives and geometry behind the elements and introduce the meta rules as (no tools should cross any elements, etc.)
- Logical inconsistency for the rule set. Since every rule should operate over the generic context, we should be able to cross check the rules.
- The desynchronization update. It fully relies now on the process of updates. We should require the systems to follow the specific versions of the rules. Also we can impose the runtime checks

### Benefits

- Finds contradictions before implementation or commissioning.
- Produces concrete counterexamples instead of relying only on review discussions.
- Forces explicit definitions of ordering, units, boundaries, tolerance, and coordinate systems.
- Detects some redundant or ineffective rules.
- Prevents constants and validation logic from being copied manually between systems.
- Generates documentation, tests, and validators from the same semantics.
- Makes rule changes reviewable through semantic comparisons between versions.
- Improves traceability from a source requirement to rules, tests, and deployed versions.
- Can examine all states covered by the mathematical model, unlike example-based testing alone.

### Drawbacks

- Formalization is expensive. Domain experts must resolve ambiguities and approve semantics.
- Solver limitations: complex nonlinear, geometric, temporal, or unbounded problems can become slow or return unknown.
- Organizational overhead: engineers must learn the DSL and change their review/release process.
- Maintenance: source traceability, versions, compatibility, and migration become long-term product responsibilities.


## Technical solution

The DSL should be able to describe numerical and logical rules, define the context entities (boards, battens, tools) as typed items. The module should be able to generate the code for general-purpose programming languages (Java, Rust, Kotlin, Python) and controller ones (PLC, KRL \[KUKA Robot Language\], etc.)

A rule set could look like this (illustrative syntax):

```yaml
rule_set: beam_group
version: 1.4.0            # the hash of this file is reported by every system at runtime
units: mm                 # all lengths are integers in millimeters
default: reject           # anything not explicitly allowed is rejected

entities:
  Beam:  { width: int }
  Group: { beams: List<Beam>, width: "sum([b.width for b in beams])" }   # defined once, used by all systems

rules:
  - id: BEAM-WIDTH
    check: beam.width >= 60 and beam.width <= 80
    message: Beam width must be 60-80 mm
  - id: GROUP-SIZE
    check: len(group.beams) <= 5
    message: A group holds at most 5 beams
  - id: GROUP-WIDTH
    check: group.width >= 180 && group.width <= 240
    message: Beam group width must be 180-240 mm
```

The analysis step compares rule sets and reports, for example:

```text
INFO  GROUP-SIZE is redundant: BEAM-WIDTH and GROUP-WIDTH already limit a group to 4 beams.
WARN  If the GROUP-WIDTH upper limit is raised to 420 mm, it can never be reached:
      BEAM-WIDTH and GROUP-SIZE together cap a group at 400 mm.
```


Written down formally, the rules show where the clash comes from (illustrative syntax):

```yaml
rule_set: abutting_battens
version: 0.3.0
units: mm
axes: x right, y up            # the coordinate system is part of the rule set

entities:
  Batten: { orientation: "V|H", x_min: int, x_max: int, y_min: int, y_max: int }

input:
  Battens: List<Batten>

definitions:                   # the reading of "abut" is fixed once, here
  Verticals:   [b for b in Battens if b.orientation == "V"]
  Horizontals: [b for b in Battens if b.orientation == "H"]
  covers_width(h, v): h.x_min <= v.x_min and h.x_max >= v.x_max
  abuts_top(h, v):    h.y_min == v.y_max and covers_width(h, v)
  abuts_bottom(h, v): h.y_max == v.y_min and covers_width(h, v)
  first_vertical:     min(Verticals, key=lambda b: b.x_min)

rules:
  - id: ABUT
    check: |
      all(
        any(abuts_top(h, v) for h in Horizontals) and
        any(abuts_bottom(h, v) for h in Horizontals)
        for v in Verticals
      )
    message: Every vertical batten needs horizontal battens abutting from top and bottom
  - id: ORIGIN
    when: len(Verticals) > 0
    check: first_vertical.x_min == 0 and first_vertical.y_min == 0
    message: The first vertical batten starts at the origin
  - id: H-RANGE
    check: all(h.x_min >= 10 and h.x_min <= 10000 for h in Horizontals)
    message: Horizontal battens start between 10 and 10,000 mm
```

The analysis step reports a minimal contradictory set:

```text
ERROR  Rule set is unsatisfiable. Minimal contradictory set: ABUT, ORIGIN, H-RANGE
       ORIGIN   -> first_vertical.x_min = 0
       ABUT     -> a horizontal batten h must cover the vertical one, so h.x_min <= 0
       H-RANGE  -> h.x_min >= 10
INFO   Every pair of these rules is satisfiable:
       ABUT + ORIGIN     : h.x_min = 0 (or lower)
       ABUT + H-RANGE    : first vertical batten at x_min = 100, h.x_min = 50
       ORIGIN + H-RANGE  : no horizontal batten is required to abut
```


On one side it is always a custom framework. What can be on the other side? The options below are compared on four needs: writing rules once, analyzing them formally, evaluating them inside the target systems, and generating code for the PLC and robot side.


#### Simulink Design Verifier

[Simulink Design Verifier](https://www.mathworks.com/products/simulink-design-verifier.html) and [PLC Coder](https://se.mathworks.com/help/plccoder/index.html) give property proving, test generation, and PLC code generation. It is a mature option, mostly suitable for the PLC side. The approach is built on signals and state, while our rules are checks over static data, so it is hard to reach the upstream systems.


#### CATIA Magic

[CATIA Magic](https://www.3ds.com/products/catia/catia-magic) supports [SysML v2](https://en.wikipedia.org/wiki/Systems_modeling_language) and is a good place to keep the specifications. It has a REST API and flags requirements that violate a constraint. It evaluates constraints on a model, but we found no solver-based proof, no runtime for the target systems, and no PLC or KRL code generation.


#### Cedar

[Cedar](https://docs.cedarpolicy.com/) is an open-source language for authorization policies. It is the closest in nature: rules live outside the code, are analyzed with SMT, and are evaluated at runtime by a Rust engine used in AWS services. The gaps are expressiveness, which is heavily restricted for geometric rules, and the lack of generators for PLC and KRL.

#### FRET and PLCverif

NASA [FRET](https://github.com/NASA-SW-VnV/fret) formalizes requirements in a restricted natural language, checks them for consistency, and can export runtime monitors. CERN [PLCverif](https://cds.cern.ch/record/2777799) verifies PLC programs by model checking. Together they cane be combined. FRET reasons about signals over time, not static mission data, and its monitors do not target our upstream languages.  

#### Custom framework

None of the tools covers the whole chain. The custom part is the glue, and it can reuse their ideas: Cedar's separated rules with solver analysis, FRET's readable semantics, and generated tests from Simulink.

- Rule representation: [CEL](https://cel.dev/) expressions inside a typed schema (YAML/JSON), versioned in git.
- Evaluation: CEL in Python and Kotlin (via the Java implementation).
- Analysis: [Z3](https://www.microsoft.com/en-us/research/project/z3-3/) for contradiction, redundancy, completeness, change impact, and test vector generation.
- PLC: a template-based generator for Structured Text (ranges, counts, equality).
- KRL: generated constants and data only, the logic stays upstream and in the PLC.
- Versioning: every rule set carries a version hash that each system reports at runtime.


#### Comparison

| Option | Write rules once | Formal analysis | Runtime evaluation in target systems | PLC / KRL code generation |
| --- | --- | --- | --- | --- |
| Simulink Design Verifier + PLC Coder | Partly (rules become Simulink models) | Yes (property proving, design errors) | PLC only, upstream hard to reach | PLC yes, KRL no |
| CATIA Magic (SysML v2) | Yes | Partly (constraint evaluation, no solver proof found) | No  | None found |
| Cedar | Yes | Yes (SMT) | Yes (Rust engine) | No  |
| FRET + PLCverif | Yes (restricted natural language) | Yes (consistency, model checking) | Partly (monitors for signal properties) | No generation, PLC verification (mainly Siemens) |
| Custom framework | Yes | Yes (Z3) | Yes (CEL in Python and Kotlin) | PLC for simple rules, KRL constants only |


## Conclusion

The concept looks promising, but it needs a proof of concept (PoC) before committing to it. 

The PoC should cover one function family, for example placing battens and stapling, and show that the framework can:

- force an explicit definition of the batten order,
- flag the redundant rule once the group limit rises above 400 mm,
- detect that the three batten rules contradict each other,
- catch the collision-behind case through a safety rule,
- block a system that runs an outdated version of the rules.

What I like most about the concept is that it centers on the rules themselves, not on any single system. The same specification can serve the design data, the MES, the PLC, and the robot code, independent of how each of them is implemented.
