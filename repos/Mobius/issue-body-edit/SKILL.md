---
name: issue-body-edit
description: Change the body of an existing task issue with exact text replacements. Use it when you update the Today, Change, Limits or Done section of a task issue.
metadata:
  roles: lead
---

# Change the body of a task issue

Do not type the full body again. A full new body can lose text that you did not read. Replace only the exact texts that change.

1. Write the current body into a new temporary directory:

   ```sh
   d=$(mktemp -d)
   gh issue view <number> -R <owner>/<repo> --json body -q .body > "$d/b.md"
   cp "$d/b.md" "$d/old.md"
   echo "$d"
   ```

2. Replace each old text with its new text. The script stops when an old text does not occur exactly one time. Then nothing changes. Use a triple-quoted string for a text with a line break or a quote.

   ```sh
   python3 -I - "$d/b.md" <<'EOF'
   import sys
   p = sys.argv[1]
   s = open(p).read()
   reps = [
       ("<old text 1>", "<new text 1>"),
       ("<old text 2>", "<new text 2>"),
   ]
   for a, b in reps:
       assert s.count(a) == 1, a[:60]
       s = s.replace(a, b)
   open(p, "w").write(s)
   EOF
   ```

3. Read the difference. Make sure that it has only the changes that you want:

   ```sh
   diff "$d/old.md" "$d/b.md"
   ```

4. Write the new body, and read it again from GitHub:

   ```sh
   gh issue edit <number> -R <owner>/<repo> --body-file "$d/b.md"
   gh issue view <number> -R <owner>/<repo> --json body -q .body | grep -n '<a word of the new text>'
   ```

Each Bash call has a new shell. When you do the steps in more than one call, set `d` again to the directory that step 1 showed.
