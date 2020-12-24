
# Git Commands
From the book "Learn Git in a Month of Lunches" by Rick Umali

## Add: add new/changed file to the repository

__git add --dry-run .__: Show what git add would do

__git add -p__: Pick parts of your changes to add to the staging area 


## Blame: which commit contributed to each line of a file

__git blame FILE__: Display blame output of FILE on the command line.

__git --no-pager blame FILE > FILE-annotate__: Save the blame output of FILE to FILE-annotate on the command line.


## Branch: list, create, or delete branches

__git branch -d master__:  Delete the branch named master.

__git branch -v__:  List all branches with SHA1 ID information.

__git branch fixing_readme YOUR_SHA1ID__:  Make a branch using YOUR_SHA1ID as the starting point.

__git branch --all__: Show remote-tracking branches in addition to local branches.

__git checkout -f master__:  Check out the master branch, throwing away any changes in your current branch.

__git branch --column__: List all branches in columns

__git branch -r --contains SHA1_ID__: Similar to the preceding command, in that it will identify all the branches that contain this SHA1 ID (-r specifies remote-tracking branches; omit this to print local branches).


## Checkout: switch branches or restore working tree files

__git checkout file__: Check out the latest committed version of the file into your working directory

__git checkout YOUR_SHA1ID__: Change your working directory to match the version specified in YOUR_SHA1ID.

__git checkout -b another_fix_branch fixing_readme__: Make a branch named another_fix_branch using branch fixing_readme as the starting point.

__git checkout tags/TAG__: Check out by tag


## Clone: clone a repository into a new directory

__git clone source destination_dir__: Clone the Git repository at source to the destination_dir.

__git clone --bare source destination_dir__: Clone the bare directory of the source repository into the destination_dir. By convention, destination_dir should end with .git.


## Commit: record changes to the repository  

__git commit --allow-empty -m "Initial commit"__: Create a commit without adding any files.


## Config: get and set repository or global options

__git config --global alias.lol "log --graph --decorate --pretty=oneline --all --abbrev-commit"__: Make an alias named lol for the git log command in the previous row.

__git config --local --list__: List the local (repository-specific) Git configuration.

__git config --global --list__: List the global (user-specific) Git configuration.

__git config --system --list__: List the system (server-specific) Git configuration.

__git config --local log.date relative__: Save the relative date format in the local Git configuration.

__git config --local --edit__: Edit the local (repository-specific) Git configuration.

__git config --global --edit__: Edit the global (user-specific) Git configuration.

__git config --system --edit__: Edit the system (server-specific) Git configuration.

__git -c core.editor=echo config --local --edit__: Print the name of the local Git configuration file.

__git -c core.editor=nano config --local --edit__: Edit the local Git configuration file using nano.

__git config core.excludesfile__: Print the value of the core.excludesfile Git configuration setting.



## Diff: show changes between commits, commit and working tree, etc

__git diff__ --staged Show any changes between the staging area and the repository

__git diff BRANCH1...BRANCH2__: Indicate the difference between BRANCH1 and BRANCH2 relative to when they first became different.

__git diff --name-status BRANCH1...BRANCH2__: Summarize the difference between BRANCH1 and BRANCH2, by listing each file and its status.


## GUI: a portable graphical interface to Git

__git gui__: Start Git GUI

__git citool__: Start Git GUI to commit changes

__gitk__: Start gitk (git log viewer)

__git gui blame FILE__: Bring up a FILE in the Git GUI showing git blame output (each line showing what commit it’s from).

__git gui browser REV__: List all files at REV (use HEAD for the current directory) in the GUI browser.


## Log: show commit logs

__git log --shortstat --oneline__: Show history using one line per commit, and listing each file changed per commit

__git log --patch__: Display the history, showing the file differences between each commit.

__git log__: --stat Display the history, showing a summary of the file changes between each commit.

__git log --patch-with-stat__: Display the history, combining patch and stat output.

__git log --oneline file_one__: Display the history for file_one.

__git log --graph --decorate --pretty=oneline --all --abbrev-commit__: View history of the repository across all branches

__git reflog__: Display the reflog (the internal history of all the times that you changed HEAD).

__git log -1__: A shorthand for git log -n 1 (show only the most recent commit).

__git log --oneline --all__: Display all commit log entries from all branches. (Normally, git log displays only entries from the current branch.)

__git log --simplify-by-decoration --decorate --all --oneline__: Display the history in a simplified form.

__git log --merges__: List commits that are the result of merges.

__git log --oneline FILE__: List commits that affect FILE.

__git log --grep=STRING__: List commits that have STRING in the commit message.

__git log --since MM/DD/YYYY --until MM/DD/YYYY__: List commits between two dates.

__git shortlog__: Summarize commits by authors.

__git shortlog -e__: Summarize commits by authors (and show email address).

__git log --author=AUTHOR__: List commits by AUTHOR (name or email).

__git log --stat HEAD^..HEAD__: List commits (with files) between the current commit and its immediate parent.

__git log --patch HEAD^..HEAD__: List commits (with text changes) between the current commit and its immediate parent.

__git log --oneline master..new_feature__: Show the commits between the master branch and the new_feature branch.

__git -c log.date=relative log -n 2__: Show the last two commits using the relative date format.


## Merge: join two or more development histories together

__git merge BRANCH2__: Merge BRANCH2 into the current branch that you’re on.

__git mergetool__: Open a tool to help perform a merge between two conflicted branches.

__git merge --abort__: Abandon a merge between two conflicted branches.

__git merge-base BRANCH1 BRANCH2__: Show the base commit between BRANCH1 and BRANCH2.

__git merge FETCH_HEAD__: Merge the new commits from FETCH_HEAD into the current branch.

__git merge --no-ff BRANCH__: Merge BRANCH into the current branch, creating a merge commit even if it’s a fast-forward commit.


## Mv: move or rename a file, a directory, or a symlink

__git mv file1 file2__: Rename file1 to file2 in the staging area


## Pull and fetch: fetch from and integrate with another repository or a local branch

__git pull__: Sync your repository with the repository that you cloned from (a.k.a. the upstream repository). This command comprises two commands: git fetch and git merge. 

__git fetch__: The first part of git pull. This brings in new commits from the remote repository and updates the remote-tracking branch.

__git pull --ff-only__: The --ff-only switch will allow a merge only if FETCH_HEAD is a descendant of the current branch (a fast-forward merge).


## Rebase: reapply commits on top of another base tip

__git reset --hard HEAD@{4}__: Reset HEAD to point to the SHA1 ID represented by HEAD@{4}. The --hard switch says to reset both the staging area and the working directory.

__git rebase master__: Rebase your current branch with the latest commit from master.

__git rebase --interactive master__: Interactively rebase your current branch with the latest commit from master. This opens an editor, allowing you to pick and choose which commits will be included in the rebase.


## Remote: manage set of tracked repositories

__git remote__:  Display the name of the remote(s) in the current repository.

__git remote -v show__:  Display the names of the remotes along with the corresponding remote URL.

__git remote add bob ../math.bob__:  Add a remote named bob that points to the local repository in ../math.bob.

__git ls-remote REMOTE__:  Display the references of a remote repository (use . as the REMOTE when you want the current local repository).

__GIT_TRACE_PACKET git ls-remote REMOTE__: Display the underlying network interaction.


## Reset: reset current HEAD to the specified state

__git reset__: file Reset your staging area, removing any changes you’ve added with git add


## Rm: remove files from the working tree and from the index

__git rm file__: Remove file from staging area


## Stash: stash the changes in a dirty working directory away

__git stash__: Set the current work in progress (WIP) to a stash (holding area), so you can perform a git checkout.

__git stash list__: List works in progress that you’ve stashed away.

__git stash pop__: Apply the most recently saved stash to the current working directory; remove it from the stash.


## Tag: create, list, delete or verify a tag

__git tag TAG_NAME -m "MESSAGE" YOUR_SHA1ID__: Create a tag named TAG_NAME, pointing to YOUR_SHA1ID. The tag will have a short MESSAGE associated with it.

__git tag__: List all tags.

__git show TAG_NAME__: Show information about the tag named TAG_NAME.

__git tag -a TAG_NAME -m TAG_MESSAGE SHA1ID__: Create a tag to the SHA1ID with the name TAG_NAME and the message TAG_MESSAGE.

__git push origin TAGNAME__: Push the tag named TAGNAME to the remote named origin.

__git push --tags__: Push all tags to the default remote.

__git push origin :TAGNAME__: Delete the tag named TAGNAME on the remote named origin.

__git tag -d TAGNAME__: Remove the tag named TAGNAME from your local repository.

__git show-ref --tags__: Get SHA1ID for the tags


## Miscellaneous

__git ls-tree HEAD__: Display all the files for HEAD (the current branch).

__git name-rev SHA1_ID__: Print a name for the specified SHA ID, based on the closest branch.

__git grep STRING__: Find all files that contain STRING.

__git cherry-pick SHA1 ID__: Copy the commit to the current branch that you’re on.

__git flow__: A Git command that becomes available after installing gitflow.


