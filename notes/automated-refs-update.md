This post goes through a pet project for myself- to automate the updating of [refs](https://git-scm.com/book/en/v2/Git-Internals-Git-References) from remotes for Git projects I work with, using Ansible.

# Staying up to date with projects

Working with multiple people on multiple projects can be a difficult thing to keep track of. Sometimes you may open a new branch to start working on a new feature, get your work done, and submit in a pull request- only to have your reviewer(s) curse you that you forgot to rebase your branch on the destination branch. As automation engineers, we want all forgetful, repetitive tasks to be done for us in some way or fashion. For me, this is one such small task that I wanted to get done via the run of a single command on my system.

# The process

I keep my Git projects within my [WSL](https://docs.microsoft.com/en-us/windows/wsl/) setup- where the root folder is my home folder. Within the home folder, I have different types of files and folders:

- Application specific folders (created by Ansible, Docker, VS Code...)
- Git projects
- Virtual environment folders
- Random development files and folders

So chalking a straightforward way to fetch all refs for all Git repositories within my root directory looked like this:

1. Get a list of all directories in root directory
2. For all directories, find out if it is a Git repository
3. If it is a Git repository, find out if Git remote repository tracking is configured
4. If 2. and 3. are satisfied, it is a repository for which remote refs can be fetched. Fetch remote refs for Git repositories that satisfied point 2. and 3.

For me, this looks like a good opportunity to use Ansible because I find writing YAML easier than dealing with shell scripts.

> NOTE: This works properly only when your Git repositories have personal access tokens configured to interact with your remote Git repositories. Read [this](https://docs.github.com/en/github/authenticating-to-github/creating-a-personal-access-token) for GitHub

# The playbook

To get a basic hang of Ansible, please find my [post on Ansible](ansible-hostname-compliance.md). In a file, I create a file named `git-fetch-remotes.yml`, and define the bed for the playbook:

```yaml
---

- name: FETCH REMOTES FOR YOUR REPOS
  hosts: localhost
  gather_facts: no

  vars:
    root_directory: "/home/username/"
```

The playbook runs on the `localhost`, which is basically the Ansible controller, and we need to let the playbook know a variable named `root_directory` so it knows where to check for Git repositories.

Now to get to the tasks:

```yaml
  tasks:
  
  - name: LIST ALL DIRECTORIES IN ROOT WITH INDICATOR
    shell: 
      cmd: "ls -d */"
      chdir: "{ { root_directory } }"
    register: dirs
  
  - name: LOOP THROUGH LIST OF ITEMS IN ROOT DIRECTORY
    include_tasks: find-git-repos.yml
    loop: "{ { dirs.stdout_lines } }"
  
  - name: GIT FETCH FOR REMOTES
    command:
      chdir: "{ { root_directory ~ item } }"
      argv:
        - git
        - fetch
        - "--all"
    loop: "{ { git_repos } }"
```

> (Please remove the spaces in between the curly brace characters, GitHub pages does not display it properly. Follow [Jinja2 variable syntax](https://ttl255.com/jinja2-tutorial-part-1-introduction-and-variable-substitution/))

A rough run-through of the tasks in here:

- Using `ls -d */`, get a list of all directories in root directory, register result in a variable
- Loop through the list of directories found, and execute tasks from a file named `find-git-repos.yml` for each of those items in the list. A task in this file registers a variable named `git_repos`, which is a list of Git repositories found in the root directory
- Run `git fetch --all` in these Git repositories

And a look into `find-git-repos.yml`:

```yaml
---

- name: GET LIST OF ALL ITEMS IN DIRECTORY
  command:
    cmd: "ls -al"
    chdir: "{ { root_directory ~ item } }"
  register: ls_al

- name: IF GIT REPO EXISTS IN DIRECTORY, REGISTER CONFIGURED GIT REMOTES
  command:
    cmd: "git remote -v"
    chdir: "{ { root_directory ~ item } }"
  when: '".git" in ls_al.stdout'
  register: remotes_list

- name: IF GIT REMOTES PRESENT, ADD TO git_repos LIST
  set_fact:
    git_repos: "{ { git_repos|default([])|union([item]) } }"
  when: remotes_list.stdout is defined and remotes_list.stdout != ""
```

- Find all files and directories within directory
- If a git repository is initialized within directory (`.git` is present), find Git remotes configured for the repository
- If Git remotes exist, we can fetch from remotes for the repository- so add it to the `git_repos` list we finally need in the main playbook!

Once this playbook is ready, you can keep an alias for it in your `.bashrc` or similar file (depending on which *nix operating system you have). I kept an alias called `go-fetch`, so when I run the command from my BASH shell, the playbook executes `ansible-playbook /path/to/git-fetch-remotes.yml`.

```bash
alias go-fetch="ansible-playbook /path/to/git-fetch-remotes.yml"
```

Additionally, you can look to have `go-fetch` run in a scheduled manner on your machine using a [cron](https://www.man7.org/linux/man-pages/man8/cron.8.html) job by adding an entry to your crontab.

# References

- [Ansible command module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/command_module.html)
- [Ansible shell module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/shell_module.html)
- [Ansible include tasks module](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/include_tasks_module.html)
- [Ansible filters](https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters.html)
- [Ansible loops](https://docs.ansible.com/ansible/latest/user_guide/playbooks_loops.html)