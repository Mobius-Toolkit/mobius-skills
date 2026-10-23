---
name: pull-request-screenshots
description: Read the UI screenshots of a pull request of Mobius-Toolkit/Mobius and compare each changed screen with main. Use it at the review of a pull request that changes `web/`, before you approve it.
metadata:
  roles: lead
---

# Screenshots of a pull request

The workflow `Screenshots` keeps the images on the branch `screenshots` of Mobius-Toolkit/Mobius. The folder `main/` has the images of the last Release run of `main` with the artifact `screenshots`. The folder `pr-<n>/` has the images of the newest head of pull request `<n>`. The workflow writes a section between `<!-- screenshots -->` and `<!-- /screenshots -->` in the pull request body:

- a table with one row for each screen that is `added`, `changed` or `removed`. The image links point to one commit of the branch `screenshots`.
- "No screen changed." when each image is the same as the image of `main`.
- "The screenshots are pending." or "The screenshots failed." when the images of the head are not ready.

The image of `main` comes from the newest `main`, not from the base of the pull request. Thus a screen can show as `changed` because of a newer commit on `main`. Before you blame the pull request, read the commits of `main` after the merge base.

Each Bash call has a new shell. Run the commands of one section in one Bash call, or set `P` and `D` again.

## Download the images

```sh
P=<pull request>
D=$(mktemp -d)
gh pr view $P -R Mobius-Toolkit/Mobius --json body --jq .body | sed -n '/<!-- screenshots -->/,/<!-- \/screenshots -->/p' > $D/section.md
grep -oE '`[a-z0-9-]+` \| (added|changed|removed)' $D/section.md
grep -oE 'https://raw\.githubusercontent\.com/[^"]+\.png' $D/section.md | while read u; do curl -sfL -o "$D/$(echo "$u" | awk -F/ '{print $(NF-1)"_"$NF}')" "$u"; done
echo $D; ls $D
```

The files have the names `main_<screen>.png` and `pr-<n>_<screen>.png`. A `removed` screen has no `pr-<n>` file, and an `added` screen has no `main` file.

When the section tells that the screenshots are pending, download the CI artifact of the head instead. The artifact has only the images of the pull request:

```sh
H=$(gh pr view $P -R Mobius-Toolkit/Mobius --json headRefOid --jq .headRefOid)
gh run list -R Mobius-Toolkit/Mobius --commit $H --workflow CI --json databaseId,conclusion
gh run download <databaseId> -R Mobius-Toolkit/Mobius -n screenshots -D $D/shots
```

## Compare a screen

```sh
n=<screen>
magick compare -metric AE $D/main_$n.png $D/pr-${P}_$n.png $D/diff_$n.png; echo
magick $D/main_$n.png $D/pr-${P}_$n.png +append $D/side_$n.png
```

The first number of `compare` is the count of changed pixels. `compare` exits with 1 when the images are different. Then read `side_<screen>.png` and `diff_<screen>.png` with the tool `Read`. Read the `-desktop` and the `-phone` image of each screen that the pull request changes.

A screen with a few changed pixels and no change in the code of that screen can be an unstable screenshot. Search the open issues for its name before you report it.
