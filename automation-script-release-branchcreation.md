Got it 👍
You want a **command-line script** (runnable in **Git Bash / CMD**) that:

* Takes **a list of repositories, release version, repo type, and Jira ticket**.
* Creates **release branches**, updates version numbers in specific files depending on repo type, then creates **feature branches** and **PRs**.

Here’s a **Bash script** that covers your steps:

---

## 🐚 `release_script.sh`

```bash
#!/bin/bash
set -e

# Input Parameters
# repos.txt format:
# repo_url releaseVersion repoType jiraTicket
# Example:
# https://github.com/org/backend-service.git 1.2.3 BE JIRA-123
# https://github.com/org/frontend-app.git 1.2.3 FE JIRA-123
# https://github.com/org/deployment.git 1.2.3 DP JIRA-123

INPUT_FILE="repos.txt"

while read -r repo releaseVersion repoType jiraTicket
do
  echo "==============================="
  echo "Processing Repo: $repo"
  echo "Release Version: $releaseVersion"
  echo "Repo Type: $repoType"
  echo "Jira Ticket: $jiraTicket"
  echo "==============================="

  # Clone repo if not exists
  repoName=$(basename "$repo" .git)
  if [ ! -d "$repoName" ]; then
    git clone "$repo"
  fi
  cd "$repoName"

  # Fetch and checkout main/master
  git fetch origin
  if git show-ref --quiet refs/heads/main; then
    git checkout main
  else
    git checkout master
  fi
  git pull

  # Create release branch
  releaseBranch="release/$releaseVersion"
  git checkout -b "$releaseBranch"
  echo "✅ Created release branch: $releaseBranch"

  # Update version numbers based on repoType
  if [ "$repoType" == "BE" ]; then
    echo "🔧 Updating pom.xml / .csproj..."
    if [ -f "pom.xml" ]; then
      sed -i.bak "s|<version>.*</version>|<version>$releaseVersion</version>|g" pom.xml
    fi
    if ls *.csproj >/dev/null 2>&1; then
      sed -i.bak "s|<Version>.*</Version>|<Version>$releaseVersion</Version>|g" *.csproj
    fi
  elif [ "$repoType" == "FE" ]; then
    echo "🔧 Updating package.json and sonar-project.properties..."
    if [ -f "package.json" ]; then
      sed -i.bak "s|\"version\": \".*\"|\"version\": \"$releaseVersion\"|g" package.json
    fi
    if [ -f "sonar-project.properties" ]; then
      sed -i.bak "s|sonar.projectVersion=.*|sonar.projectVersion=$releaseVersion|g" sonar-project.properties
    fi
  elif [ "$repoType" == "DP" ]; then
    echo "📦 Deployment project, skipping version update."
  fi

  git add .
  git commit -m "chore: prepare release $releaseVersion"
  git push origin "$releaseBranch"

  # Create feature branch from release branch
  featureBranch="feature/$jiraTicket"
  git checkout -b "$featureBranch"
  echo "✅ Created feature branch: $featureBranch"
  git push origin "$featureBranch"

  # Create PR (using GitHub CLI)
  echo "🔗 Creating PR from $featureBranch -> $releaseBranch"
  gh pr create --base "$releaseBranch" --head "$featureBranch" --title "$jiraTicket - Feature changes" --body "Auto-generated PR for $jiraTicket"

  cd ..
done < "$INPUT_FILE"
```

---

## ✅ Usage

1. Install **GitHub CLI** (`gh`) for PR creation.

   * [Install Guide](https://cli.github.com/)
   * Run `gh auth login` once.

2. Create `repos.txt` file with inputs:

```
https://github.com/org/backend-service.git 1.2.3 BE JIRA-123
https://github.com/org/frontend-app.git 1.2.3 FE JIRA-123
https://github.com/org/deployment.git 1.2.3 DP JIRA-123
```

3. Run script:

```bash
chmod +x release_script.sh
./release_script.sh
```

---

## ⚡ Key Notes

* `${releaseVersion}`, `${jiraTicket}` placeholders are dynamically replaced.
* Supports `pom.xml`, `.csproj`, `package.json`, and `sonar-project.properties`.
* Uses **GitHub CLI** (`gh`) for PR creation.
* For GitLab/Azure DevOps, API calls would replace the PR creation step.

---

👉 Do you want me to extend this script for **GitLab/Gerrit/Bitbucket PRs** too, or will you be using **GitHub only**?
