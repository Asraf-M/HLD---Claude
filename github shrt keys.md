-------------------------------
PS C:\\GitHub\\sdlc-danfoss-polarion-codebase> git add .github/knowledgeBas.instruction.md
-------------------------------

**Stage specific file:**
git add .github/knowledgeBas.instruction.md

-------------------------------
**Stage all changed files:**
git add .

-------------------------------
**Unstage a specific file (keeps your changes, just removes from staging):**
git restore --staged .github/knowledgeBas.instruction.md

-------------------------------
**Unstage everything:**
git restore --staged .

-------------------------------
cd C:\\GitHub\\sdlc-danfoss-polarion-codebase
git commit -m "Add knowledgeBase instruction file"
git push origin Develop

-------------------------------  -------------------------------  -------------------------------  -------------------------------  

## Pull / Get Specific File or Folder from GitHub

**Fetch then get a specific file from remote branch:**
git fetch origin
git checkout origin/Develop -- .github/vm-copilot.instructions.md

-------------------------------
**Get a specific folder from remote:**
git fetch origin
git checkout origin/Develop -- .github/

-------------------------------
**Restore a file to its last committed state (discard local changes):**
git restore .github/vm-copilot.instructions.md

-------------------------------
**Pull everything from remote and merge:**
git pull

-------------------------------

