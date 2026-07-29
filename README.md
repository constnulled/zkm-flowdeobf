# Static control-flow deobfuscator

<p align="center">
  <img src="./img/deobf.jpg" alt="Static control-flow deobfuscator">
</p>

An offline, conservative ASM 9.7.1 command-line toolkit for statically
normalizing obfuscated JVM archives. It never loads, initializes, executes, or
reflectively queries classes from the input archive. Each class is parsed as an
ASM `ClassNode`; methods are analyzed with `Analyzer<BasicInterpreter>`. Every
mutation is transactional: a class is published only after it passes ASM
structural checks (`CheckClassAdapter`), `BasicVerifier`, and any stage-specific
postcondition. On any parse, analysis, or verification failure the original
class bytes are retained and the reason is recorded.

## Requirements

- Java / Javac 17
- Gradle 8.x with dependencies cached (offline builds supported)
- ASM 9.7.1 (Gradle-managed)
- CFR 0.152, for the CFR structure audit only. Point the audit scripts at your
  local `cfr-0.152.jar`; it is not bundled.

## Core pass: `ControlFlowDeobfuscator`

```text
gradle run --args="input.jar output.jar report.tsv"

gradle run --args="input.jar output.jar report.tsv \
  --remove-identity-rethrows --normalize-exception-ranges \
  --normalize-zkm-shared-tails --materialize-residual-stacks \
  --specialize-binary-sentinel-tails --clean-constant-residual-islands"
```

The output archive preserves non-class resources. A method is changed only when
the transformation is local and the resulting method passes ASM analysis and
class structural checks. The report is tab-separated with columns `class`,
`method`, `offset`, `action`, and `reason`. `offset` is the original
instruction index (a stable static location; the ASM tree does not retain
original byte offsets).

### Implemented safe reductions

* folding direct integer constant comparisons (`ICONST`, `BIPUSH`, `SIPUSH`,
  integer `LDC` followed by `IF*`) and strictly adjacent constant
  `LCMP`/`FCMP`/`DCMP` followed by `IF*`, including `ACONST_NULL` with
  `IFNULL`/`IFNONNULL`;
* removal of executable instructions whose analyzer frame is unreachable;
* removal of `GOTO` whose target is the next executable instruction;
* coalescing consecutive labels and updating all references;
* strict ZKM residual-stack integer/null guard recovery;
* proof-gated materialization of residual operand stacks across guarded edges;
* exact zero/nonzero tail specialization for closed binary sentinel locals,
  without assuming either runtime value;
* removal of side-effect-free
  `int, (LDC-long|LLOAD), (LDC-long|LLOAD), LXOR, POP2, POP` residual islands
  with at least one literal operand, when CFG, exception-range, and
  verifier-frame proofs pass;
* proof-gated removal of typed identity-rethrow exception handlers, including
  mixed semantic-handler components and the exact
  `ASTORE x; ALOAD x; identity; ATHROW` form;
* handler-order-preserving normalization of crossing exception ranges; and
* normalization of proven ZKM shared tails.

### ZKM guard-group acceptance

The ZKM rewrite accepts a method-local guard group only when all of the
following hold:

* at least two guards using the same local and opcode leave another operand on
  the JVM stack;
* the local is not a receiver or parameter slot, has exactly one typed store,
  and every other use is an adjacent `ILOAD` + `IFEQ`/`IFNE` or `ALOAD` +
  `IFNULL`/`IFNONNULL` guard;
* at least one non-residual guard uses the opposite opcode, while no residual
  guard uses it.

Residual guards and non-taken standalone guards are removed. An opposite guard
is restored to `GOTO` only when the original fall-through block already has a
stack-map frame. Otherwise the fake local load is replaced by a constant that
makes the original conditional branch taken; this is reported as
`zkm-pin-taken-guard` and preserves the original StackMapTable block shape.
Every site is reported independently as `zkm-remove-residual-guard`,
`zkm-remove-never-guard`, `zkm-restore-goto`, or `zkm-pin-taken-guard`. The
complete method is still rolled back if ASM analysis, `BasicVerifier`, or class
structural validation fails.

The default invocation is fail-closed. `--assume-zkm-sentinels` is an explicit
analysis mode for structurally identified mutable ZKM sentinels. One supported
family uses a proven `<clinit>` seed when every non-initializer setter call is
in the same method after its final sentinel guard and no control-flow edge
returns to the guarded prefix. This is reported as
`clinit-seed+post-guard-self-write`; repeated invocation and 32-bit integer
overflow are explicitly not claimed as a formal proof. The rule is not active in
the default safe mode.

### Identity-rethrow removal

Identity-rethrow removal preserves the ordered residual handler signature for
every protected instruction. Catch-all handlers, type annotations, external
handler entry, unknown exception hierarchy, and any signature mismatch are
rejected. A synchronized method or explicit monitor region is eligible only when
the identity handler body is monitor-transparent and the same route proof
retains every real catch-all/finally `MONITOREXIT` path. Methods with
unsupported or invalid control flow are left byte-for-byte unchanged. The proof
model is synchronous JVM execution; asynchronous `Thread.stop`/JVMTI injection
timing is outside the equivalence claim.

`--specialize-binary-sentinel-tails` and `--clean-constant-residual-islands` are
explicit opt-in stages. Each is fail-closed and publishes a class only after an
atomic class transaction, `CheckClassAdapter`, `BasicVerifier`, and a
fixed-point postcondition pass.

This pass does not remove ZKM long-zero guards, generally flatten switch
dispatchers, or guess arbitrary opaque predicates.

## Reproducible control-flow fixed point and JVM load differential

The control-flow-only audit starts from an explicitly hashed input, runs the
current `ControlFlowDeobfuscator` until a zero-change round, and then runs
residue, CFR, and real JVM class-loading checks. It does not invoke the DES
pipeline or publish a product JAR.

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File `
  .\scripts\Invoke-ControlFlowFixedPointAudit.ps1 `
  -InputJar .\input.jar `
  -OutputDir .\control-flow-fixed-point-audit `
  -ExpectedInputSha256 <sha256-of-input.jar> `
  -MaxRounds 8 `
  -SpecializeBinarySentinelTails `
  -CleanConstantResidualIslands
```

This is the fail-closed mode. Add `-AssumeZkmSentinels` only to reproduce the
explicit structural-analysis mode described above; that switch is recorded in
`metadata.txt` and is not promoted to a formal constant proof. Use
`-SkipCfrAudit` for a quick candidate/residue/JVM-gate run when the full CFR
audit will be launched separately against the resulting `final.jar`.

The script runs `gradle --offline --no-daemon --console=plain test`, constructs
the runtime classpath from the compiled classes and Gradle-managed ASM 9.7.1
dependencies, and records every command and source-file hash.

`scripts/ArchiveJvmLoadAudit.java` is independent of ASM. For every root class it
creates a fresh child-first loader backed by the complete archive, calls
`Class.forName(name, false, loader)`, and then `resolveClass`; no class is
initialized. Missing external dependencies are retained in the raw baseline and
candidate TSV files and compared by class, exception type, and message. The gate
fails on a baseline-pass/candidate-fail regression, a missing candidate class, or
a new or changed `ClassFormatError`/`VerifyError` family failure.

The JVM differential can also be run independently:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File `
  .\scripts\Invoke-JvmClassLoadDiff.ps1 `
  -BaselineJar .\baseline.jar `
  -CandidateJar .\final.jar `
  -OutputDir .\jvm-load-independent
```

## Reproducible CFR structure audit

Run the audit from PowerShell with an explicit execution-policy override:

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File `
  .\scripts\Invoke-CfrStructureAudit.ps1 `
  -InputJar .\input.jar -OutputDir .\cfr-structure-audit
```

The script performs a fresh CFR 0.152 decompilation and emits method-attributed
`findings.tsv`, `files.tsv`, `methods.tsv`, `summary.tsv`, `summary.md`, and
input/tool hashes in `metadata.txt`. `blockN` labels are readability signals,
not presumed dead-code deletion targets. Interpreting the CFR signal deltas
(e.g. `Unable to fully structure code`, `Loose catch block`,
`ConfusedCFRException`, and `blockN` counts) as a structuring improvement is a
review conclusion, not a claim that semantic names or the remaining runtime
closure are complete.

## CFG fingerprint scan

The read-only scanner is available as
`cn.openvape.flowdeobf.CfgFingerprintScanner`:

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.CfgFingerprintScanner input.jar output-dir label
```

It emits class, method, and candidate TSV files. Candidates include state
variables, dispatcher-shaped switches, and syntactically provable opaque
predicates, with confidence and evidence. The scanner parses with ASM only; it
does not define, load, initialize, or execute input classes. Results are review
priorities, not automatic rewrites.

For each state-like switch, the scanner also emits `recovery.tsv`. This is a
report-only CFG normalization suggestion: each row records the case/default
target block, executable range, normal and exception successors, a conservative
back-edge/SCC marker, state-store count, and opaque-state evidence. Every row
uses a `review_only` recommendation; no block order or class bytes are changed.
The output is suitable for manually auditing dispatcher candidates before any
semantic rewrite is attempted.

The scanner also recognizes report-only ZKM flow fingerprints: long-zero guards,
repeated integer/null guards with residual operand-stack values, and opposite
conditional jumps on the same local that may encode a replaced `GOTO`. The
repeated-local requirement avoids the common false positive from ordinary
ternary-expression bytecode. These rows remain `review_only`: guard values can
come from mutable cross-class static state, so stack shape alone is not a
branch-truth proof.

## ZKM string recovery and rewriting

`ZkmStringDecryptor` reconstructs an archive's stateful class-long-key model,
decrypts the outer packed tables and inner `(IJ)String` sites, and can emit a
separate verified JAR. It never replaces the input archive and never defines,
loads, or initializes an input class.

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmStringDecryptor \
  input.jar report-dir [rewritten.jar]
```

The table parser binds each packed literal to the exact `String[]` field and
allocated capacity. Multiple literals are concatenated in initializer order; the
initial DES chunk length is read from bytecode instead of guessed. Length
markers above 255 are supported because only ciphertext chunks, not framing
characters, must be Latin-1.

Class keys are recovered from two isolated state-model branches: the current
transactional order with mandatory superclass commits, and an independent JVM
hierarchy order. Their class-by-class union accepts equal keys, rejects and
reports conflicts, and uses outer/inner DES success as the validation oracle.
Argument values are taken from unique ASM `SourceInterpreter` producers, with the
adjacent-expression evaluator retained as a fallback.

The optional rewrite preserves all argument evaluation. Each verified
`invokedynamic (IJ)String` is replaced by `POP2`, `POP`, and a plaintext `LDC`,
which has the same net stack effect and does not alter preceding side effects or
stack-map boundaries. Each changed class passes `CheckClassAdapter` and
`BasicVerifier`; the complete output archive is verified again after writing. The
report separates integrity and coverage gates.

## ZKM integer-index recovery and rewriting

`ZkmIntegerDecryptor` applies the same offline class-key model to standard
`invokedynamic (IJ)I` sites. It identifies each class-local integer bootstrap,
recovers its packed `long[]` table with DES/CBC/NoPadding, resolves unique
integer and long producers with `SourceInterpreter`, and decrypts the low 32
bits of the selected inner DES block.

`ZkmIntegerDecryptor.proveClassKey(ClassNode, long)` exposes the same analysis as
a pure static, immutable proof. It passes only when the bootstrap, decryptor
specification, class-key field, outer mask, and packed long table are each unique
and every integer site has resolvable arguments, an in-range table index, and
successful outer and inner DES processing. The proof never defines or initializes
the input class.

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmIntegerDecryptor \
  input.jar report-dir [rewritten.jar]
```

The optional rewrite preserves argument evaluation by replacing a proven site
with `POP2`, `POP`, and the narrowest JVM integer constant instruction. A changed
class is committed only after `CheckClassAdapter` and `BasicVerifier` pass; the
output archive is verified again after writing. `sites.tsv`, `tables.tsv`,
`rewrite.tsv`, `audit.txt`, and `gate.txt` distinguish argument, table, DES,
key-validation, and class-rollback failures. As with the string pass, no input
class is defined, initialized, or executed.

`ZkmClassKeySelector` can use this proof as an explicit final oracle on a
remaining-key archive, after the string/member/enum-selected owners have already
been removed:

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmClassKeySelector \
  remaining.jar selector-report --residual-integer-oracle

java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmLongKeyDeobfuscator \
  remaining.jar long-key-report rewritten.jar --residual-integer-oracle
```

The equivalent static API is `ZkmClassKeySelector.selectResidual(classes)`. The
default selector and long-key deobfuscator paths do not enable this stage. The
residual API rejects a mixed-stage archive if a string, member, or enum oracle
still selects any owner. DES/CBC/NoPadding has no authenticity check, so applying
the integer oracle to mixed-stage frontier candidates can admit accidental
in-range random plaintexts. `integer-candidates.tsv` records every candidate's
bootstrap/spec/table cardinality, per-site coverage, values, and exact failure
code; selection still requires exactly one candidate to pass all sites. Selector
and long-key audit files record the staged input SHA-256.

## ZKM member invokedynamic directization

Run `ZkmMemberIndyDirectizer` before bootstrap cleanup. It decodes each
owner-local ZKM member `invokedynamic` site, projects the decoded raw member
through the current class and method mappings, preserves the call-site type
adaptation with explicit reference casts, and replaces the site with an ordinary
JVM method instruction:

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmMemberIndyDirectizer \
  input.jar report-dir class-map.tsv method-map.tsv [rewritten.jar]
```

The four-argument form is an audit-only dry run. A rewrite requires unique
constant producers for both trailing `long` keys, a decoded method target, a
collision-safe mapping projection, a reference-only descriptor adaptation, and a
known class/interface invocation mode. Changed classes are verified individually;
the optional archive is then verified and atomically committed. Unresolved sites
remain intact and are listed in `sites.tsv`, `rewrite.tsv`,
`projection-audit.txt`, and `audit.txt`. When a decoded member declaration is
absent from the original class, the tool deliberately does not guess an alias.

## ZKM bootstrap support cleanup

`ZkmBootstrapSupportCleaner` is the post-rewrite cleanup pass for member, string,
and integer bootstrap infrastructure. Run it after member `invokedynamic` sites
have been converted to ordinary bytecode:

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmBootstrapSupportCleaner \
  input.jar report-dir [rewritten.jar]
```

The two-argument form is a dry run. It discovers support families, constructs
their private same-class method and MethodHandle dependency graphs, proves
archive reference closure, plans `<clinit>` producer slices, and verifies the
simulated class changes without writing an archive. The optional output is
written to a temporary file, verified as a complete archive, and atomically
committed.

A family is removed only when all of these gates hold:

* its bootstrap, linker, resolver or decryptor, table fields, and table populator
  methods match a descriptor-based ZKM family shape;
* every method and field is private static and has no archive reference outside
  the family or the planned initializer slice;
* each initializer slice has no incoming control edge, exception-range boundary,
  escaping local or stack value, or unknown side effect;
* the rewritten class passes `BasicVerifier`, the named members are confirmed
  absent, and the complete output archive passes verification.

`families.tsv`, `blockers.tsv`, `rewrite.tsv`, `removed-members.tsv`, and
`audit.txt` distinguish live bootstrap references from uncertain initializer
boundaries. A live member bootstrap is an expected skip when this pass is run
before member directization; it is never treated as permission to delete the
resolver. No input class is defined, loaded, initialized, or reflectively
queried.

## Unified DES recovery pipeline

`ZkmDesDeobfuscator` is the archive-wide entry point for DES recovery. It runs
the proof-driven stages in order, feeds each verified archive into the next
stage, and keeps the individual candidate/rejection/verifier reports:

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmDesDeobfuscator \
  input.jar report-dir [rewritten.jar]
```

The current stages recover closed PKCS5 single-string constants, local
string/enum arrays, mixed-initializer string fragments, strict NoPadding
scalar-long initializers, local NoPadding `long[]` tables, and literal-key
NoPadding scalar islands embedded in mixed initializers. It also directizes
proven constant `(IJ)I` helper/indy sites and deletes the now-unreferenced
NoPadding helper closure. The delayed PKCS5 `(IJ)String` family is handled by
proving its outer field table, index formula, and every constant call site, then
replacing calls with plaintext and deleting the table/helper/bootstrap closure.
Template implementations remain separate so an unsupported shape cannot weaken
another stage's proof. New DES shapes are added as stages behind the same entry
point.

State-derived NoPadding scalar islands use the statically reconstructed
`ZkmLongKeyState` graph plus JVM superclass/default-interface initialization
order. The pass independently proves the local `classKey ^ mask` data flow,
retains the original key bootstrap and all owner field writes, and rejects an
owner when no proven key provenance is available.

The pipeline has two independent gates. `integrity_gate` requires every changed
class and the final archive to verify without rollback. The stricter
`coverage_gate` requires the final archive to contain zero constant `DES/*`
`Cipher.getInstance` sites. The overall gate cannot pass while any DES residue
remains, even when all performed rewrites are structurally valid. The input
classes are parsed as ASM data and are never defined, loaded, or initialized.

All stages share three proof primitives:

* `ZkmDesConstantEvaluator` implements the accepted PKCS5/NoPadding, Latin-1, BMP
  string, scalar-long, and packed-table semantics. It refuses a key whose
  provenance is marked as a hypothesis; recognizers must provide a literal,
  validated class key, or a proven derivation such as `classKey ^ mask`.
* `ZkmCfgIslandProof` checks control-flow entries/exits, exception containment,
  boundary stack heights, field/method effects, and live-out locals before a
  crypto region is replaceable.
* `ZkmClassRewriteTransaction` serializes each mutation in isolation, runs ASM
  structural checks and `BasicVerifier`, applies a stage-specific semantic
  postcondition, and returns the original bytes on any failure.

This separation is the extension contract for project-wide recovery: a new
template supplies bytecode provenance and an exact replacement, while DES
evaluation, CFG proof, verification, and rollback stay common.

## Residual DES helper audit and NoPadding directization

`ZkmDesHelperDeobfuscator` is a standalone post-table pass for residual
`(IJ)String` and `(IJ)I` DES helpers. It reports both families and rewrites only
the strict NoPadding integer form whose table, selector, and call arguments are
all statically proven:

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmDesHelperDeobfuscator \
  input.jar report-dir [rewritten.jar]
```

The pass handles explicit `long[]` tables and the unique fixed-eight-byte packed
table form. It preserves argument evaluation with `POP2`, `POP`, and an integer
constant, then removes the private helper/bootstrap/linker closure only when the
archive reference graph is closed. Calls depending on an unvalidated long-key
field remain unchanged. This entry point is intentionally separate from
`ZkmDesDeobfuscator`; it does not weaken or silently extend a pipeline stage's
proof policy.

## Field-backed PKCS5 string helpers

`ZkmDirectStringHelperDeobfuscator` handles the two-layer residual string family
whose outer PKCS5 initializer publishes an encrypted `String[]` table and whose
private `(IJ)String` helper decrypts table entries on demand:

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmDirectStringHelperDeobfuscator \
  input.jar report-dir [rewritten.jar]
```

The pass binds the packed outer table to its exact field and capacity, proves the
canonical ZKM byte decoder and inner index formula, and resolves every live
helper argument from a unique `SourceInterpreter` producer. Calls are replaced
with `POP2`, `POP`, and plaintext `LDC`, preserving argument evaluation. Private
bootstrap adapters are deleted only when their signature-matched reference
subgraph has no external inbound edge. Once all helper references are gone, the
helper and byte decoder are removed and the outer crypto entry is redirected to
the original business continuation; `BasicVerifier` must prove the crypto blocks
unreachable before they are physically pruned.

Dynamic live arguments, an unvalidated long-key-derived outer key, ambiguous
tables, noncanonical decoders, and nonclosed bootstrap references are rejected.
Every changed class is transactional and the optional archive is published only
after whole-archive verification. The package-static entry is
`rewrite(Path, Path, Path)` and its result is `Summary`; this pass remains
separate from the unified pipeline until its stage ordering is selected.

## Direct static DES string constants

`ZkmDirectStringConstantDeobfuscator` folds a closed single-field
`DES/CBC/PKCS5Padding` initializer:

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmDirectStringConstantDeobfuscator \
  input.jar report-dir [rewritten.jar]
```

The two-argument form is an audit-only dry run. The pass requires exactly one
constant DES key, one Latin-1 ciphertext, the canonical big-endian key-byte loop,
the fixed JCE and byte-decoder call set, and one write to a declared
`static final String` field. It replaces the complete crypto-only `<clinit>` with
a plaintext `LDC` and `PUTSTATIC`. The private static byte decoder is removed only
when the initializer call is its sole archive reference.

This stage deliberately rejects `DES/CBC/NoPadding`, local `String[]` and enum
name tables, exception-covered or mixed business initializers, and all dynamic
inputs. Changed classes are verified transactionally and the optional archive
must pass whole-archive verification before atomic publication. The input classes
are never defined or initialized. `candidates.tsv`, `removed-helpers.tsv`,
`verifier.tsv`, `audit.txt`, and `gate.txt` record the proof and every
conservative rejection.

## Direct local `String[]` DES tables

`ZkmDirectStringArrayDeobfuscator` is the separate second-stage pass for the
closed local string-array form used by enum and object initializers. It folds the
canonical PKCS5 key setup, packed Latin-1 ciphertext chunks, and the exact ZKM
byte decoder into an explicit `new String[N]` plus plaintext `AASTORE` sequence,
while preserving the initializer's following business code and any
NoPadding/integer table. Both the direct loop and the split-block CFG variant are
accepted only when the crypto island has one entry, one exit, empty stacks at its
boundaries, no exception range, and no dynamic input. Outside the island the array
local must be consumed only by constant-index `AALOAD`; whole-array stores, calls,
and other escapes are rejected. A decoder is removed only when the archive
reference count proves that the rewritten loop sites are its sole references.

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmDirectStringArrayDeobfuscator \
  input.jar report-dir [rewritten.jar]
```

The two-argument form is an audit-only dry run. Reports are written as
`candidates.tsv`, `tables.tsv`, `removed-helpers.tsv`, `verifier.tsv`,
`audit.txt`, and `gate.txt`.

## Direct mixed-initializer DES string fragments

`ZkmDirectStringFragmentDeobfuscator` handles the strict mixed form that the
single-field pass intentionally rejects. It proves a unique `DES/CBC/PKCS5Padding`
island with one constant key, one Latin-1 ciphertext, the canonical JCE call set,
one private static byte decoder, and one static final `String` target. The island
must have a single forward exit, no incoming edge from outside, and empty operand
stacks at both boundaries. A surrounding exception range is accepted only when it
completely contains the island and its handler is outside it; the range and
handler remain unchanged. The pass replaces only that island with `LDC` plus
`PUTSTATIC`; all instructions before and after it remain in their original order,
preserving surrounding state setup, object construction, and collection creation.

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmDirectStringFragmentDeobfuscator \
  input.jar report-dir [rewritten.jar]
```

The two-argument form is an audit-only dry run. A decoder is removed only when its
archive reference count is exactly the one call in the removed island.
`DES/CBC/NoPadding`, multiple crypto islands, partial/intersecting exception
ranges, dynamic inputs, and non-empty boundary stacks are rejected
transactionally. `candidates.tsv`, `removed-helpers.tsv`, `verifier.tsv`,
`audit.txt`, and `gate.txt` record proofs and conservative rejections.

## Read-only residue inventory

`DeobfuscationResidueScanner` inventories bytecode that still has a known
obfuscation shape after the staged passes. It parses class bytes with ASM only;
it never defines, loads, initializes, or reflectively queries an input class:

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.DeobfuscationResidueScanner \
  input.jar report-dir
```

The report contains `classes.tsv`, `methods.tsv`, `sites.tsv`, `audit.txt`, and
`gate.txt`. It records archive and parse coverage, statically proven ZKM long-key
bootstrap/field/read chains, exact `(IJ)String` and `(IJ)I` `invokedynamic` sites,
member sites with trailing `JJ`, DES PKCS5/NoPadding factory calls split between
`<clinit>` and helpers, MethodHandle/MutableCallSite and bootstrap method/field
handles, and identity-rethrow handlers including typed `T -> T` methods. A row is
emitted only for a bytecode shape the scanner can establish; CFR structuring
failures and inferred semantics are deliberately not counted.

## Atomic ZKM runtime closure

`ZkmRuntimeClosureDeobfuscator` folds the long-key chains, directizes every
remaining `(IJ)I` site, removes the central obfuscation-runtime classes, and
invokes bootstrap-family cleanup in a private staging archive. It publishes the
requested output only after the final archive contains no runtime class,
descriptor, field, method, handle, raw constant-pool, integer-indy, private
bootstrap, or `MutableCallSite` residue:

```text
java -cp control_flow_deobfuscator.jar \
  cn.openvape.flowdeobf.ZkmRuntimeClosureDeobfuscator \
  input.jar report-dir output.jar pre-removal-authority.jar
```

No input class is loaded or initialized. A NoPadding integer table has no
authentication tag, so successful DES processing and an in-range index are not by
themselves a unique key proof. The automatic four-argument authority path
therefore rejects ambiguous candidates and does not create `output.jar`.

An optional fifth argument is an explicit, separately audited manual-semantic
evidence file. It is never read implicitly from the application resources:

```text
... input.jar report-dir output.jar authority.jar semantic-evidence.tsv
```

Evidence must cover every integer site in each named owner and bind the exact
class, method, descriptor, frame-independent instruction coordinate, integer
argument, and `literal ^ (keyField ^ mask)` expression. It can select a key only
when the pre-removal authority candidate set leaves exactly one candidate that
decrypts all recorded values. `semantic-evidence.tsv`, `failures.tsv`,
`audit.txt`, and `gate.txt` distinguish a bytecode-fingerprint match from a unique
candidate proof. Expected values are never written directly when the candidate
proof fails.

## 许可证

本仓库采用 [MIT](LICENSE) 许可。
