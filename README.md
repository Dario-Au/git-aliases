# list of git aliases
open gitconfig: `git config --global --edit`

```
upstream = "!f() { \
	branch=${1:-main}; \
	git fetch && git rebase origin/$branch; \
}; f"

restart = "!f() { \
	branch=${1:-main}; \
	echo \"WARNING: This will discard all local changes on $branch.\"; \
	read -p 'Are you sure you want to continue? (y/N) ' confirm; \
	if [ \"$confirm\" = \"y\" ]; then \
			git fetch && git switch $branch && git reset origin/$branch --hard; \
	else \
			echo 'Aborted.'; \
	fi; \
}; f"

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
}; f"

patch = "!f() { \
	branch=$(git rev-parse --abbrev-ref HEAD); \
	count=${1:-2}; \
	if ! git diff --quiet || ! git diff --cached --quiet; then \
		echo 'You have uncommitted changes. Please commit or stash them first.'; \
		exit 1; \
	fi; \
	git fetch && git rebase -i HEAD~$count; \
	read -p \"Force-push to $branch? (y/N) \" confirm; \
	if [ \"$confirm\" = \"y\" ]; then \
		git push origin \"$branch\" --force-with-lease; \
	else \
		echo \"Aborted.\"; \
	fi; \
}; f"

out = "!f() {	\
	git checkout -b ${1}; \
}; f"

here = rev-parse --abbrev-ref HEAD

```
