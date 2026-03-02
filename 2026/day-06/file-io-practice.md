# Day 06 – Linux File I/O Practice Notes

## What I Practiced

Reading and writing text files using fundamental Linux commands.

---

## Commands I Ran

### 1. Create an empty file
```bash
touch notes.txt
```
**What it does:** Creates an empty file named `notes.txt`. If the file already exists, it just updates the timestamp — it never overwrites content.

---

### 2. Write the first line (overwrite with `>`)
```bash
echo "Learning Linux file I/O - Day 06" > notes.txt
```
**What it does:** The `>` operator redirects output into a file. If the file has existing content, it **overwrites** it entirely. Use with caution!

---

### 3. Append more lines (with `>>`)
```bash
echo "DevOps engineers read logs and configs daily" >> notes.txt
echo "Mastering redirection makes you faster" >> notes.txt
echo "Use > to overwrite a file" >> notes.txt
echo "Use >> to append without erasing" >> notes.txt
echo "cat reads the full file at once" >> notes.txt
echo "head and tail read parts of a file" >> notes.txt
```
**What it does:** The `>>` operator **appends** to the end of the file without erasing existing content. Safe to use for adding log entries or new lines.

---

### 4. Write and display at the same time using `tee`
```bash
echo "tee writes to stdout AND a file simultaneously" | tee -a notes.txt
```
**What it does:** `tee` reads from stdin and writes to **both** the terminal (stdout) and the file at the same time. The `-a` flag makes it append instead of overwrite. Great for scripts where you want to log AND see output live.

---

### 5. Read the full file with `cat`
```bash
cat notes.txt
```
**Output:**
```
Learning Linux file I/O - Day 06
DevOps engineers read logs and configs daily
Mastering redirection makes you faster
Use > to overwrite a file
Use >> to append without erasing
cat reads the full file at once
head and tail read parts of a file
tee writes to stdout AND a file simultaneously
```
**What it does:** Prints the entire file to the terminal. Best for small files. For large log files, use `less` or `tail` instead.

---

### 6. Read only the first few lines with `head`
```bash
head -n 3 notes.txt
```
**Output:**
```
Learning Linux file I/O - Day 06
DevOps engineers read logs and configs daily
Mastering redirection makes you faster
```
**What it does:** Shows the first N lines. Default is 10 lines. Useful for checking the beginning of log files or config headers.

---

### 7. Read only the last few lines with `tail`
```bash
tail -n 3 notes.txt
```
**Output:**
```
cat reads the full file at once
head and tail read parts of a file
tee writes to stdout AND a file simultaneously
```
**What it does:** Shows the last N lines. Default is 10 lines. **This is the most used command in DevOps** — `tail -f logfile.log` follows a file in real time as new lines are written.

---

## Summary Table

| Command | Purpose | Common Use |
|---------|---------|------------|
| `touch file` | Create empty file | Start a new log/config |
| `echo "..." > file` | Write/overwrite file | Reset a file with new content |
| `echo "..." >> file` | Append to file | Add entries without erasing |
| `tee -a file` | Write + display at once | Scripts with live + logged output |
| `cat file` | Read full file | Small files, quick checks |
| `head -n N file` | Read first N lines | Check file headers |
| `tail -n N file` | Read last N lines | Check recent log entries |
| `tail -f file` | Follow file in real time | **Live log monitoring** |

---

## Why This Matters for DevOps

Every day in DevOps involves:
- Reading **application logs** → `tail -f /var/log/app.log`
- Updating **config files** → `echo "key=value" >> config.env`
- Writing **deployment scripts** that log output → `./deploy.sh | tee -a deploy.log`
- Checking **the last error** in a file → `tail -n 50 error.log`

These 7 commands cover 80% of file operations you'll do on the job.

---

## One Command I'll Use Most

```bash
tail -f /var/log/syslog
```
Real-time log following is essential for debugging live systems. Once you learn `tail -f`, you use it constantly.

---

*Day 06 of #90DaysOfDevOps | #DevOpsKaJosh | #TrainWithShubham*
