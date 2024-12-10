# 2024-12-09 Record Decisions

## Context

There are several projects in the Reich/CASE/ASTER lab, many of which are
interconnected. Discussions for new features in these projects take place
across the GitHub repositories in both the
[reichlab](https://github.com/reichlab) and
[hubverse-org](https://github.com/hubverse-org) organizations. The current
setup sometimes leads to challenges in communictating across the landscape of
projects.

1. decisions from discussions with subsets of individuals are sometimes not
   communicated clearly to a wider group
2. discussion threads can become lengthy and drawn out with conclusions
   difficult to find (see for example [the discussion of the "target data"
   format](https://github.com/orgs/hubverse-org/discussions/9)) or they may be
   spread around issues in separate repositories
3. new lab members may not be able to glean the full context for the motivation
   of past decisions and may not be able to ascertain who is responsible for
   individual projects without asking others in the lab

Ultimately this leads to a fragmented project landscape. A decision made
upstream could negatively impact a downstream project maintained by another
member of the lab with little notice. Moreover, without a clear record of
decisions, complex projects rely heavily on the [core developer's
fully-connected mental
model](https://carpentries.github.io/instructor-training/instructor/aio.html#building-a-mental-model),
which drastically reduces the goat factor (the non-zero chance that the
developer will leave the project to raise goats).

Having a centralized space for recording decisions and discussions will provide
a path towards alleviating the above concerns and consequences.

### Aims

 - create a central space for formal decisions
 - establish a standard document to record decisions
 - establish a standard document to record project details
 - establish a formal process to record decisions
 - create a central space to record artifacts associated with decisions
 - establish a formal process to propose new ideas

### Anti-Aims

 - perscribe a particular mode or method by which discussions happen

## Decision

### Central space for formal decisions

We will create the reichlab/decisions to as a central space to record decisions
related to reichlab projects including hubverse related projects. It will have
two folders: `decisions/` and `project-posters/` with the following structure:

```
reichlab/decisions
├─ README.md
├─ decisions/
│  ├─ YYYY-MM-DD-ldr-cladetime-nextclade-integration.md
│  ├─ YYYY-MM-DD-ldr-cladetime-nextclade-integration/
│  │  └─ clade-diagram.png
│  └─ YYYY-MM-DD-rfc-reichlab-name.md
└─ project-posters/
   ├─ cladetime/
   │  └─ cladetime-poster.md
   └─ reichlab-rebrand/
      └─ reichlab-rebrand.md
```

The `decisions/` folder contains [Architecture Decision
Records](#standard-document-to-record-decisions) and the `project-posters/`
folder contains [Project Posters](#standard-document-to-record-project-details)
for high-level overviews of project details. Each decision may be associated
with any number of projects and vice versa.

### Standard document to record decisions

We will use Architecture Decision Records, as described by Michael
Nygard in [documenting architecture
decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)
to establish a standard document to record decisions with the following modifications:

1. decisions will be dated using ISO 8601 format, not numbered.
2. decisions will be in two categories: 
   1. `rfc`: request for comments, which requires feedback before implementation.
   2. `ldr`: lightweight decision record, which is low-stakes and does not require feedback.
3. Projects that are affected will be recorded in a list at the end of the document

### Standard document to record project details

We will use a [project
poster](https://www.atlassian.com/software/confluence/templates/project-poster)
for each project with associated artifcats. Each poster describes what the
project is, who the project owners are, why we are doing it, and who it is for
along with any associated artifacts (template forthcoming).

### Formal process to record decisions

We will use GitHub Pull Requests as a formal method to record decisions.
Decisions that supersede old decisions will cause the old decision's status to
be "superseded."

Once a decision is recorded as accepted, only its status may change.

### Central space for recording artifacts associated with decisions

Artifacts associated with decisions (e.g. rendered HTML content, markdown,
figures, data, etc) will be stored in a folder with the same name as the ADR.

```
├─ decisions/
│  ├─ YYYY-MM-DD-ldr-cladetime-nextclade-integration.md
│  ├─ YYYY-MM-DD-ldr-cladetime-nextclade-integration/
│  │  └─ clade-diagram.png
```

### Formal process for discussion of new ideas

Each new idea should be opened as a GitHub issue, discussion, or as a pull
request with a proposal for an idea. Each idea will eventually lead to an ADR
proposal and should be open for discussion of individual details. Individuals
should coordinate and determine the best method they would like to use to
record discussions.

## Status

Proposed

## Consequences

Consequences as described by Nygard in [documenting architecture
decisions](https://cognitect.com/blog/2011/11/15/documenting-architecture-decisions)

## Projects

 - None

