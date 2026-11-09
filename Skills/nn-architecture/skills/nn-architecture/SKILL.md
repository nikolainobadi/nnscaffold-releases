---
name: nn-architecture
description: Universal architecture principles that hold for any Swift project, stated without reference to a kind of project. Covers the noun kinds used to classify and name types, how to choose names, how to decompose large units of work, and how to request an exception to a rule. USE WHEN naming a type, deciding what kind of type something is, splitting a large type or sequence of work, or when an architecture skill such as nn-cli-architecture routes here. Not for the rules of a specific kind of project: its own architecture skill owns those.
---

# nn-architecture

Rules that hold for any architecture, stated without reference to one. An architecture's own rules say how each principle applies there.

Every rule here takes judgment to apply. Only the catch-all rule is enforced by a tool, when a type is scaffolded.

## Noun kinds

Every type is exactly one kind. Check the kinds in this order and take the first match.

| Order | Kind | What it is | Test | Examples |
| --- | --- | --- | --- | --- |
| 1 | Structural | A term the architecture itself defines | Is it in the architecture's glossary? | Listed per architecture |
| 2 | Thing | Data and the rules about that data, with no injected collaborators | Would it still mean something outside the code that uses it? | `VersionTag`, `Invoice` |
| 3 | Role | Stands for an outside resource or boundary | Does it mainly front something outside the program? | `ConfigStore`, `GitClient` |
| 4 | Agent | Holds collaborators and has a job | Does it exist only to do a job? | `TagCreator`, `CdnConfigurator` |

When nothing fits, rethink the boundary first, then follow Exceptions.

The kind tests decide the noun. Agents are named for the knowledge they own, and a rule that needs only a type's own data lives on that type.

## Choosing a name

These apply to the names a project chooses for itself.

1. **Concept first.** Say in one sentence what the type knows or is, then name it from that sentence.
2. **The name covers everything the type does.** When a type gains a second action that belongs with it, rename the type rather than splitting it.
3. **Refusal test.** If you can't say what a type would refuse to do, the name is too vague.
4. **One meaning per word, one word per meaning,** across the project.
5. **Full words.** No abbreviations beyond ones everyone uses, such as `URL`.
6. **No catch-all words.** A name a project chooses never uses Manager, Helper, Handler, Processor, Service, Util, or Utility. Terms in an architecture's glossary are exempt. If nothing more specific fits, rethink the boundary first, then follow Exceptions.
7. **Glossary terms are reserved.** A glossary term means only what the glossary says. A project doesn't reuse it for anything else.

## Decomposition

**Group by knowledge, not by time.** Each module hides one body of knowledge: a design decision or outside system that could change on its own. Only the coordinator knows the order things happen in.

A body of knowledge is one axis of change, which is what the Single Responsibility Principle means by a responsibility. It does not mean a module does one thing: `CdnConfigurator` has two actions and one reason to change. Count only changes that are likely, not every change you can imagine.

A module may be called at two different moments. That is one owner with two calls, so don't split it because the calls are far apart.

Signs that code was split by time:

- Types named for when they run, such as `PrepareStep` or `FinishStep`
- One step doing setup for a later step
- Two steps holding halves of the same concern
- Values passed along only so a later step can finish an earlier concern

Time-ordered stages are fine when each one owns a different kind of knowledge. They are a problem when time is the only thing separating them.

| Split by time | Split by knowledge |
| --- | --- |
| `PrepareStep`: builds assets and sets up the CDN | `AssetCompiler`: how assets are built |
| `UploadStep`: uploads | `StorageUploader`: how assets reach storage |
| `FinishStep`: activates the CDN and cleans up | `CdnConfigurator`: `prepare` and `activate`, both owned here |

A CDN change now touches one module, even though the CDN is used at two moments.

### When to split

Split a module when it holds more than one body of knowledge, and you can name each one.

Signs worth checking:

- Two likely changes would touch unrelated parts of it
- Its dependencies fall into groups that its methods use separately
- Its name needs an "and", or only a vague word covers it
- Testing one part needs setup that's irrelevant to another
- It is unusually long for its kind of module

A sign only prompts the check. The split is justified once you can name the separate bodies of knowledge. Length never blocks a change or requires a refactor on its own.

Don't split a module because:

- It is long but holds one body of knowledge
- Its steps happen at different times
- It might gain more knowledge later

Every module created by a split must hide something. If a piece only forwards calls, it stays where it was.

### Checking a boundary

**Boundary test.** Name a likely change and count the modules whose knowledge it changes. One means the boundary is good. Two or more means that knowledge needs a single owner. Callers updating a call, and a coordinator gaining a line for a new step, don't count.

**Map before moving code.** Before splitting existing code, list its steps, what each one depends on and does, and every place the steps are tied together. The ties show where knowledge is leaking and usually point at the real modules. A step can be a good mental model and still be a poor code boundary.

### Shared code

**Extract shared code when it is shared knowledge.** Ask whether changing one copy would force the same change in the other. If it would, give that knowledge one owner, even with only two copies. If not, the code only looks alike, so leave it, however many copies there are.

An extracted module follows the same rules as any other: it hides something, and it is named for what it knows, never for being shared.

### Coordinators

Each sequence has exactly one coordinator. Add a separate coordinator only when it hides something, and don't nest coordinators.

A coordinator holds its collaborators and knows the sequence. It may call modules in order, loop to do so, pass one module's output to the next, reshape data between their types, and act on a named field or case a module returned or a decision already made.

It never makes a new decision or computes something to find out what a result means. If it would have to, that module's return type is too shallow: return an outcome that names each case.

A coordinator reads as one line per action at a single level, so the sequence is visible without knowing how any step works.

### Dependencies between modules

A module may depend on another when it needs that module's answer to finish its own job. A module never calls another just because it comes next: that order belongs to the coordinator. Dependencies run one way, with no cycles. A module receives any collaborator that holds state or fronts something outside the program rather than building it. It may build a stateless collaborator that does neither.

### Testing

Test each piece of knowledge in the module that owns it, through that module's public surface. Test a coordinator only for its sequence: the order of steps, what each step receives, and that a failure stops the steps after it. Don't repeat a module's cases through its coordinator.

Use real collaborators inside the program, and replace only what fronts something outside it. Add a protocol for a module only when it has more than one real implementation, fronts something outside the program, or sits at a boundary the architecture's own rules define.

If a coordinator test needs a lot of setup unrelated to the sequence, check whether a boundary is wrong.

## Failure and recovery

**Check before changing.** Do every check that can be done without changing anything before the first change is made.

**Hardest to undo goes last.** Steps run in the order their dependencies require. Where the order is still free, put the steps that are harder to undo later, and anything other people can see or act on after everything that could still fail. The order is the coordinator's knowledge, so a change in it touches only the coordinator.

**Stop and report by default.** When a step fails, stop and say where things stand.

**Undo only when all of these hold:**

- Stopping would leave something broken that a person would have to fix by hand
- The undo touches only what this run changed
- Nobody else has seen or acted on those changes
- The undo can't lose anyone's work

Undo in reverse order. Each module owns how to undo its own work, and the coordinator owns only the order. If an undo fails, stop and report what's left over. Never retry it or guess.

**Say where things stand.** A failure after the first change reports what failed, what was done or undone, what wasn't done, and what to do next, in terms whoever started it would recognize. The coordinator knows which steps finished, so it supplies that part. A module reports only its own failure.

A failure before the first change only needs to say what's wrong and what to do next.

**A re-run is always safe.** Running work again after it failed or was interrupted never damages or duplicates anything. It may repeat harmlessly, continue from where the last run stopped, or refuse and say why.

Continue when work stays behind after a failure and redoing it is costly or impossible. To continue, check what actually exists rather than a saved record of progress. Skip a step only when its result is confirmed to be what this run would have produced. If something is there but doesn't match, stop and report it.

**An interruption can stop the work at any moment.** Cleanup on interruption restores only what the program borrowed, such as the terminal or child processes. It doesn't undo work or start new steps. Recovery is the re-run's job.

A step that must never be left half done is made atomic, such as writing to a temporary file and renaming it over the original.

## Exceptions

A rule can be wrong or incomplete. When following one seems wrong, stop and ask before departing from it. Name the rule, say why it doesn't fit this case, show what was tried within the rule, and propose the alternative. Matching existing code is not a reason on its own. An exception approved more than once is a sign the rule itself should change.
