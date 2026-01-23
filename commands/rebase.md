## Command: rebase-with-develop

### Usage
rebase-with-develop <branch-1> <branch-2> ...

### Behavior
When the user invokes this command with a list of branch names, do the following steps **in order**:

1. Remember the currently checked-out branch.
2. Checkout the `develop` branch.
3. Update `develop` using `git pull` (do not use rebase for this step).
4. Update all submodules recursively:
   - `git submodule update --recursive --init --progress -f`
5. For each branch provided by the user, in the order listed:
   - Checkout the branch
   - Create a backup branch before the rebase with the same name but the current date and time as part of the branch name.
   - Rebase it onto `develop` if the branch is the first branch in the list of branches, or the previous branch in the list otherwise. Use interactive rebase option in git.
   - If a conflict occurs:
     - Stop immediately
     - Inform the user which branch failed
     - Wait until the conflict or issue is solved manually
     - Once confirmed by the user solved the issue execute ``git rebase --continue`` to continue with the interactive rebase process.
     - Once the rebase is done, force push the branch if there were no issues, or ask the user if we should force push the branch otherwise.
     - Once the push is done, continue with the following branch.
6. Print a short summary of what was rebased and which branch was used to rebase each of the branches in the list
7. Print a list of all the commands executed and mention if there were errors or issues with one of these.

### Safety rules
- If any listed branch does not exist locally, ask before creating or fetching it.
- Never delete branches.

### Assumptions
- `develop` is the integration branch.
- The user wants a clean, linear history.
