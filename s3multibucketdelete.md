Below is a detailed and visually appealing `README.md` for your enhanced script, complete with diagrams, step-by-step explanations, and simple language so anyone can understand and use it confidently.

---

# 🚀 **Bulk S3 Bucket Deletion with TMUX Resilience**

### ✅ What This Tool Does

* **Safely deletes multiple S3 buckets**, one by one.
* Prompts **at every level**—folders, root objects, versions, delete markers, and final bucket.
* **Runs inside a TMUX session**, so even if your CloudShell times out, it keeps going.
* **Stops on error**, drops you into TMUX to inspect issues immediately.

---

## 🌟 Why This Matters

1. S3 can host **huge buckets**—naive scripts might fail or get timed out in CloudShell.
2. `tmux` ensures the session **persists even when disconnected** ([docs.aws.amazon.com][1], [reddit.com][2]).
3. Strict bash settings (`set -Eeuo pipefail`) combined with `trap ERR` provide **robust error detection and safe exit** ([howtogeek.com][3]).

---

📘 Overview
This script helps you securely delete one or more AWS S3 buckets, even large ones, without worrying about connection timeouts (like CloudShell). It runs inside a persistent tmux session, ensures every deletion is confirmed, and stops safely if there's an error.


🧩 How It Works — High-Level Flow


┌─────────────────┐    Launch tmux     ┌─────────────┐
│ You run script  │ ─────────────────► │ tmux session│
└─────────────────┘                    └─────────────┘
                                          │
                                          ▼
                        ┌─────────────────────────────┐
                        │ Ask for bucket names (list) │
                        └─────────────────────────────┘
                                          │
                 ┌────────────────────────┼────────────────────────┐
                 ▼                        ▼                        ▼
       ┌───────────────────┐    ┌──────────────────┐    ┌──────────────────┐
       │ Validate bucket A │    │ … bucket B …      │    │ … bucket N …      │
       └───────────────────┘    └──────────────────┘    └──────────────────┘
                 │                        │                        │
          Ask confirm             Ask confirm             Ask confirm
    ┌────────────┴─────────┐ ┌────────────┴─────────┐ ┌────────────┴─────────┐
    ▼                      ▼ ▼                      ▼ ▼                      ▼
 Confirm & delete       Skip                Confirm & delete       Skip
   prefixes, root,          ...                 ...
   versions, markers
                 │                        │                        │
                 └──────────┬─────────────┴─────────────┬──────────┘
                            ▼                           ▼
                  ┌──────────────────────────────────────────────┐
                  │ Final ask: delete the entire bucket?         │
                  └──────────────────────────────────────────────┘
                            │                           │
                    Confirm → Delete                Skip
                            └──────────┬─────────────────────┘
                                       ▼
                              💬 “All done!” at end



┐
│ Run script → Not in tmux?
│ → Spawn tmux session → Exit current shell
└──────────────────────────────────────────▶ Inside tmux:
    ↓
    Prompt for bucket names (until blank line)
    ↓
    For each bucket:
        Verify exists → Confirm proceed?
            • List folders → Confirm each → Delete if approved
            • List root objects → Confirm → Delete if approved
            • List versioned items & delete markers → Confirm each type → Delete if approved
            • FINAL confirmation → Delete bucket
    ↓
    If any error occurs → Trap catches it → tmux opens for you to inspect!
    ↓
    All done — “All done!” message displayed


🧩 Code Walkthrough
1. Strict Mode & Error Trap
bash
Copy
Edit
#!/usr/bin/env bash
set -Eeuo pipefail
trap 'on_error $? $LINENO' ERR
-e: Exit on any failing command

-u: Treat undefined variables as errors

-o pipefail: Catch failures inside pipelines 
reddit.com
+1
reddit.com
+1
stackoverflow.com
+2
howtogeek.com
+2
stackoverflow.com
+2

-E: Make sure ERR trap works in functions/subshells 
reddit.com
+15
stackoverflow.com
+15
stackoverflow.com
+15

trap ERR: Calls on_error() on any command failure

2. Error Handler
bash
Copy
Edit
on_error() {
  echo "❌ ERROR at line $2: exit code $1"
  tmux attach -t "$SESSION" || true
  exit "$1"
}
Automatically opens tmux so you can inspect what went wrong.

3. Tools Validation
bash
Copy
Edit
env_check() {
  for cmd in aws jq tmux; do
    command -v "$cmd" >/dev/null || {
      echo "❌ '$cmd' is missing. Please install it."
      exit 1
    }
  done
}
Ensures aws, jq, and tmux are available before doing anything.

4. main() — Entry Point & TMUX Launcher
bash
Copy
Edit
if [ -z "${TMUX:-}" ]; then
  tmux new-session -d -s "$SESSION" bash "$0" "$@"
  echo "📺 Connect later with: tmux attach -t $SESSION"
  exit
fi
If not already inside tmux, it restarts the script inside a new session and exits current shell.

5. Reading Buckets
bash
Copy
Edit
echo "🔹 Enter bucket names (one per line), blank to finish:"
while read -rp "> " name && [[ -n "$name" ]]; do
  buckets+=("$name")
done
Supports input of multiple bucket names interactively.

6. Secure Deletion per Bucket
bash
Copy
Edit
aws s3api head-bucket --bucket "$bucket"
confirm "Proceed deleting '$bucket'?" || return
Checks bucket existence and requests user confirmation before continuing.

7. Deleting Folders (Prefixes)
bash
Copy
Edit
mapfile -t prefixes < <(
  aws s3api list-objects-v2 --bucket "$bucket" --delimiter "/" \
    --query 'CommonPrefixes[].Prefix' --output text
)
for p in "${prefixes[@]}"; do
  confirm "Delete folder '$p'?" && aws s3 rm "s3://$bucket/$p" --recursive
done
Gets top-level folders and prompts deletion for each individually.

8. Deleting Root-level Objects
bash
Copy
Edit
root_objs=$(aws s3api list-objects-v2 --bucket "$bucket" ...)
if [[ -n "$root_objs" ]]; then
  echo "Preview:"; printf '%s\n' "$root_objs" | head -n5
  confirm "Delete all root objects?" && aws s3 rm "s3://$bucket/" --recursive
fi
Shows a preview and asks for permission before deleting all root-level files.

9. Handling Versioned & Delete-Marker Objects
bash
Copy
Edit
for typ in Versions DeleteMarkers; do
  items=$(aws s3api list-object-versions --bucket "$bucket" --query "${typ}[] | []")
  count=$(jq length <<<"$items")
  if (( count )); then
    echo "Found $count $typ"
    confirm "Delete all $typ?" && aws s3api delete-objects ...
  fi
done
Retrieves and prompts deletion of all versioned/deleted objects — if versioning is enabled.

10. Final Bucket Deletion
bash
Copy
Edit
confirm "🔥 FINAL: Delete bucket '$bucket' permanently?" \
  && aws s3api delete-bucket --bucket "$bucket"
Requires final approval before deleting the bucket itself.


Everything runs inside tmux, so even if your browser disconnects, the cleanup continues.

Each step (prefixes, root objects, versioned items, bucket itself) is confirmed before execution.

If any error happens, the script stops and drops you into tmux to investigate.


---


## 🧠 Flow Diagram

```
You run script ─► Checks if not in TMUX ─► Spawns TMUX session ─► Inside TMUX:
 └─► Prompts for bucket names ─► For each bucket:
       ♦ Validates bucket exists
       ♦ Confirms deletion of folders
       ♦ Confirms deletion of root-level objects
       ♦ Confirms deletion of versions/delete-markers
       ♦ Final confirmation to delete bucket
 └─► If error occurs: stops and drops you into TMUX for debugging
 └─► All buckets processed → "All done!"
```

---

## 🛠 Step-by-Step Setup

1. **Save** script as `bulk_delete_tmux_safe.sh`.
2. **Install tools** in CloudShell:

   ```bash
   sudo yum install -y tmux jq
   ```
3. **Make executable**:

   ```bash
   chmod +x bulk_delete_tmux_safe.sh
   ```
4. **Run it**:

   ```bash
   ./bulk_delete_tmux_safe.sh
   ```

   * It will launch inside a TMUX session automatically.
   * Reconnect with:

     ```bash
     tmux attach -t s3deleter_tmux
     ```
5. **Enter bucket names**, one per line, press **Enter** on an empty line.
6. **Answer prompts**—script won’t delete anything without your approval.
7. If an error occurs, it’ll show an error message and stay open in TMUX for inspection.

---

## 🔎 Explained by Code

```bash
#!/usr/bin/env bash
set -Eeuo pipefail
trap 'on_error $? $LINENO' ERR
```

* **Strict mode** and **error trap** ensure immediate detection and safe debugging ([reddit.com][4], [howtogeek.com][3]).

```bash
env_check() {
  for cmd in aws jq tmux; do
    command -v "$cmd" >/dev/null || { echo "❌ '$cmd' missing."; exit 1; }
  done
}
```

* Verifies required tools are installed before proceeding.

```bash
if [ -z "${TMUX:-}" ]; then
  tmux new-session -d -s "$SESSION" bash "$0" "$@"
  exit 0
fi
```

* If not in TMUX, it **re-launches itself inside a session**, preserving the run environment ([reddit.com][2]).

```bash
delete_bucket_safe() {
  aws s3api head-bucket --bucket "$bucket"
  confirm "Proceed deleting bucket '$bucket'?"
  # ... folder deletion prompts ...
  confirm "🔥 FINAL: Delete bucket '$bucket'?"
  aws s3api delete-bucket --bucket "$bucket"
}
```

* Nested confirmations for each deletion step, ensuring **no accidental deletions**.

```bash
on_error() {
  echo "Error at line $2 (exit code $1)"
  tmux attach -t "$SESSION" || true
  exit "$1"
}
```

* Displays error info and automatically **opens TMUX** for diagnostics without leaving you hanging.

---

## 💡 Tips & Variations

* **Redirect input** from a file:

  ```bash
  ./bulk_delete_tmux_safe.sh < buckets.txt
  ```
* **Enable logging**:

  ```bash
  tmux attach -t s3deleter_tmux
  script -f delete.log
  ```
* **Dry-run**: Add `echo` before `aws` commands to preview actions.
* **Add SIGINT trap** to gracefully handle Ctrl+C if needed.

---

## 🎓 Summary

This tool is:

* 💪 **Robust**: survives errors and CloudShell disconnections.
* 🔐 **Safe**: always asks before deleting anything.
* 👥 **User-friendly**: simple prompts, clear messages, easy debugging.
* 🔄 **Flexible**: handles multiple buckets in one go.

Perfect for anyone doing large-scale AWS S3 cleanups without worrying about session limits or accidental deletions. Let me know if you'd like diagrams in PNG format or a visual flowchart!

[1]: https://docs.aws.amazon.com/cloudshell/latest/userguide/customizing-cshell.html?utm_source=chatgpt.com "Customizing your AWS CloudShell experience - AWS CloudShell"
[2]: https://www.reddit.com/r/HowToHack/comments/1i776tk?utm_source=chatgpt.com "what is the difference between opening a new terminal and using tmux to start a new session?"
[3]: https://www.howtogeek.com/821320/how-to-trap-errors-in-bash-scripts-on-linux/?utm_source=chatgpt.com "How to Trap Errors in Bash Scripts on Linux"
[4]: https://www.reddit.com/r/aws/comments/kdp5k2?utm_source=chatgpt.com "AWS CloudShell – Command-Line Access to AWS Resources"
