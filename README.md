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
	cd -- \"$(git rev-parse --show-toplevel)\" || return 1; \
	\
	status=$(git status --porcelain); \
	if [ -z \"$status\" ]; then \
		echo 'Nothing to do -- working tree clean.'; \
		return 0; \
	fi; \
	\
	if [ -n \"$GIT_EDITOR\" ]; then EDITOR_CMD=\"$GIT_EDITOR\"; \
	elif [ -n \"$EDITOR\" ]; then EDITOR_CMD=\"$EDITOR\"; \
	else EDITOR_CMD=$(git var GIT_EDITOR 2>/dev/null); \
		if [ -z \"$EDITOR_CMD\" ]; then EDITOR_CMD=\"notepad\"; fi; \
	fi; \
	echo \"hint: Waiting for your editor to close the file...\"; \
	\
	if command -v mktemp >/dev/null 2>&1; then \
		tmp=$(mktemp \"${TMPDIR:-/tmp}/git-this-XXXXXX\"); \
		stash_list=$(mktemp \"${TMPDIR:-/tmp}/git-this-stash-XXXXXX\"); \
	else \
		tmp=\"${TMPDIR:-/tmp}/git-this-$$\"; \
		stash_list=\"${TMPDIR:-/tmp}/git-this-stash-$$\"; \
	fi; \
	trap 'rm -f \"$tmp\" \"$stash_list\"' EXIT INT TERM; \
	: > \"$stash_list\"; \
	\
	{ \
		echo '# [x] stage  [ ] unstage  [s] stash  [D] discard'; \
		echo '# Lines starting with # are ignored.'; \
		echo ''; \
		echo '# Staged Changes:'; \
		printf '%s\\n' \"$status\" | grep '^[AMDRTC]' | while IFS= read -r line; do \
			file=$(printf '%s' \"$line\" | cut -c4-); \
			case \"$file\" in *' -> '*) file=${file##* -> };; esac; \
			printf '[x] %s\\n' \"$file\"; \
		done; \
		echo ''; \
		echo '# Unstaged Changes:'; \
		printf '%s\\n' \"$status\" | grep '^.[MDRTC]' | while IFS= read -r line; do \
			file=$(printf '%s' \"$line\" | cut -c4-); \
			case \"$file\" in *' -> '*) file=${file##* -> };; esac; \
			printf '[ ] %s\\n' \"$file\"; \
		done; \
		echo ''; \
		echo '# Untracked Files:'; \
		printf '%s\\n' \"$status\" | grep '^??' | while IFS= read -r line; do \
			file=$(printf '%s' \"$line\" | cut -c4-); \
			printf '[ ] %s\\n' \"$file\"; \
		done; \
	} > \"$tmp\"; \
	\
	eval \"$EDITOR_CMD \\\"$tmp\\\"\"; \
	if [ $? -ne 0 ]; then \
		echo 'Editor exited with an error -- aborting.'; \
		return 1; \
	fi; \
	\
	section=''; \
	cr=$(printf '\\r'); \
	while IFS= read -r line; do \
		line=${line%$cr}; \
		case \"$line\" in \
			'# Staged'*) section=staged; continue;; \
			'# Unstaged'*) section=unstaged; continue;; \
			'# Untracked'*) section=untracked; continue;; \
			'#'*|'') continue;; \
		esac; \
		case \"$line\" in '['?'] '*) ;; *) continue;; esac; \
		mark=$(printf '%s' \"$line\" | cut -c2); \
		file=$(printf '%s' \"$line\" | cut -c5-); \
		case \"$mark\" in \
			x|X) \
				if [ \"$section\" != staged ]; then \
					git add -- \"$file\"; \
				fi;; \
			s|S) \
				printf '%s\\0' \"$file\" >> \"$stash_list\";; \
			D) \
				if [ \"$section\" = staged ]; then \
					git restore --staged -- \"$file\" 2>/dev/null; \
				fi; \
				if git ls-files --error-unmatch -- \"$file\" >/dev/null 2>&1; then \
					git restore -- \"$file\" 2>/dev/null; \
				else \
					git clean -f -- \"$file\" 2>/dev/null; \
				fi;; \
			' ') \
				if [ \"$section\" = staged ]; then \
					git restore --staged -- \"$file\" 2>/dev/null; \
				fi;; \
		esac; \
	done < \"$tmp\"; \
	\
	if [ -s \"$stash_list\" ]; then \
		xargs -0 git stash push -m \"Stashed $(date +%Y-%m-%d_%H:%M)\" -- < \"$stash_list\"; \
	fi; \
}; f"

publish = "!f() { \
	branch=$(git rev-parse --abbrev-ref HEAD); \
	if [ \"$branch\" = main ]; then \
		echo 'Push blocked: use git push to update main directly'; \
		exit 1; \
	fi; \
	git push origin HEAD:$branch; \
	git branch -u origin/$branch; \
}; f"

force = "!f() { \
	branch=$(git rev-parse --abbrev-ref HEAD); \
	read -p \"Force-push to $branch? (y/N) \" confirm; \
	if [ \"$confirm\" = \"y\" ]; then \
		git push origin \"$branch\" --force-with-lease; \
	else \
		echo \"Aborted.\"; \
	fi; \
}; f"

pop = stash pop
continue = rebase --continue
undo = reset --soft HEAD~1
here = rev-parse --abbrev-ref HEAD
```
