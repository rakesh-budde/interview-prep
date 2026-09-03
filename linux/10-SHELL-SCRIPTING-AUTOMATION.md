# Section 10: Shell Scripting & Automation Mastery

This section covers Bash at a mastery level — expansion order, file-descriptor plumbing, job control,
signal trapping, text processing, and the discipline of writing production-grade, idempotent
automation scripts. This is the material behind both scripting interview exercises and real-world
production tooling reliability.

## Subtopic Index
- [POSIX Shell vs Bash Extensions](#posix-shell-vs-bash-extensions)
- [Variables, Arrays, Subshells](#variables-arrays-subshells)
- [Process Substitution and Command Substitution](#process-substitution-and-command-substitution)
- [Pipes and Redirection Internals (file descriptors, dup2)](#pipes-and-redirection-internals-file-descriptors-dup2)
- [Exit Codes and set -e/-u/-o pipefail](#exit-codes-and-set--e-u-o-pipefail)
- [Signal Trapping in Scripts (trap)](#signal-trapping-in-scripts-trap)
- [Here-documents and Here-strings](#here-documents-and-here-strings)
- [Text Processing (sed, awk, grep internals, regex engines)](#text-processing-sed-awk-grep-internals-regex-engines)
- [xargs and Parallelism](#xargs-and-parallelism)
- [Job Control (bg, fg, disown, nohup)](#job-control-bg-fg-disown-nohup)
- [Writing Idempotent, Production-Grade Scripts](#writing-idempotent-production-grade-scripts)

---

## POSIX Shell vs Bash Extensions

POSIX defines a standardized shell command language specification that any conforming shell (`dash`,
`ksh`, `bash` running in POSIX mode) must implement identically, specifically so that portable scripts
declaring `#!/bin/sh` behave predictably regardless of which actual POSIX-compliant shell a given
system happens to symlink `/bin/sh` to. Bash implements the POSIX specification as a baseline but adds
a substantial number of non-portable extensions on top — arrays (`declare -a`, entirely absent from
POSIX sh), the `[[ ]]` extended test construct (supporting pattern matching and regex without the
word-splitting/globbing hazards of POSIX's `[ ]`/`test`), C-style `((...))` arithmetic evaluation and
`for ((;;))` loops, process substitution (`<(...)`/`>(...)`, discussed below), brace expansion
(`{a,b,c}`, `{1..10}`), the `local` keyword for genuinely scoped function-local variables, and
associative arrays (`declare -A`) — none of which are guaranteed to exist, or to behave identically,
under a strictly POSIX-only shell like `dash`. This distinction has real, concrete operational
consequences: a script written and tested under an interactive `bash` session but declaring `#!/bin/
sh` as its shebang can fail unpredictably (sometimes silently producing wrong results rather than a
clear error) when actually executed on a system where `/bin/sh` is symlinked to `dash` (the Debian/
Ubuntu default) rather than `bash`, since `dash` simply doesn't implement bashisms like arrays or
`[[ ]]` at all — this exact class of bug is common enough in container images (many minimal base
images, like Alpine's, use `busybox`'s even more minimal `ash` as `/bin/sh`) that it's a standard,
well-known "gotcha" specifically worth defending against by either declaring `#!/bin/bash` explicitly
(and ensuring bash is actually installed in the target environment) whenever bash-specific features are
used, or by deliberately writing genuinely POSIX-compliant scripts (avoiding bashisms entirely) when
portability/minimal-image-size across varying shell implementations is a priority. `checkbashisms` (a
Debian-packaged static-analysis tool) can automatically flag bash-specific constructs in a script
claiming POSIX `sh` compatibility, providing a concrete, automatable way to catch this class of bug
before it reaches production rather than discovering it only when a script unexpectedly fails on a
different system's `/bin/sh`.

### Key commands
```
readlink -f /bin/sh                # confirm which actual shell /bin/sh is symlinked to on this system
checkbashisms myscript.sh             # static analysis flagging bash-specific constructs in a POSIX-sh script
bash --posix -c 'source myscript.sh'    # test a script's behavior under bash's own POSIX-compatibility mode
dash myscript.sh                        # directly test under a strict POSIX shell, if installed
```

## Variables, Arrays, Subshells

Bash variables are untyped strings by default (even numeric-looking assignments are stored as text
unless explicitly used in an arithmetic context), assigned with no spaces around `=`
(`VAR=value`, since `VAR = value` is instead parsed as attempting to run a command literally named
`VAR` with arguments `=` and `value`) and referenced with `$VAR` or, for safety against ambiguous
word-boundary parsing, `${VAR}`. Arrays (`arr=(one two three)`, indexed numerically from zero by
default, or associative arrays via `declare -A assoc` with arbitrary string keys) are a bash-specific
extension absent from POSIX sh, and correctly expanding an array's elements as separate, individually-
quoted words (`"${arr[@]}"`, critically *with* the double quotes) versus a single space-joined string
(`"${arr[*]}"`, or the unquoted `${arr[@]}` which is subject to further word-splitting/globbing) is one
of the most consequential and commonly-misunderstood quoting distinctions in bash scripting, directly
determining whether an array of filenames containing spaces is handled correctly (`"${arr[@]}"`,
preserving each element as one distinct word regardless of embedded whitespace) or silently mangled
(any other form). A subshell is a genuinely separate child shell process, created explicitly by
wrapping commands in `(...)` (as opposed to `{...}`, which groups commands in the *current* shell
without forking) or implicitly by constructs like command substitution and pipeline elements running
in the background — critically, variable assignments and `cd` calls made *inside* a subshell are
entirely invisible to the parent shell once the subshell exits, since it's a genuinely distinct
process with its own independent copy of the environment, which is precisely why a loop like
`cat file | while read line; do count=$((count+1)); done; echo $count` frequently surprises engineers
by reporting `count` as still zero afterward — the `while` loop's right-hand side of a pipe runs in a
subshell in traditional POSIX-pipe semantics (though bash's `lastpipe` shell option can change this
specific case), so `count`'s increments inside the loop never persist back to the parent shell's own
variable of the same name once the pipeline completes.

### Key commands
```
arr=("one two" three); printf '%s\n' "${arr[@]}"   # correct, safe array expansion preserving embedded spaces
declare -A map=([key1]=val1 [key2]=val2); echo "${map[key1]}"   # associative array usage
(cd /tmp && ls)                        # subshell: cd here does not affect the calling shell's directory
shopt -s lastpipe                        # (bash-specific) let the last pipeline element run in the current shell, not a subshell
```

## Process Substitution and Command Substitution

Command substitution (`$(command)`, or the older backtick syntax `` `command` `` which nests and
escapes far more awkwardly and is generally discouraged in new scripts) captures a command's standard
output as a string, with trailing newlines stripped, substituting that captured text directly into the
surrounding command line — `result=$(some_command)` is the standard idiom for capturing output into a
variable, and it necessarily runs the substituted command in a subshell (with all the
variable-assignment-invisibility consequences just discussed), since its output must be fully captured
before the surrounding command can be constructed and executed. Process substitution (`<(command)`/
`>(command)`, a bash-specific extension entirely absent from POSIX sh) is a related but distinct and
more powerful construct: rather than capturing output as a string, it creates a named, filesystem-
visible reference (implemented as a special file under `/dev/fd/` on Linux, backed by an anonymous
pipe connected to the substituted command's stdin/stdout) that can be passed anywhere a regular
filename argument is expected — `diff <(sort file1) <(sort file2)` runs `sort` on each file
independently and feeds each command's *live output stream* directly to `diff` as if it were reading
from two ordinary files, without ever needing an intermediate temporary file on disk and without
requiring `diff` itself to have any special support for reading from a pipe rather than a real file.
This distinction matters practically: command substitution is appropriate when you need a program's
*entire* output captured as a string for further string manipulation or a single scalar value; process
substitution is appropriate when you need to feed a command's output as if it were a *file* to another
program that expects file arguments specifically (and, notably, expects to potentially `seek()` within
that "file" in some cases — process substitution's pipe-backed nature means genuine seeking doesn't
actually work the way it would on a real file, which occasionally surfaces as a subtle limitation for
programs that need true random-access file behavior rather than a sequential stream).

### Key commands
```
files=$(ls *.txt)                 # command substitution: capture output as a string
diff <(sort a.txt) <(sort b.txt)     # process substitution: feed two live command outputs as "files" to diff
while read -r line; do echo "$line"; done < <(some_command)   # process substitution avoiding the subshell pipe-to-while issue
tee >(gzip > out.gz) < input.txt       # process substitution as an output target, splitting a stream to multiple consumers
```

## Pipes and Redirection Internals (file descriptors, dup2)

(The kernel-level mechanics of pipes and file descriptor inheritance are covered in Section 1; this
entry focuses on the shell-scripting-level syntax and its precise semantics.) Every redirection
operator the shell provides is ultimately syntactic sugar over the same underlying `open()`/`dup2()`/
`close()` syscall sequence performed before `execve()`-ing the target command, and understanding this
precisely resolves several commonly-confusing redirection idioms. `command > file 2>&1` redirects
stdout to the file *first*, then makes fd 2 a duplicate of whatever fd 1 currently points to (the just-
opened file) — order matters critically here: `command 2>&1 > file` instead first duplicates fd 2 onto
whatever fd 1 *currently* points to (typically the terminal, if this is the first redirection in the
command), and only afterward redirects fd 1 to the file, meaning stderr continues going to the
terminal while only stdout goes to the file, the opposite of the likely intended behavior — this
exact ordering trap is one of the single most common redirection mistakes in real-world shell
scripts. Custom file descriptors beyond the standard 0/1/2 can be explicitly opened and used for more
sophisticated I/O plumbing (`exec 3< file` opens fd 3 for reading from a file for the remainder of the
script's execution, letting multiple different parts of a script read from the same file descriptor
sequentially without needing to reopen it, or `exec 3>&1; command >&3 3>&-` for temporarily
duplicating and later closing an extra reference to stdout) — a technique used in more advanced
scripts needing fine-grained control over exactly which stream goes where at each step, beyond what
the basic 0/1/2 redirection operators alone can express conveniently. `/dev/stdin`, `/dev/stdout`,
`/dev/stderr`, and `/dev/fd/N` provide a filesystem-path-based way to reference these same file
descriptors, useful specifically for programs that only accept file path arguments and have no native
concept of reading from an already-open file descriptor directly.

### Key commands
```
command > out.log 2>&1              # correct order: redirect stdout, then duplicate stderr onto it
command 2>&1 > out.log                 # common mistake: stderr still goes to terminal, only stdout to file
exec 3< file.txt; read -u 3 line; exec 3<&-   # open, use, and explicitly close a custom file descriptor
strace -e trace=dup2,open -f bash -c 'command > out 2>&1'   # observe the actual syscalls a redirection produces
```

## Exit Codes and set -e/-u/-o pipefail

Every command execution produces a numeric exit status (0 conventionally meaning success, any nonzero
value meaning some form of failure, with the specific nonzero value's meaning defined per-command by
convention, not by any kernel-enforced standard), retrievable immediately after execution via `$?` and
central to virtually all shell-level control flow (`if command; then`, `command && next_command`,
`command || fallback_command` all branch purely on this exit status). `set -e` (`errexit`) causes the
shell to immediately exit with a nonzero status the moment any simple command fails (returns nonzero),
specifically intended to prevent a script from silently continuing past a failed step under the
mistaken assumption everything is fine — but its actual behavior has several well-documented, commonly
-misunderstood exceptions (a command's failure inside an `if`/`while` condition, or as any but the last
element of a pipeline without `pipefail` also set, or within `&&`/`||` chains, does *not* trigger
`errexit`'s exit, since the shell reasonably assumes those contexts are deliberately testing/handling
the failure rather than accidentally ignoring it) that every engineer relying on `set -e` for safety
should understand precisely rather than assuming it catches every possible failure unconditionally.
`set -u` (`nounset`) causes referencing any unset variable to be treated as an error rather than
silently expanding to an empty string, catching typos in variable names (a classic, otherwise entirely
silent bug class) at the moment of reference rather than allowing a script to proceed with unintended,
empty-string-substituted behavior. `set -o pipefail` changes a pipeline's overall exit status to be
the *rightmost* nonzero exit status among all its stages (rather than bash's default behavior of a
pipeline's exit status being simply whatever its *last* command returned, regardless of any earlier
stage's failure) — without `pipefail`, `command_that_fails | grep pattern` reports success (grep's
own exit status) even if `command_that_fails` genuinely failed, silently masking the real failure
entirely unless `pipefail` is explicitly enabled to surface it. The now-standard defensive idiom
`set -euo pipefail` at the top of production scripts combines all three protections, and is
overwhelmingly considered a baseline best practice for any non-trivial production automation script,
though genuinely understanding each flag's specific scope and documented exceptions (rather than
treating the combined idiom as an unconditional safety guarantee) remains essential for writing
scripts that behave correctly rather than merely appearing safer.

### Key commands
```
set -euo pipefail                   # standard defensive header for production bash scripts
echo $?                               # check the most recent command's exit status
command; echo "exit status: $?"          # explicitly capture and display an exit status for debugging
false | true; echo $?                       # without pipefail: reports 0 (true's status), masking false's failure
set -o pipefail; false | true; echo $?        # with pipefail: correctly reports nonzero
```

## Signal Trapping in Scripts (trap)

The `trap` builtin registers a shell command (or function) to execute when the shell receives a
specified signal, letting a script perform graceful cleanup (removing temporary files, releasing
locks, logging a clear failure reason) regardless of *how* the script's execution ends — whether
through normal completion, an explicit `kill`, or an unexpected error triggering `set -e`'s exit.
`trap 'cleanup' EXIT` is the single most valuable pattern here: `EXIT` is a pseudo-signal that bash
fires whenever the shell (or script) is about to exit for *any* reason at all (natural completion,
an explicit `exit` call, or termination via a real signal not otherwise trapped and handled
separately), making a single `trap ... EXIT` registration a comprehensive, reliable "no matter what
happens, always run this cleanup" guarantee, meaningfully more robust than manually calling a cleanup
function at every single possible exit point in a script (which is both tedious and easy to
accidentally miss one path for, e.g., an early `return`/`exit` added later during script maintenance
that the original author didn't anticipate needing cleanup too). Trapping specific real signals
(`trap 'handler' TERM INT`) additionally lets a script react specifically and differently to an
explicit termination request versus other exit paths — useful for long-running scripts that need to,
say, finish their current unit of work cleanly rather than being abruptly killed mid-operation when
asked to stop, though it's important to recognize `SIGKILL` cannot be trapped at all (as discussed in
Section 2), so a script's own cleanup logic can never protect against a truly forceful, unconditional
kill — only against a `SIGTERM`-based graceful-shutdown request the caller was courteous enough to
send instead. A common, genuinely important production pattern combines `trap` with a temporary
resource, ensuring it's always cleaned up: `tmpfile=$(mktemp); trap 'rm -f "$tmpfile"' EXIT` guarantees
the temporary file is removed whether the script completes normally, errors out under `set -e`, or is
explicitly interrupted, entirely eliminating the class of bug where a script's early, unanticipated
exit path leaves stray temporary files accumulating on disk indefinitely.

### Key commands
```
trap 'rm -f "$tmpfile"' EXIT           # guaranteed cleanup on any exit path, including via set -e
trap 'echo "interrupted"; exit 130' INT   # custom handling for Ctrl-C specifically
trap -p                                    # list currently registered traps for the current shell
trap - TERM                                  # reset a specific signal's trap back to default behavior
```

## Here-documents and Here-strings

A here-document (`<<DELIMITER ... DELIMITER`) lets a script embed a multi-line block of literal text
directly inline, fed to a command's standard input as though it were a separate file, without needing
a genuinely separate file on disk at all — extremely common for embedding configuration file content,
SQL statements, or multi-line messages directly within a script's own source rather than requiring an
accompanying external template file. By default, variables and command substitutions *within* a
here-document's body are expanded (`cat <<EOF\nHome directory: $HOME\nEOF` substitutes the actual
value of `$HOME`), which is usually the desired behavior for generating dynamic content — but quoting
the delimiter (`<<'EOF'` instead of `<<EOF`) disables this expansion entirely, treating the entire body
as fully literal text, which is essential when the embedded content itself needs to contain literal
`$`, backticks, or other characters the shell would otherwise try to interpret/expand (embedding a
literal shell script template, or SQL containing `$1`-style positional parameters not meant for the
outer shell to touch at all). A here-string (`<<< "text"`) is a simpler, single-line variant providing
a string directly as standard input without the multi-line delimiter ceremony, a convenient shorthand
for `command <<< "$variable"` in place of the more verbose `echo "$variable" | command` idiom, with the
practical advantage of not spawning an extra `echo` subprocess/pipe just to feed a single string into
a command's stdin. Both constructs are, again, bash-specific extensions with no POSIX sh guarantee
(here-documents are actually POSIX-standardized and quite portable; here-strings specifically are the
purely bash-specific addition of the two), relevant to the same portability considerations discussed
earlier in this section when a script's target execution environment isn't guaranteed to be bash
specifically.

### Key commands
```
cat <<EOF > config.yaml
name: myapp
home: $HOME
EOF
                                        # here-document with variable expansion
cat <<'EOF' > script-template.sh
echo "literal \$HOME is not expanded here"
EOF
                                        # quoted delimiter: fully literal, no expansion
grep pattern <<< "$variable"              # here-string: simpler single-value stdin feeding
```

## Text Processing (sed, awk, grep internals, regex engines)

`grep`, `sed`, and `awk` form the classical UNIX text-processing toolkit, each occupying a distinct,
deliberately narrow niche in the "do one thing well" philosophy discussed in Section 1. `grep` searches
for lines matching a pattern (basic regular expressions by default, extended regular expressions with
`-E`, and Perl-compatible regular expressions with GNU grep's `-P` extension, each supporting a
progressively richer, though also progressively less portable/standardized, pattern syntax) and prints
matching lines, doing nothing beyond that single filtering job. `sed` (stream editor) applies
line-oriented text transformations — substitution (`s/pattern/replacement/`), deletion, insertion —
processing input one line at a time through an implicit pattern-space buffer, making it ideal for
straightforward, line-scoped find-and-replace operations but awkward for anything requiring genuine
cross-line context or arithmetic. `awk` is the most powerful and genuinely general-purpose of the
three, implementing a small, complete programming language organized around pattern-action pairs
(`pattern { action }`, executed once per input record, with fields automatically split and accessible
as `$1`, `$2`, etc., and `$0` for the whole line), supporting variables, arrays, arithmetic, and
user-defined functions — appropriate for genuinely non-trivial text-processing logic (computing sums/
averages across columns, reformatting structured output, conditional logic based on multiple fields
simultaneously) that would be awkward or impossible to express cleanly in `sed` alone. Regular
expression engines themselves come in two fundamentally different implementation families with
genuinely different performance characteristics: POSIX basic/extended regular expressions (and most
`grep`/`sed`/`awk` implementations, by default) typically use a finite-automaton-based matching
approach with guaranteed linear-time matching complexity relative to input length regardless of
pattern complexity, while Perl-compatible regular expressions (PCRE, used by `grep -P` and most
scripting languages' native regex support) use a backtracking approach supporting much richer pattern
features (lookahead/lookbehind assertions, backreferences) at the cost of potentially catastrophic,
exponential-time worst-case matching behavior for certain pathological pattern/input combinations
("catastrophic backtracking," a genuine, well-documented denial-of-service vector — ReDoS — when
untrusted input is matched against a naively-written backtracking regex), a distinction worth knowing
precisely when choosing which regex flavor/tool to reach for, particularly for any pattern matching
applied to untrusted, externally-supplied input.

### Key commands
```
grep -E 'pattern1|pattern2' file        # extended regex, alternation without backslash-escaping
sed -i 's/old/new/g' file                 # in-place substitution across a file
awk -F, '{sum += $3} END {print sum}' file  # sum a specific CSV column using awk's built-in arithmetic
grep -P '(?<=foo)bar' file                    # PCRE lookbehind assertion, unavailable in POSIX-flavor grep
```

## xargs and Parallelism

`xargs` bridges a fundamental mismatch between commands that produce a list of items on standard
output (like `find` or `grep -l`) and commands that expect those items as command-line arguments
rather than as piped standard input (`rm`, `chmod`, most ordinary utilities not specifically written to
read a target list from stdin) — reading whitespace/newline-delimited (or, more robustly,
NUL-delimited via `-0`, correctly handling filenames containing spaces or even embedded newlines that
plain whitespace-delimited parsing would mishandle) input and constructing and executing one or more
invocations of a target command with those items appended as arguments, automatically batching many
items into as few invocations as the target command's maximum argument-list length permits (avoiding
the classic "argument list too long" failure of naively trying to pass an enormous number of arguments
to a single command invocation directly). Beyond this core find-and-transform bridging role, `xargs
-P N` provides simple, effective parallelism: rather than invoking the target command once
sequentially per batch, it runs up to N invocations concurrently, which is a remarkably easy way to
parallelize an otherwise-sequential, embarrassingly-parallel batch operation (resizing a directory
full of images, running an independent validation check against many files) without needing to write
explicit background-process/job-management logic — `find . -name '*.jpg' -print0 | xargs -0 -P 8 -I{}
convert {} -resize 50% {}.small.jpg` processes up to 8 images concurrently with minimal additional
script complexity beyond the equivalent sequential version. A commonly-cited important safety
practice: always preferring `-print0`/`-0` (NUL-delimited) over plain newline/whitespace-delimited
input whenever filenames are involved, specifically because filenames can legally contain spaces,
newlines, or nearly any other byte except NUL and a path separator, meaning naive whitespace-delimited
parsing (the `xargs`/`find` default without `-0`) can silently mis-split filenames containing spaces
into multiple, incorrect arguments — a subtle but genuinely real correctness bug in scripts that
haven't adopted the NUL-delimited convention.

### Key commands
```
find . -name '*.log' -print0 | xargs -0 rm             # safe deletion, correctly handling filenames with spaces
find . -name '*.txt' | xargs -P 4 -I{} gzip {}            # parallelize an independent per-file operation across 4 workers
echo "a b c" | xargs -n1 echo                               # split input into individual invocations, one argument each
xargs --show-limits                                            # inspect the actual max-argument-length limit in effect
```

## Job Control (bg, fg, disown, nohup)

(The kernel-level process-group/session mechanics underlying job control are covered in Section 2;
this entry focuses on the interactive/scripting-level commands built on top of them.) An interactive
shell tracks every pipeline it launches as a numbered "job," displayed via `jobs`, and a job can be
running in the foreground (has the terminal's attention, receiving keyboard-generated signals like
Ctrl-C directly), running in the background (`command &`, or suspended via Ctrl-Z and then resumed in
the background with `bg`, continuing execution without terminal focus), or stopped (suspended via
Ctrl-Z, not running at all until explicitly resumed via `fg`/`bg`). `fg %N`/`bg %N` move a specific
numbered job into the foreground or background respectively, and `disown %N` removes a job from the
shell's own job table entirely without stopping it — critically, a disowned job is no longer
considered a child the shell needs to track, but it does *not* by itself prevent the job from
receiving `SIGHUP` if the shell's own session ends (closing the terminal); `disown -h %N` specifically
marks the job to be exempted from `SIGHUP` delivery specifically, without removing it from the job
table entirely, a subtly different and sometimes more appropriate variant depending on exactly which
behavior is actually desired. `nohup command &` is the more commonly-reached-for, simpler alternative
achieving a similar practical outcome for background job persistence beyond a terminal session's
lifetime: it explicitly makes the launched command ignore `SIGHUP` (redirecting its stdout/stderr to a
file named `nohup.out` by default if they'd otherwise be connected to a terminal, since a
disconnected terminal has nowhere for that output to go), letting it survive the terminal disconnecting
regardless of the shell's own job-table bookkeeping. For anything beyond a quick, one-off background
task meant to outlive an interactive session, a proper systemd service/timer unit (Section 7) is
overwhelmingly the more robust, production-appropriate choice over `nohup`/`disown`-based ad-hoc
backgrounding — those techniques are genuinely useful for interactive convenience but lack systemd's
restart policies, structured logging, and reliable cgroup-based process tracking for anything intended
to run unattended and reliably over the long term.

### Key commands
```
command &                            # launch directly in the background
jobs                                    # list current shell's tracked jobs
fg %1 / bg %1                             # bring job 1 to foreground / resume it in background
disown -h %1                                # exempt job 1 from SIGHUP without removing job-table tracking
nohup command &                               # simpler: ignore SIGHUP outright, redirecting output to nohup.out
```

## Writing Idempotent, Production-Grade Scripts

An idempotent script is one that produces the same correct end state regardless of how many times it's
run — a genuinely essential property for any automation intended to be safely re-run after a partial
failure, retried by an orchestration system, or executed repeatedly by configuration-management
tooling (Ansible, cloud-init, provisioning scripts) without accumulating incorrect duplicate state or
failing outright on a second invocation. Achieving idempotency in practice means favoring
"ensure state X" operations over "perform action Y" operations wherever an equivalent exists:
`mkdir -p` (succeeds silently whether or not the directory already exists) instead of a bare `mkdir`
(fails on the second run); checking whether a line already exists in a config file before appending it
again (`grep -qxF "$line" file || echo "$line" >> file`) rather than unconditionally appending, which
would otherwise duplicate the line on every re-run; using a package manager's "ensure installed"
semantics rather than an unconditional install command that might fail or behave unexpectedly against
an already-installed package. Beyond idempotency specifically, production-grade scripts consistently
apply several other disciplines covered throughout this section in combination: `set -euo pipefail` as
a baseline safety net; explicit, meaningful exit codes distinguishing different failure categories
(rather than every failure path exiting with a generic `1`, which makes it impossible for a caller/
orchestrator to distinguish "input validation failed" from "a dependency was unreachable" from "the
operation itself failed partway through" without parsing free-form error text); structured, timestamped
logging (to both stdout for interactive visibility and, ideally, a location a monitoring/log-
aggregation pipeline can also consume) rather than silent operation or unstructured, hard-to-parse
free-text output; `trap`-based cleanup guaranteeing temporary resources are always released regardless
of exit path; input validation at the script's own entry point (checking required arguments/environment
variables are actually present and sane *before* proceeding, rather than failing confusingly deep
inside the script's logic when a missing precondition is finally encountered); and, for scripts
performing genuinely destructive or hard-to-reverse operations, an explicit dry-run mode
(`--dry-run`, printing what *would* be done without actually doing it) letting an operator verify a
script's intended behavior safely before committing to its real execution against production. None of
these individual practices are complicated in isolation, but consistently applying all of them together
is precisely what separates a quick, ad-hoc script from genuinely production-grade automation trusted
to run unattended against critical infrastructure.

### Key commands
```
set -euo pipefail                     # standard safety-net header
mkdir -p /path/to/dir                   # idempotent directory creation
grep -qxF "$line" file || echo "$line" >> file   # idempotent "ensure line present" pattern
trap 'rm -f "$tmpfile"' EXIT               # guaranteed temp-resource cleanup
[[ "${1:-}" == "--dry-run" ]] && DRY_RUN=1   # simple dry-run flag support pattern
```

---

### Interview Questions

**Conceptual/Internals (8)**

1. **Why can a script declaring `#!/bin/sh` behave unexpectedly on some systems but not others?**
   `/bin/sh` is frequently symlinked to a minimal, strictly POSIX-compliant shell (`dash` on Debian/
   Ubuntu, `ash`/busybox on many minimal container base images) rather than `bash`, and any bash-
   specific extensions used in the script (arrays, `[[ ]]`, process substitution) simply don't exist
   or behave differently under those shells, causing failures or silently wrong behavior specifically
   on systems where `/bin/sh` isn't actually bash.

2. **Explain why `"${arr[@]}"` and `"${arr[*]}"` behave differently, and why the distinction matters.**
   `"${arr[@]}"` expands each array element as its own separate, individually-quoted word, correctly
   preserving elements containing embedded spaces as single arguments. `"${arr[*]}"` joins all elements
   into a single string separated by the first character of `$IFS`, losing the individual-element
   boundary information — using the wrong form when iterating over filenames or arguments that may
   contain spaces is a common, subtle correctness bug.

3. **Why does `command 2>&1 > file` not achieve the commonly-intended "redirect both stdout and
   stderr to file" behavior?**
   Redirection operators are processed left to right; `2>&1` first duplicates fd 2 onto whatever fd 1
   currently points to (the terminal, if this is the first redirection), and only afterward does
   `> file` redirect fd 1 itself to the file — leaving stderr still pointed at the terminal. The
   correct order is `> file 2>&1`, redirecting stdout to the file first, then duplicating stderr onto
   that same, already-redirected destination.

4. **What specifically does `set -o pipefail` change, and what problem does it solve?**
   By default, a pipeline's exit status is simply whatever its last command returned, regardless of any
   earlier stage's failure — `failing_command | grep pattern` reports success based on grep's exit
   status alone. `pipefail` changes the pipeline's overall exit status to the rightmost nonzero exit
   status among all stages, surfacing an earlier stage's failure that would otherwise be silently
   masked by a later, successfully-exiting stage.

5. **Why is `trap 'cleanup' EXIT` considered more robust than manually calling a cleanup function at
   every point a script might exit?**
   `EXIT` is a pseudo-signal bash fires whenever the shell is about to exit for any reason — normal
   completion, an explicit `exit` call, or an unhandled real signal — making one `trap ... EXIT`
   registration a comprehensive guarantee that doesn't require anticipating and instrumenting every
   individual exit path manually, which is both tedious and easy to miss for a new exit path
   introduced later during script maintenance.

6. **Why does a `while read line; do ...; done < <(command)` pattern avoid a subtlety that
   `command | while read line; do ...; done` has?**
   In the piped form, the `while` loop runs in a subshell (the right-hand side of a pipe), so any
   variables set inside the loop body do not persist back to the calling shell once the pipeline
   completes. Using process substitution instead (`< <(command)`) feeds the same data into the loop
   without placing the loop itself inside a subshell, so variables set inside it remain visible to the
   rest of the script afterward.

7. **What makes a script idempotent, and give a concrete example of a non-idempotent operation and
   its idempotent equivalent.**
   An idempotent script produces the same correct end state no matter how many times it's run. A bare
   `mkdir /path` is non-idempotent (fails if the directory already exists from a prior run); `mkdir -p
   /path` is the idempotent equivalent, succeeding silently whether or not the directory already
   exists.

8. **Why can PCRE-style backtracking regular expressions pose a denial-of-service risk that POSIX-
   style regex engines generally don't?**
   POSIX basic/extended regex engines typically use finite-automaton-based matching with guaranteed
   linear-time complexity regardless of pattern complexity. PCRE's backtracking approach supports
   richer features (lookahead, backreferences) but can exhibit catastrophic, exponential-time
   worst-case behavior for certain pathological pattern/input combinations, a genuine ReDoS risk when
   matching untrusted input against a naively-constructed backtracking regex.

**Scenario/Troubleshooting (6)**

9. **A deployment script using `set -e` continues executing past a failed command inside an `if`
    condition, contrary to the team's expectation.**
    This is `errexit`'s well-documented, correct-per-specification behavior, not a bug: a command's
    failure as part of an `if`/`while` condition (or as a non-final pipeline element without
    `pipefail`, or within `&&`/`||` chains) does not trigger `errexit`'s exit, since the shell
    reasonably assumes those contexts are deliberately testing/handling the failure. The fix is
    explicitly checking and handling the exit status within the conditional logic itself, rather than
    relying on `set -e` to catch it there.

10. **A script processing a directory of user-uploaded files fails or behaves incorrectly whenever a
    filename contains a space.**
    The script is very likely using unquoted variable expansion or whitespace-delimited `find`/`xargs`
    piping without `-print0`/`-0`, causing filenames with embedded spaces to be incorrectly split into
    multiple arguments. Fix by consistently quoting variable expansions (`"$file"`, not `$file`) and
    using NUL-delimited (`find ... -print0 | xargs -0 ...`) patterns throughout wherever filenames flow
    through the pipeline.

11. **A backup script wrapped with `nohup script.sh &` still gets killed when the initiating SSH
    session disconnects unexpectedly, despite `nohup` being used correctly.**
    Confirm whether the script itself spawns further child processes that don't inherit the same
    `SIGHUP`-ignoring disposition, or whether the session is being terminated by something other than
    `SIGHUP` (a more forceful termination of the whole session's process group by the SSH server/PAM
    session cleanup). For anything genuinely needing to survive independent of any interactive
    session's lifecycle at all, migrating to a proper systemd service/timer unit rather than
    `nohup`-based backgrounding is the more robust, recommended fix.

12. **A CI pipeline step using `xargs -P 4` to parallelize a batch operation occasionally produces
    corrupted output files, though the same operation works correctly when run sequentially.**
    Check whether the parallelized operations have any shared-state contention (writing to a shared
    intermediate file, appending to a shared log without appropriate locking) that only manifests under
    genuine concurrent execution — sequential execution masks race conditions that true parallelism
    exposes. Fix by ensuring each parallel invocation operates on genuinely independent output
    targets, or by adding explicit locking/synchronization if shared state is unavoidable.

13. **A production script that has run reliably for months suddenly fails with "argument list too
    long" after a directory it processes grew substantially.**
    The script is very likely passing an unbounded, directly-expanded glob (`rm *.log` or similar)
    directly as command-line arguments rather than piping a list through `xargs`, which automatically
    batches arguments within the system's actual maximum argument-list length limit. Fix by refactoring
    to `find ... -print0 | xargs -0 command`, which handles arbitrarily large lists correctly regardless
    of how many files are involved.

**FAANG-level Deep Dive (6)**

15. **Explain precisely why `set -e`'s behavior with pipelines requires `pipefail` to be meaningfully
    useful, and describe a scenario where even `set -euo pipefail` together still fails to catch a
    real error.**
    Without `pipefail`, `set -e` only observes a pipeline's overall (last-command) exit status, missing
    earlier-stage failures entirely; `pipefail` fixes this specific gap. However, even with all three
    flags set, a command substitution's own internal failure inside an otherwise-successful larger
    expression (`var=$(failing_command); other_command "$var"`, where `other_command` might still
    "succeed" operating on empty/garbage input from the failed substitution) is not automatically
    caught by `errexit` in every bash version/context reliably, since command substitution failure
    propagation historically has its own set of edge cases distinct from pipeline failure propagation —
    illustrating that `set -euo pipefail` substantially improves but does not create an unconditional,
    complete safety guarantee against every failure mode.

16. **Why does process substitution's `/dev/fd/N`-based implementation mean it cannot fully replace a
    real temporary file for every use case, even though it avoids creating one?**
    Process substitution's "file" is backed by an anonymous pipe, which supports only sequential
    reading (or writing) and has no genuine random-access seek capability the way a real, disk-backed
    temporary file does. A program that needs to `seek()` backward within the data it's given (some
    archive tools, certain database bulk-load utilities validating a file's structure by reading it
    multiple times or out of order) will fail or behave incorrectly when given a process-substitution
    pseudo-file instead of a genuine seekable file, which is precisely the scenario where a real
    temporary file (`mktemp`) remains necessary despite process substitution's general convenience and
    efficiency advantage for purely sequential consumers.

17. **Explain why `trap ... EXIT` cleanup logic itself needs to be written carefully to avoid masking
    the script's own real exit status.**
    If the trap handler itself executes a command whose own exit status differs from zero (even
    unintentionally, like a cleanup `rm` that fails because the file was already removed), and the trap
    handler doesn't explicitly preserve and re-exit with the original failing exit status the script
    was in the process of exiting with, the script's final observed exit status can become the trap
    handler's own (possibly successful) exit status instead of the original failure that triggered the
    exit in the first place — silently converting what should be a visible failure into an apparently
    successful script run. Correct practice captures `$?` at the very start of the trap handler and
    explicitly `exit`s with that preserved value at the handler's end, after performing cleanup.

18. **Why does `awk`'s automatic field-splitting model make it fundamentally better suited than `sed`
    for column-oriented data transformation, at a conceptual level?**
    `sed` operates purely on the line as an undifferentiated string, with no native concept of fields/
    columns at all — any column-aware logic must be hand-built via regular expressions matching
    delimiter positions, which becomes increasingly awkward and fragile as the required logic grows
    beyond simple substitution. `awk` automatically splits each input record into fields (based on a
    configurable field separator) as a first-class part of its execution model, exposing them as
    directly indexable variables (`$1`, `$2`, ...) with native arithmetic and comparison support,
    making genuinely column-aware logic (summing a column, comparing two fields, conditional logic
    spanning multiple fields) a natural, concise expression rather than requiring `sed`'s
  line-as-string workaround approach.

19. **Explain why `xargs -P N`'s parallelism model can produce a genuinely different (and sometimes
    incorrect) result compared to the equivalent sequential invocation, beyond simple raw execution
    speed.**
    Parallel invocations share no ordering guarantee relative to each other and may execute in
    overlapping time windows, meaning any operation with side effects that depend on execution order
    (appending to a shared log file without proper locking, incrementing a shared counter file,
    operations that assume a specific processing order for correctness) can produce genuinely different
    — and potentially incorrect or corrupted — results under real concurrency that a sequential
    execution's implicit ordering guarantee happened to mask, which is precisely why introducing
    parallelism to an existing sequential script requires explicitly verifying (not merely assuming)
    that each parallelized unit of work is genuinely independent with no hidden shared-state
    dependency.

20. **Why is checking `${VAR:-default}` different from `${VAR:=default}`, and why does the distinction
    matter for idempotent script design?**
    `${VAR:-default}` merely *substitutes* `default` in the expansion's result if `VAR` is unset or
    empty, without actually assigning `default` back into `VAR` itself — subsequent references to
    `$VAR` later in the script still see it as unset/empty. `${VAR:=default}` both substitutes *and*
    assigns `default` into `VAR` for the remainder of the script's execution, meaning subsequent
    references correctly see the now-defaulted value — using the wrong form (particularly `:-` when
    `:=` was actually needed) is a subtle bug where a script appears to correctly apply a default value
    once but then behaves as though the variable were still unset everywhere else it's referenced
    afterward, directly undermining a script's intended idempotent, predictable behavior across its
    full execution.

### Hands-On Labs

**Lab 1: Build a defensive, idempotent deployment script**
- Objective: Apply every production-grade scripting discipline covered in this section together.
- Setup: Any Linux shell environment.
- Tasks: Write a script that provisions a user, a directory, and a systemd service unit; make it
  idempotent (safe to re-run without error or duplication); add `set -euo pipefail`, a `trap`-based
  cleanup for any temp files used, input validation for required arguments, and a `--dry-run` mode.
- Expected outcome: A script verified safe to run repeatedly with identical, correct results each
  time, including a working dry-run mode.

**Lab 2: Diagnose and fix redirection-order and quoting bugs**
- Objective: Directly experience and fix the classic redirection-order and array-quoting mistakes.
- Setup: Any Linux shell.
- Tasks: Write a script with the `2>&1 > file` ordering mistake and confirm stderr still appears on the
  terminal; fix the order and confirm correct behavior; separately, write a script iterating over an
  array of filenames-with-spaces using unquoted `${arr[@]}` and observe the incorrect splitting, then
  fix it with `"${arr[@]}"`.
- Expected outcome: A documented before/after for both classic bugs, with root cause explained in your
  own words.

**Lab 3: Parallelize a batch operation safely with xargs**
- Objective: Correctly parallelize an embarrassingly-parallel task while avoiding shared-state hazards.
- Setup: A directory of test files.
- Tasks: Write a sequential script performing an independent per-file operation (e.g., computing a
  checksum and writing it to its own per-file output, not a shared log); convert it to use
  `xargs -P N`; verify identical, correct results at higher speed; then deliberately introduce a
  shared-log-append version and observe/document the resulting corruption under parallelism.
- Expected outcome: A working, correctly-parallelized version, plus a documented demonstration of the
  shared-state hazard it specifically avoided.

**Lab 4: trap-based guaranteed cleanup under multiple exit paths**
- Objective: Verify `trap ... EXIT` cleanup fires reliably regardless of how a script exits.
- Setup: Any Linux shell.
- Tasks: Write a script creating a temp file with `trap 'rm -f "$tmpfile"' EXIT`; test that the temp
  file is removed after normal completion, after an explicit early `exit 1`, and after being killed
  with `SIGTERM` mid-execution (not `SIGKILL`, which cannot be trapped).
- Expected outcome: Verified, documented cleanup across all three exit scenarios.

**Lab 5: POSIX portability audit**
- Objective: Practice distinguishing bash-specific constructs from POSIX-portable ones.
- Setup: A Linux system with both `bash` and `dash` installed.
- Tasks: Write a script using several bashisms (arrays, `[[ ]]`, process substitution); run it under
  `dash` and document each resulting failure; rewrite it to be strictly POSIX-compliant and confirm it
  now runs identically under both `bash` and `dash`.
- Expected outcome: A documented list of bashisms encountered and their POSIX-compliant replacements.

### Production Incidents

**Incident 1: A silent data-loss bug from an unquoted array expansion in a backup script**
- Symptom: A backup script intended to archive a list of directories (some containing spaces in their
  names) silently skips or mis-archives a subset of directories, discovered only when a restore was
  attempted and specific directories were missing.
- Investigation: Code review found the script iterated over an array of directory paths using
  unquoted `${dirs[@]}` rather than `"${dirs[@]}"`, causing directories with embedded spaces to be
  word-split into multiple, individually-incorrect path fragments.
- Root cause: The unquoted array expansion silently mishandled any directory name containing a space,
  with no error raised at the time — the script appeared to "work" for the common case of
  space-free directory names, masking the bug until a genuinely space-containing directory was
  eventually included.
- Recovery: Directories missed by previous backup runs could not be recovered from that backup
  mechanism and required restoration from an alternate, older backup source.
- Prevention: Fixed the quoting throughout the script, and added a linting step (`shellcheck`, which
  specifically flags this exact class of unquoted-expansion issue) to the script's CI validation
  pipeline to catch this category of bug automatically before future deployment.

**Incident 2: A masked pipeline failure caused a data pipeline to silently process incomplete data
for weeks**
- Symptom: A downstream analytics report is found to be subtly, consistently incomplete over several
  weeks, though the data pipeline's own logs and monitoring showed no failures during that entire
  period.
- Investigation: Found the pipeline's ingestion script piped a data-fetching command through a
  filtering/transformation stage (`fetch_data | transform | load_data`) without `set -o pipefail`; the
  fetch stage had begun intermittently failing (an upstream API change), but since `transform`/
  `load_data` continued to exit successfully on whatever partial/empty input they received, the
  pipeline's own exit-status-based success monitoring never detected any problem.
- Root cause: Without `pipefail`, an earlier pipeline stage's failure was completely invisible to the
  script's own success/failure determination, which relied solely on the final stage's exit status.
- Recovery: Backfilled the missing data once the upstream API issue was identified and fixed, requiring
  a fully separate remediation effort to reconstruct the several weeks of incomplete analytics.
- Prevention: Added `set -euo pipefail` to this and all similar pipeline scripts fleet-wide, and added
  explicit row-count/data-volume sanity checks as an additional, independent layer of pipeline health
  verification beyond pure exit-status monitoring alone.

**Incident 3: A non-idempotent provisioning script caused duplicate configuration after a retried
deployment**
- Symptom: After an automated deployment system retried a provisioning step following a transient
  network failure partway through its first attempt, several hosts ended up with duplicated
  configuration file entries and, in a few cases, duplicate cron jobs performing the same scheduled
  task twice.
- Investigation: The provisioning script unconditionally appended configuration lines and cron entries
  on every run rather than checking whether they already existed, meaning a retried run (necessary
  because the first attempt had partially completed before failing) reapplied the same appends a
  second time.
- Root cause: The script was written assuming it would only ever run exactly once per host, an
  assumption violated the moment any retry-on-failure automation was introduced around it.
- Recovery: Manually identified and de-duplicated the affected configuration and cron entries across
  impacted hosts.
- Prevention: Rewrote the provisioning script using idempotent patterns throughout (checking for
  existing entries before appending, using `mkdir -p` and equivalent "ensure state" operations
  consistently) and added an explicit idempotency test to the script's validation suite, running it
  twice in a row against a clean test host and asserting identical final state after both runs.
