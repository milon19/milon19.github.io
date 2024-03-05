# Setup multiple github in same Mac

- Step 1: Navigate to the .ssh Directory
  - `cd ~/.ssh` or `mkdir ~/.ssh`

- Step 2: Create SSH Keys
  - `ssh-keygen -t rsa -C "email@personal.com" -f "github-personal"`
  - `ssh-keygen -t rsa -C "email@company.com" -f "github-company"`

- Step 3: Add SSH Keys to the Agent
  - `ssh-add --apple-use-keychain ~/.ssh/github-personal`
  - `ssh-add --apple-use-keychain ~/.ssh/github-company`

- Step 4: Add SSH Public Keys to GitHub Accounts
  - `pbcopy < ~/.ssh/github-personal.pub`
  - `pbcopy < ~/.ssh/github-company.pub`

- Step 5: Configure SSH Alias
  - `open ~/.ssh/config` or `touch ~/.ssh/config`
  - then open it and add aliases:
```
Host github.com-personal

   HostName github.com

   User git

   IdentityFile ~/.ssh/github-personal

Host github.com-company

   HostName github.com

   User git

   IdentityFile ~/.ssh/github-company
```

- Step 6: Clone and Configure Repositories
  - `git clone git@github.com-personal:github-personal/{repo-name}.git`
  - `git clone git@github.com-company:github-company/{repo-name}.git`
