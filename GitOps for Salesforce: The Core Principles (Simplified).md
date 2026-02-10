# GitOps for Salesforce: The Core Principles (Simplified)

Let me give you the **practitioner's guide** - how I'd explain this to a team starting Salesforce DevOps today.

---

## 🎯 **The Four Pillars**

### 1. **BUILD & RUN: Two Timelines, One Org**

**The Problem:**
- You have ONE production org
- But TWO types of work happening:
  - **RUN (Today)**: Fixing live bugs that hurt customers *now*
  - **BUILD (Tomorrow)**: Building new features for next quarter

**The Solution:**
```
master branch    → Production (what's live RIGHT NOW)
integration      → Staging (what's coming NEXT MONTH)
```

**The Rule:**
> Never mix "fixing today" with "building tomorrow" in the same branch.

**Why?**
- A hotfix takes 2 hours. You deploy it today.
- A new feature takes 2 months. You deploy it next quarter.
- If you mix them, you can't deploy the hotfix without accidentally deploying half-finished features.

---

### 2. **DAG: The "No U-Turns" Rule**

**What it means:**
```
DAG = Directed Acyclic Graph
     = "Git history flows one direction, no loops"
```

**In plain English:**
```
✅ ALLOWED:  master → integration  (promote production code to staging)
❌ FORBIDDEN: integration → master  (don't merge staging back to production)
```

**Why people want this:**
- Clean git history
- Easy to see "what came from where"
- Matches traditional software dev (Java, Python)

**Why it FAILS in Salesforce:**
```
Monday:    Hotfix in master (fix a validation rule)
Tuesday:   Feature in integration (adds new field)
Wednesday: Try to merge integration → master for release
Result:    CONFLICT in the same XML file

Git thinks these are two unrelated changes fighting.
Reality: They're both valid, just need to coexist.
```

**My Verdict:** 
> **Ignore DAG for Salesforce.** The XML merge conflicts aren't worth the "pretty graph."

---

### 3. **Retrofit: Syncing the Two Timelines**

**The Core Concept:**
> "Retrofit" means: Copy what happened in RUN track back to BUILD track.

**Why you need it:**
```
         RUN (master)          BUILD (integration)
Monday   [Fix bug A]           [Building feature X]
         ↓
         Deploy to Prod
                               [Still building feature X]
                               ← BUG: Still has bug A!
```

**Without retrofit:**
- Next quarter, you deploy feature X
- It OVERWRITES your bug fix
- Bug A comes back to production

**The Fix (Two Ways):**

#### **Option A: Git Merge (Recommended)**
```bash
# After deploying hotfix to production
git checkout integration
git merge master  # "Give BUILD track everything from RUN track"
git push
```

**Pros:**
- Simple
- Git tracks the shared history
- Fewer conflicts later

**Cons:**
- Creates "loops" in git graph (violates DAG)

---

#### **Option B: Org Retrofit (Special Cases)**
```bash
sf hardis:org:retrieve:sources:retrofit
```

**When to use this:**
- Someone changed production ORG directly (bypassed Git)
- You need to "pull reality" from Salesforce back into Git

**Example:**
```
Admin Jane logs into Production
Jane creates a new Report in the UI (not in Git)
You run: sf hardis:org:retrieve:sources:retrofit
It detects the Report and commits it to your branch
```

**My Rule:**
```
Git Merge = Syncing CODE you wrote
Org Retrofit = Catching DRIFT you didn't write
```

---

### 4. **Monitoring: The 3 Critical Checks**

You can't manage what you don't measure. Here's what to monitor:

#### **A. Are the Tracks Aligned?** (Drift Detection)
```bash
# Check if master and integration have diverged badly
git log master..integration --oneline | wc -l

# If > 100 commits apart: DANGER
# Do an immediate retrofit merge
```

**Alert threshold:** If RUN and BUILD are >1 week apart, conflicts will explode.

---

#### **B. Are Sandboxes Fresh?** (Environment Health)
```bash
# Check when sandbox was last refreshed
sf data query \
  --query "SELECT SandboxName, LastModifiedDate FROM SandboxInfo" \
  --target-org production

# If > 30 days old: STALE
# Refresh before next hotfix
```

**Alert threshold:** Hotfix sandbox >1 week old = useless for emergency fixes.

---

#### **C. Are Hotfixes Retrofitted?** (Process Compliance)
```bash
# Check for commits in master NOT in integration
git log integration..master --oneline

# If ANY commits found: INCOMPLETE RETROFIT
# Automated alert to DevOps team
```

**Alert threshold:** If hotfix deployed >24 hours ago and not retrofitted = future bug.

---

## 🎓 **The Mental Model I Use**

Think of it like **two train tracks:**

```
TRACK 1 (RUN):     [Station: Production]
   ↓
   Quick repairs done at this station
   Train runs every day

TRACK 2 (BUILD):   [Station: Staging]
   ↓
   Major renovations happening here
   Train runs once per quarter
```

**The Bridge (Retrofit):**
```
Every night, a bridge copies repairs from Track 1 → Track 2
So when Track 2's train finally departs (quarterly release),
it has all the repairs Track 1 made during the quarter.
```

**If you forget the bridge:**
- Track 2 deploys the quarterly release
- It LACKS the repairs from Track 1
- Bugs come back

---

## 🔧 **My Recommended Workflow (Pragmatic)**

### **Daily Operations**

```bash
# Morning: Check if any hotfixes need retrofitting
git checkout integration
git pull origin master  # If this fails, you have conflicts to resolve
git push

# Automated via GitHub Actions at 2 AM every day
```

### **Hotfix Flow**

```bash
# 1. Branch from production reality
git checkout master
git pull
git checkout -b fix/critical-bug

# 2. Fix in VSCode, deploy to hotfix sandbox
sf project deploy start --target-org hotfix-sandbox

# 3. Merge to master (deploys to production via CI/CD)
git push origin fix/critical-bug
# Create PR → master

# 4. IMMEDIATELY retrofit (same day)
git checkout integration
git merge master
git push
```

### **Release Flow**

```bash
# When BUILD track is ready for production (quarterly)
git checkout master
git merge integration  # "Promote staging to production"
git push  # Triggers production deployment
```

---

## ⚠️ **The ONE Rule That Matters Most**

> **Never go more than 24 hours between hotfix deployment and retrofit.**

**Why:**
- Conflicts grow exponentially with time
- Easy to forget what the fix was for
- Next developer will hit the conflict, not you

**Enforcement:**
```yaml
# GitHub Action: Auto-retrofit on hotfix merge
on:
  push:
    branches: [master]

jobs:
  auto-retrofit:
    runs-on: ubuntu-latest
    steps:
      - name: Merge master → integration
        run: |
          git checkout integration
          git merge master || exit 1  # FAIL BUILD if conflicts
          git push
```

---

## 📊 **Decision Tree (When to Use What)**

```
User Reports Bug
       ↓
   Is it breaking production?
       ↓
   YES → RUN track (master branch)
       ↓
   Fix → Deploy → RETROFIT same day
   
   NO → Is it a new feature?
       ↓
   YES → BUILD track (integration branch)
       ↓
   Develop → Test → Deploy next quarter
```

---

## 🎯 **The Simplified Truth**

**DAG:** Ignore it for Salesforce. XML doesn't play nice with strict DAG rules.

**BUILD & RUN:** Two branches for two speeds of work (emergency vs. planned).

**Retrofit:** Merge master → integration DAILY or suffer later.

**Monitoring:** Track drift between branches, sandbox freshness, and retrofit compliance.

**Bottom Line:**
> Git is your source of truth, but Salesforce is stateful. 
> Your job is keeping Git (the map) and Salesforce (the territory) synchronized.

---

**This is how I'd train a new team.** Want me to add anything specific?
