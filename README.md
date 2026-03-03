# list of git aliases
open gitconfig: `git config --global --edit`

```
edit = config --global --edit

restart = "!f() { \
  branch=${1:-main}; \
  echo \"WARNING: This will discard all local changes on $branch.\"; \
  read -p 'Are you sure you want to continue? (y/N) ' confirm; \
  if [ \"$confirm\" = \"y\" ]; then \
      git fetch && git switch $branch && git reset origin/$branch --hard; \
  else \
      echo 'Aborted.'; \
  fi; \
}; f \"$@\""

refresh = "!f() { \
  echo fetching...; \
  git fetch; \
  if [ $? -eq 0 ]; then \
    last_status=$(git status --untracked-files=no --porcelain); \
    if [ \"$last_status\" != \"\" ]; then \
      echo stashing local changes...; \
      git stash; \
    else \
      echo nothing to stash...; \
    fi;\
    if [ $? -eq 0 ]; then \
      echo rebasing...;\
      git rebase;\
      if [ $? -eq 0 ]; then \
        if [ \"$last_status\" != \"\" ]; then\
          echo applying stashed changes...;\
          git stash pop;\
          if [ $? -ne 0 ]; then \
            echo STASH POP FAIL - you will need to resolve merge conflicts with git mergetool; \
          fi; \
        fi; \
      else \
        echo REBASE FAILED - you will need to manually run stash pop; \
      fi;\
    fi;\
  fi; \
  if [ $? -ne 0 ]; then \
    echo ERROR: Operation failed; \
  fi; \
}; f \"$@\""

patch = "!f() { \
  branch=$(git rev-parse --abbrev-ref HEAD); \
  if ! git diff --quiet || ! git diff --cached --quiet; then \
    echo 'You have uncommitted changes. Please commit or stash them first.'; \
    exit 1; \
  fi; \
  if [ -z \"$1\" ]; then \
    count=2; \
    git fetch && git rebase -i HEAD~$count; \
  else \
    case \"$1\" in \
      *[!0-9]* ) \
        onto_branch=\"$1\"; \
        if [ -n \"$2\" ]; then \
          case \"$2\" in \
            *[!0-9]* ) \
              git fetch origin \"$onto_branch\" && git rebase origin/\"$onto_branch\"; \
              ;; \
            * ) \
              count=\"$2\"; \
              git fetch origin \"$onto_branch\" && git rebase origin/\"$onto_branch\" && git rebase -i HEAD~$count; \
              ;; \
          esac; \
        else \
          git fetch origin \"$onto_branch\" && git rebase origin/\"$onto_branch\"; \
        fi; \
        ;; \
      * ) \
        count=\"$1\"; \
        git fetch && git rebase -i HEAD~$count; \
        ;; \
    esac; \
  fi; \
  read -p \"Force-push to $branch? (y/N) \" confirm; \
  if [ \"$confirm\" = \"y\" ]; then \
    git push origin \"$branch\" --force-with-lease; \
  else \
    echo \"Aborted.\"; \
  fi; \
}; f \"$@\""

out = "!f() {	\
  git checkout -b ${1}; \
}; f \"$@\""

fix = "!f() { \
  git reflog; \
  printf '\\n'; \
  if [ -n \"$1\" ]; then \
    sel=\"$1\"; \
  else \
    printf 'Reset to HEAD@: '; \
    IFS= read -r sel; \
  fi; \
  if [ -z \"${sel}\" ]; then \
    echo 'Aborted: no selection'; \
    return 0; \
  fi; \
  case \"${sel}\" in \
    ''|*[!0-9]*) echo 'Aborted: selection must be a number'; return 1 ;; \
    *) ;; \
  esac; \
  target=\"HEAD@{${sel}}\"; \
  echo \"Resetting to ${target}\"; \
  git reset --hard \"${target}\"; \
}; f \"$@\""

this = "!f() { \
  if [ -n \"$GIT_EDITOR\" ]; then EDITOR_CMD=\"$GIT_EDITOR\"; \
  elif [ -n \"$EDITOR\" ]; then EDITOR_CMD=\"$EDITOR\"; \
  else EDITOR_CMD=$(git var GIT_EDITOR 2>/dev/null); \
    if [ -z \"$EDITOR_CMD\" ]; then EDITOR_CMD=\"notepad\"; fi; \
  fi; \
  echo \"hint: Waiting for your editor to close the file...\"; \
  \
  if command -v mktemp >/dev/null 2>&1; then \
    tmp=$(mktemp /tmp/staged-changes-XXXXXX); \
  else \
    tmp=\"${TMPDIR:-/tmp}/staged-changes-$$\"; \
  fi; \
  \
  { \
    echo \"> Hint: [x] to stage, [ ] to unstage, [s] to stash, [D] to discard\"; \
    echo \"\"; \
    echo \"> Staged Changes:\"; \
    git status --porcelain | grep -E '^[AMDRC]' | while IFS= read -r line; do \
      file=$(printf \"%s\" \"$line\" | cut -c4-); \
      printf \"[x] %s\n\" \"$file\"; \
    done; \
    echo \"\"; \
    echo \"> Unstaged Changes:\"; \
    git status --porcelain | grep -E '^.[AMDRC?]' | while IFS= read -r line; do \
      file=$(printf \"%s\" \"$line\" | cut -c4-); \
      printf \"[ ] %s\n\" \"$file\"; \
    done; \
  } > \"$tmp\"; \
  \
  eval "$EDITOR_CMD \"$tmp\""; \
  \
  stash_files=(); \
  \
  while IFS= read -r line; do \
    mark=$(printf \"%s\" \"$line\" | cut -c2); \
    file=$(printf \"%s\" \"$line\" | cut -c5-); \
    if [ \"$mark\" = \"x\" ]; then \
      git add -- \"$file\"; \
    elif [ \"$mark\" = \"s\" ]; then \
      stash_files+=(\"$file\"); \
    elif [ \"$mark\" = \"D\" ]; then \
      git restore --staged -- \"$file\" 2>/dev/null; \
      git restore -- \"$file\" 2>/dev/null; \
    else \
      git restore --staged -- \"$file\" 2>/dev/null; \
    fi; \
  done < \"$tmp\"; \
  \
  if [ ${#stash_files[@]} -gt 0 ]; then \
    git stash push -m \"Stashed Changes $(date +%Y-%m-%d)\" -- \"${stash_files[@]}\"; \
  fi; \
  \
  rm -f \"$tmp\"; \
}; f"

publish = "!f() { \
  b=$(git rev-parse --abbrev-ref HEAD); \
  if [ \"$b\" = main ]; then \
    echo 'Push blocked: use git push to update main directly'; \
    exit 1; \
  fi; \
  git push origin HEAD:$b; \
  git branch -u origin/$b; \
}; f"

pop = stash pop
continue = rebase --continue
undo = reset --soft HEAD~1
here = rev-parse --abbrev-ref HEAD
```
