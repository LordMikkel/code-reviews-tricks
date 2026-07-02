# Review Cheat Sheet 🧾

Simple, fast, and a few 'hacker' tricks for code review 🧑‍💻

<p align="center"> <img src="img.png" alt="alt text" /> </p>

## Setup ⚙️

Set these variables once:

```bash
PROGRAM="program"
SRC="src"
```

## Format 🤖

```bash
norminette .
```

## Build and Inspect 🛠️

Dry-run the build to see commands without executing:

```bash
make -n
```

Bad practice detector:

```bash
scan-build make
```

or

```bash
make CC="gcc -fanalyzer -Wall -Wextra -Werror"
```

Show undefined symbols (helps spot missing/forbidden calls):

```bash
nm -u ./$PROGRAM
```

Show all global variables:

```bash
nm ./$PROGRAM | grep -E " [DBC] "
```

Show dependencies:

```bash
ldd ./$PROGRAM
```

## Security Check 🔒

Show password or secrets inside binary:

```bash
strings ./$PROGRAM | grep -E "pass|passwd|password|shadow|secret|ssh|key|API|token"
```

Analysis of the level of hardening:

```bash
checksec file ./$PROGRAM
```

Analysis of known vulnerabilities:

```bash
opengrep scan --config p/c
```

## Grep for Allocations and Error Controls 🔎

Find hot memory functions fast:

```bash
grep -E "(malloc|calloc|realloc|free|strdup|strjoin|memcpy|memmove)" -R $SRC -n -C 5
```

## Memory and Leaks 🧠

Build with AddressSanitizer (ASan):

```bash
make CC="gcc -Wall -Wextra -Werror -g3 -fsanitize=address"
```

Run with Valgrind:

```bash
valgrind \
	--leak-check=full \
	--show-leak-kinds=all \
	--track-origins=yes \
	--errors-for-leak-kinds=all \
	--error-exitcode=1 \
	./$PROGRAM
```

Convenience Alias:

```bash
alias all_leaks="valgrind \
	--leak-check=full \
	--show-leak-kinds=all \
	--track-origins=yes \
	--track-fds=yes \
	--errors-for-leak-kinds=all \
	--error-exitcode=1"
```

Discover invisible Leaks:

```bash
# Terminal 1
./$PROGRAM
```

```bash
# Terminal 2
htop -p $(pidof $PROGRAM)
```

Then inside htop:
- `F4` → filter by name
- Watch **RES** column — if it grows over time → leak

Another way could be:

```bash
# Terminal 1
./$PROGRAM
```

```bash
# Terminal 2
PID=$(pidof $PROGRAM)
while kill -0 $PID 2>/dev/null; do
  awk '/VmRSS/{print $2" KB"}' /proc/$PID/status
  sleep 0.5
done
```

## Threads 🧵

Find race conditions:

```bash
make CC="gcc -Wall -Wextra -Werror -g3 -fsanitize=thread"
```

## Resource Limits ⏱️

Show peak memory usage for Valgrind and the program:

```bash
cat /proc/$(pidof valgrind)/status
```

Limit virtual memory to force edge cases:

```bash
(ulimit -v 200000; all_leaks ./$PROGRAM)
```

Limit CPU time to catch hangs:

```bash
ulimit -t 2; ./$PROGRAM
```

## Tracing 🔬

Trace live-syscalls:

```bash
strace ./$PROGRAM
```

Summarize syscalls:

```bash
strace -c ./$PROGRAM
```

## Performance ⚡

Record performance of the program to understand which functions could be problematic (run your program, interact, and exit):

```bash
perf record --call-graph dwarf ./$PROGRAM
perf report --dsos $PROGRAM
```

## Heavy inputs 🧨

Random input:

```bash
./$PROGRAM < /dev/urandom
```

```bash
head -c 4096 /dev/urandom | ./$PROGRAM
```

Huge input:

```bash
yes "A" | head -c 5M | ./$PROGRAM
```

Single massive line:

```bash
python3 -c "print('A' * 100000)" | ./$PROGRAM
```

Multi line:

```bash
printf '\n%.0s' {1..10000} | ./$PROGRAM
```

Null bytes:

```bash
printf 'hello\x00world\n' | ./$PROGRAM
```

Many small lines fast:

```bash
seq 1 200000 | ./$PROGRAM
```
