Here's a detailed `README.md` file that explains the purpose and every section of your `s3deleter_tmux` bash script:

---

# 🧹 `s3deleter_tmux` - Safe and Interactive AWS S3 Bucket Deletion Script

This script is designed to **safely delete AWS S3 buckets** with **multiple interactive confirmations**, **versioning support**, and **tmux-based session logging**. It prevents accidental deletion by showing previews and asking for confirmation before each critical operation.

---

## 🔧 Requirements

Before running this script, ensure the following tools are installed:

* `aws` CLI (configured with appropriate credentials)
* `jq` (for JSON parsing)
* `tmux` (for session handling)

---

## 🚀 How to Use

```bash
chmod +x s3deleter_tmux.sh
./s3deleter_tmux.sh
```

You'll be prompted to enter one or more bucket names and will be guided through an interactive deletion process in a `tmux` session.

---

## 📜 Script Breakdown

### 1. **Shebang and Safety Flags**

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
```

* Uses `bash` with:

  * `-E`: trap ERR in functions
  * `-e`: exit on error
  * `-u`: unset variable error
  * `-o pipefail`: fail if any command in pipeline fails

---

### 2. **Trap for Error Handling**

```bash
trap 'on_error $? $LINENO' ERR
```

* Catches errors and runs `on_error` with exit code and line number.

---

### 3. **Variables**

```bash
SESSION="s3deleter_tmux"
LOGFILE="$HOME/${SESSION}_$(date +%Y%m%d_%H%M%S).log"
```

* `SESSION`: tmux session name
* `LOGFILE`: stores script output logs with timestamps

---

### 4. **Error Handler Function**

```bash
on_error() {
  ...
}
```

* Displays error message with exit code and line number.
* Attaches to the tmux session for debugging.
* Exits the script.

---

### 5. **User Confirmation Prompt**

```bash
confirm() {
  ...
}
```

* Prompts the user with `y/n`
* Times out after 120 seconds
* Used throughout the script before every delete action

---

### 6. **Environment Check**

```bash
env_check() {
  ...
}
```

* Ensures required commands (`aws`, `jq`, `tmux`) are available.
* Exits with an error if any tool is missing.

---

### 7. **Safe Bucket Deletion Function**

```bash
delete_bucket_safe() {
  ...
}
```

**Steps it performs:**

* 🟡 Check if bucket exists
* ❓ Ask confirmation before proceeding
* 📁 Lists all "folders" (prefixes) and deletes them with confirmation
* 🗂️ Lists and optionally deletes root-level objects
* 🔁 Handles versioned buckets: delete object versions and delete markers
* 🔥 Asks for **final confirmation** before deleting the bucket itself

---

### 8. **Main Function**

```bash
main() {
  ...
}
```

* Calls `env_check`
* Handles launching into a `tmux` session (if not already in one)
* Prompts user for bucket names
* Loops through all given buckets and calls `delete_bucket_safe`

---

### 9. **Script Execution**

```bash
main "$@"
```

* Entry point to the script, passes any arguments

---

## 🧪 Sample Run

```
📥 Enter bucket names (one per line). Blank line to finish:
> test-bucket-123
> another-bucket
>
🔄 Starting bucket: test-bucket-123
Proceed deleting bucket 'test-bucket-123'? (y/n): y
  Delete folder 'logs/'? (y/n): y
    ✅ Deleted logs/
  Delete all root-level objects? (y/n): y
    ✅ Root objects deleted
  Found 3 Versions
  example1.txt
  example2.txt
  Delete all Versions? (y/n): y
    ✅ Deleted Versions
🔥 FINAL: Delete bucket 'test-bucket-123'? (y/n): y
    ✅ Bucket deleted
✔️ Done with test-bucket-123
```

---

## 🛡️ Safety Features

* Multiple user confirmations
* Timeout if user is idle (2 mins)
* Graceful error handling
* Runs in `tmux` to allow pause/resume
* Logging everything to a timestamped file

---

## 🧼 Cleanup

To close the tmux session after completion:

```bash
tmux kill-session -t s3deleter_tmux
```

---

## 📂 Logs

Logs are stored in your home directory:

```
~/s3deleter_tmux_YYYYMMDD_HHMMSS.log
```

---

## 👨‍💻 Author

This script is ideal for DevOps engineers and cloud administrators looking to manage S3 cleanup operations safely and interactively.

---

Let me know if you want the same script converted to Python or with extra features like:

* Dry-run mode
* Email notification
* Multi-region support

---
