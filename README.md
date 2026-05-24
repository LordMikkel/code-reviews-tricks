# Review Cheat Sheet

Simple, fast, and a bit "hacker" tricks for code-reviewing

## Format

```bash
norminette .
```

## Build and Inspect

Dry-run the build to see commands without executing:

```bash
make -n
```

Show undefined symbols (helps spot missing/forbidden calls):

```bash
nm -u ./program
```

Show all global variables:

```bash
nm ./program | /bin/grep -E " [DBC] "
```

Show password or secrets inside binary:

```bash
strings ./program | grep -E "pass|passwd|password|shadow|secret|ssh|key|API|token"
```

Show dependencies:

```bash
ldd ./program
```


## Grep for Allocations and Error Controls

Find hot memory functions fast:

```bash
grep -E "(malloc|calloc|realloc|free|strdup|strjoin|memcpy|memmove)" -R src/ -n -C 5
```

## Memory and Leaks

Build with AddressSanitizer (ASan):

```bash
make CC="gcc -Wall -Wextra -Werror -g3 -fsanitize=address"
```

Run with ASan:

```bash
./program
```

Valgrind

```bash
valgrind \
	--leak-check=full \
	--show-leak-kinds=all \
	--track-origins=yes \
	--errors-for-leak-kinds=all \
	--error-exitcode=1 \
	./program
```

Invisible Leaks

```bash
# Terminal 1
./program
```

```bash
# Terminal 2
htop -p $(pidof program)
```

Then inside htop:
- `F4` → filter by name
- Watch **RES** column — if it grows over time → leak

other way could be

```bash
# Terminal 1
./program
```

```bash
# Terminal 2
PID=$(pidof program)
while kill -0 $PID 2>/dev/null; do
  awk '/VmRSS/{print $2" KB"}' /proc/$PID/status
  sleep 0.5
done
```

## Resource Limits

Peek at valgrind memory process stats while running:

```bash
cat /proc/$(pidof valgrind)/status
```

Limit virtual memory to force edge cases:

```bash
(ulimit -v 200000; all_leaks ./program)
```

Limit CPU time to catch hangs:

```bash
ulimit -t 2; ./program
```

## Tracing

Trace live-syscalls:

```bash
strace ./program
```

Trace resume syscalls:

```bash
strace -c ./program
```

## Performance

Record performance of the program to understand which functions could be problematic (run your program, interact, and exit):

```bash
perf record --call-graph dwarf ./program
perf report --dsos program
```

## Heavy inputs

Random input:

```bash
./program < /dev/urandom
```

```bash
head -c 4096 /dev/urandom | ./program
```

Huge input:

```bash
yes "A" | head -c 5M | ./program
```

Single massive line:

```bash
python3 -c "print('A' * 100000)" | ./program
```

Null bytes

```bash
 printf 'hello\x00world\n' | ./program
```

Many small lines fast:

```bash
seq 1 200000 | ./program
```
