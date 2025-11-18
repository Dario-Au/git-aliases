# list of git aliases
open gitconfig: `git config --global --edit`

```
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

continue = rebase --continue

here = rev-parse --abbrev-ref HEAD

```
