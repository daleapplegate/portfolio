---
project: OPAS (Oracle Provider Application System)
project_full_name: pas-presentation
language: Java 8
build_tool: Maven 3.6+
framework: Spring Framework, Spring Security (LDAP), JSF 2.1, ICEfaces 3.3
database: Oracle 10g+
deployment_targets:
  - embedded-tomcat-jar
  - external-tomcat-war
runtime: Tomcat 9
entrypoint_class: com.mpqh.opas.tiger.pas.presentation.OPASMain
artifact_version: 26.0.0-SNAPSHOT
ci_cd_platform: Azure DevOps
primary_package: com.mpqh.opas.tiger.pas.presentation
datasource_implementation: Tomcat DBCP2 BasicDataSourceFactory
---

# OPAS Deployment Guide

## Overview
This guide documents all deployment paths for OPAS (Pas-Presentation), including the executable JAR runner, external Tomcat deployment, and manual WAR deployment. All methods are cross-platform (macOS, Linux, Windows).

**For Build & Scripting Reference:** See [OPAS_BUILD_REFERENCE.md](OPAS_BUILD_REFERENCE.md) for structured environment variables, build decision matrices, canonical command templates, and common scripting patterns.

---

## Table of Contents

1. [Getting Started (TL;DR)](#getting-started-tldr)
   - Fastest: Run Embedded Tomcat JAR
   - Production: Deploy to External Tomcat

2. [Prerequisites](#prerequisites)
   - Environment Variables
   - Required Software
   - Network Access
   - Project Structure
   - Maven Profiles

3. [Git Branching Policy (Azure DevOps)](#git-branching-policy-azure-devops)
   - Goals
   - Standard Branch Model
   - Rules
  - Versioning and Release Policy
  - Pipeline Execution Policy
   - PR and Merge Strategy
   - Cleanup Checklist
   - Example Commands

4. [Project Structure and Source Locations](#project-structure-and-source-locations)
   - Two Source Directories
   - Finding Code in Dual Structure
   - Development Guidelines: Tiger = Modern Code
   - How Maven Handles Both Locations
   - Finding Your Tomcat Installation

5. [Deployment Path 1: Executable JAR (Embedded Tomcat 9)](#deployment-path-1-executable-jar-embedded-tomcat-9)
   - Build the JAR
   - Run the JAR
   - Verify Startup

6. [Deployment Path 2: External Tomcat 9 Deploy](#deployment-path-2-external-tomcat-9-deploy)
   - Prerequisites
   - Build and Deploy
   - Persist Your Tomcat Path
   - Configure Application Datasource
   - Start Tomcat
   - Verify Deployment
   - Troubleshooting

7. [Deployment Path 3: Manual WAR Deployment](#deployment-path-3-manual-war-deployment)
   - Build the WAR
   - Deploy to Tomcat
   - Verify

8. [Configuration](#configuration)
   - Database Connection
   - LDAP Authentication
   - Application Properties

9. [Troubleshooting](#troubleshooting)
   - Common Issues & Solutions

10. [Build Artifacts](#build-artifacts)
    - Standard Outputs
    - Source Configuration

11. [Quick Start Summary](#quick-start-summary)
    - Executable JAR
    - External Tomcat
    - Manual WAR

12. [Cross-Platform Compatibility](#cross-platform-compatibility)
    - Configurable Properties
    - Platform-Specific Notes
    - Sharing Configuration
    - Testing Cross-Platform Compatibility

13. [Local Secret Setup (VS Code)](#local-secret-setup-vs-code)
    - Files
    - One-Time Setup
    - Verification
    - Security Rules

14. [VS Code Run and Debug with OPASMain](#vs-code-run-and-debug-with-opasmaindebug-with-opas-main)
    - Prerequisites
    - Option 1: Run/Debug OPASMain directly
    - Option 2: Attach debugger to running process
    - Debugging Notes

---

### Fastest: Run Embedded Tomcat JAR

```bash
# 1. From OPAS root, build
mvn -Ptiger clean package

# 2. Run from project root (any platform)
java -Dopas.contextPath=/pas-presentation \
     -Dopas.docBase=$(pwd)/PasPresentation/target/pas-presentation \
  -jar PasPresentation/target/pas-presentation-26.0.0-SNAPSHOT-runner.jar

# 3. Access at http://localhost:8080/pas-presentation
```

**Requires:** Environment variables set (see Prerequisites below)

### Production: Deploy to External Tomcat

```bash
# 1. Find your Tomcat webapps path:
#    macOS: /usr/local/Cellar/tomcat/9.0.x/libexec/webapps
#    Linux: /opt/tomcat/webapps or /usr/local/tomcat/webapps
#    Windows: C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps

# 2. Build with your Tomcat path
mvn -pl PasPresentation -am package \
  -DexternalTomcatDeploy \
  -Dtomcat.deploy.path=/YOUR/TOMCAT/WEBAPPS/PATH \
  -DskipTests

# 3. Start Tomcat and access at http://localhost:8080/pas-presentation
```

---

## Prerequisites

### Environment Variables
Set these before running any deployment:

```bash
# macOS/Linux:
export OPAS_DB_URL='jdbc:oracle:thin:@[DATABASE IP]:1521:[DB SCHEMA]'
export OPAS_DB_USERNAME='[PAS APP USERNAME]'
export OPAS_DB_PASSWORD='[YOUR_SECURE_PASSWORD]'

# Windows (PowerShell):
$env:OPAS_DB_URL = "jdbc:oracle:thin:@[DATABASE IP]:1521:[DB SCHEMA]"
$env:OPAS_DB_USERNAME = "[PAS APP USERNAME]"
$env:OPAS_DB_PASSWORD = "[YOUR_SECURE_PASSWORD]"

# Windows (cmd.exe):
set OPAS_DB_URL=jdbc:oracle:thin:@[DATABASE IP]:1521:[DB SCHEMA]
set OPAS_DB_USERNAME=[PAS APP USERNAME]
set OPAS_DB_PASSWORD=[YOUR_SECURE_PASSWORD]
```

**Security Note:** Never commit passwords to source control. Use local `.env` files or secure credential management.

### Required Software
- Maven 3.6+
- Java 8+ (JDK or JRE)
- Oracle JDBC driver (included in project)
- Oracle 10g or later (database target)

### Network Access
- **For LDAP Authentication:** VPN connection required to reach `ldap://marspd3ap:1389/dc=mars,dc=mpqhf,dc=org`
- **For Database:** Network connectivity to `[DATABASE IP]:1521` (Oracle host)

### Project Structure
Ensure you're running commands from the OPAS root directory (where `pom.xml` exists at root level):
```
OPAS/
├── pom.xml (root)
├── PasPresentation/
│   ├── pom.xml
│   ├── WebRoot/
│   ├── src/
│   └── target/
├── PasBusiness/
├── PasIntegration/
└── ...
```

### Maven Profiles

Use these profiles to simplify common builds:

- `tiger` (root `pom.xml`): convenience profile for embedded Tomcat builds (sets `skipTests=true`).
- `external-tomcat-deploy` (PasPresentation `pom.xml`): enables copy-to-Tomcat deployment flow.

Common commands:

```bash
# Embedded Tomcat build (recommended)
mvn -Ptiger clean package

# Embedded Tomcat build for PasPresentation + required modules only
mvn -Ptiger -pl PasPresentation -am clean package

# External Tomcat deploy
mvn -pl PasPresentation -am package -DexternalTomcatDeploy -Dtomcat.deploy.path=/path/to/tomcat/webapps -DskipTests
```

## Logging Guidance for Debugging

When adding temporary debug logging in OPAS, prefer guarded INFO logs in hot paths.

- Use if (log.isInfoEnabled()) around INFO messages when the message is built from multiple string concatenations, loops, or method calls.
- This avoids unnecessary work when INFO logging is disabled.
- It is especially important in frequently called methods such as list-building, provider lookups, and render-time helper logic.

When simple direct logging is acceptable:

- A static, short message with no expensive value building.
- Low-frequency code paths where performance impact is negligible.

Practical rule:

- For temporary diagnostics in provider/profile dropdown logic, keep INFO logs guarded.
- After root cause is confirmed, remove or reduce temporary debug logs before release.

## Git Branching Policy (Azure DevOps)

### Goals
- Keep `master` stable and releasable.
- Avoid branch chains (`branch-from-branch-from-branch`) for active development.
- Make monthly project work easy to merge and easy to clean up.

### Standard Branch Model
- `master`: production/release branch.
- `feature/<project-name>`: one active branch per project stream.
- Optional temporary branch only when needed: `integration/next` (short-lived holding branch).

### Rules
1. Always create new work branches from `master`.
2. Use PRs for all merges (no direct pushes to `master`).
3. Keep one primary feature branch per project; avoid stacked long-lived branches.
4. Delete branches after merge.

### Versioning and Release Policy

OPAS versioning standard:
- Use `YY.MINOR.PATCH`
- `YY` = calendar year (2026 => `26`)
- `MINOR` = increment for each production release from `master`
- `PATCH` = increment for hotfixes on a released minor line

Examples:
- `26.0.0` initial 2026 release
- `26.1.0` next planned release
- `26.1.1` hotfix on top of `26.1.0`

Branch/build metadata policy:
- Do not bump release numbers for feature branch merges
- Keep development line as `-SNAPSHOT` on `master` between releases
- Add build traceability via CI metadata (branch, build number, commit SHA)

Release branch flow:
1. `master` carries next dev version (example: `26.2.0-SNAPSHOT`)
2. Create `release/26.2.0` from `master`
3. Set version to `26.2.0` on release branch and test
4. Deploy/tag release from release branch
5. Merge release branch back to `master`
6. Bump `master` to next dev version (example: `26.3.0-SNAPSHOT`)

### Pipeline Execution Policy

- Canonical pipeline YAML: `azure-pipelines.yml`
- CI triggers run for `feature/*` and `platform/*` branches
- Pull request validation runs for PRs targeting `master`
- Windows deploy stage is manual at queue time using parameter `runWindowsDeploy`
- `runWindowsDeploy` defaults to `false` to prevent accidental server deploys during normal CI builds

### Typical Feature Branch Sync Routine

Sync your feature branch from `master` frequently (daily when active, and always before opening/completing a PR).

#### Recommended Team Flow (Merge-Based)

```bash
# From your feature branch
git checkout feature/<name>

# Update remote refs
git fetch origin --prune

# Bring latest master into your branch
git merge origin/master

# Resolve conflicts, run build/tests, then push
mvn -Ptiger clean package
git push origin feature/<name>
```

Use this flow when multiple developers share long-lived feature branches and want stable, auditable history.

#### Alternative Flow (Rebase-Based)

```bash
git checkout feature/<name>
git fetch origin --prune
git rebase origin/master
mvn -Ptiger clean package
git push --force-with-lease origin feature/<name>
```

Use this only when your branch is not shared or your team explicitly allows force-push on feature branches.

#### Pre-PR Sync Checklist
1. `git fetch origin --prune`
2. Sync from `origin/master` (merge or rebase per team policy)
3. Build passes locally (`mvn -Ptiger clean package`)
4. Push synced branch before creating/updating PR

### Commit Message and Squash Policy

Keep feature branch history concise and reviewable.

Commit message rules:
- Use short, imperative messages that describe the completed change.
- Prefix bug fixes with the Azure DevOps work item when applicable.
- Preferred bug format: `Bug <id>: <short description>`
- Example: `Bug 234: Fix discharge county context for agency needs`

Commit squash rules:
- Before opening or completing a PR, squash local work so the branch has one meaningful commit per feature or bug fix.
- The final commit should represent the complete logical change, not intermediate debugging, formatting, or checkpoint commits.
- If multiple unrelated fixes are discovered, split them into separate branches/PRs instead of one squashed commit.
- Do not rewrite history on shared branches without coordinating with other contributors.

Recommended squash flow before PR:

```bash
git checkout feature/<name>
git fetch origin --prune
git merge origin/master
mvn -Ptiger clean package
git reset --soft origin/master
git commit -m "Bug <id>: <short description>"
git push --force-with-lease origin feature/<name>
```

Use `--force-with-lease` only for your own feature branch or when the team agrees the branch is not shared.

### PR and Merge Strategy
- PR target is normally `master`.
- Require build validation in Azure DevOps before completing PR.
- Require at least 1 reviewer.
- Prefer squash merge for cleaner history (or use one merge style consistently team-wide).

### When Another Project Must Release First
If your project is ready but cannot merge to `master` yet:
1. Merge your working branches up into your single project feature branch (for example, `feature/cfs`).
2. Option A (preferred): keep PR open from `feature/cfs` to `master` and wait.
3. Option B (if team needs shared integration before release): PR into short-lived `integration/next`.
4. After the other project releases, sync `feature/cfs` with latest `master`, re-run build/tests, then complete PR to `master`.

### Cleanup Checklist (After Merge to master)
1. Delete merged branch in Azure DevOps.
2. Delete local branch.
3. Delete any temporary holding branch (`integration/next`) if used.
4. Confirm only active branches remain.

### Example Commands

Refresh an existing local branch from remote safely:

```bash
git checkout platform/maven
git status
git branch -vv
git pull --rebase
```

If you have uncommitted local changes:

```bash
git stash -u
git pull --rebase
git stash pop
```

```bash
# Rename an old branch to the proper feature branch name
git checkout master-maven
git branch -m feature/cfs
git push -u origin feature/cfs
git push origin --delete master-maven

# Sync feature branch with latest master before final PR
git checkout master
git pull
git checkout feature/cfs
git merge master

# After merge to master, clean up local branches
git branch -d feature/cfs
```

## Project Structure and Source Locations

### Two Source Directories

The OPAS project has **two source directories** due to its migration from Eclipse (legacy) to Maven:

#### 1. Maven Standard Location: `src/`
The primary source directory following Maven conventions:
```
PasPresentation/src/
├── main/
│   ├── java/
│   │   ├── com/mpqh/pas/
│   │   │   ├── tiger/           ← **MODERN CODE** - All new development goes here
│   │   │   │   ├── beans/
│   │   │   │   ├── services/
│   │   │   │   ├── controllers/
│   │   │   │   └── ...
│   │   │   └── presentation/    ← Legacy code (pre-Tiger)
│   │   │       ├── beans/
│   │   │       └── ...
│       └── runner.xml     ← Custom Maven Assembly descriptor for executable JAR
├── test/
```

**Key Convention: TIGER = Modern Code**

- **`tiger` package**: All new features, refactored code, and modern implementations
- **Other packages** (e.g., `presentation`): Legacy code maintained for backward compatibility

**When to use**:
- Check `tiger/` for new feature development and modern implementations
- Check other packages for legacy code and existing functionality

#### 2. Legacy Location: `WebRoot/`
Pre-Maven Eclipse project structure (retained for backward compatibility):
```
PasPresentation/WebRoot/
├── WEB-INF/
│   ├── lib/               ← Legacy JAR dependencies (system-scoped in pom.xml)
│   │   ├── spring-*.jar
│   ├── nscSecurityContext.xml    ← Spring Security LDAP configuration
│   ├── applicationContext.xml    ← Spring application context
│   └── web.xml                   ← Servlet descriptor
├── css/                   ← Stylesheets
├── images/                ← Application images
├── js/                    ← JavaScript files
└── *.jsp                  ← JSP template files
```

**When to use**: Check this directory for JSP pages, web configuration, and legacy framework JARs.

### Finding Code in Dual Structure

| To Find | Location | Modern/Legacy |
|---------|----------|----------------|
| **New feature code** | `src/main/java/com/mpqh/opas/tiger/pas/presentation/...` | **MODERN ✓** |
| **Modern beans** | `src/main/java/com/mpqh/opas/tiger/pas/presentation/beans/` | **MODERN ✓** |
| **Modern services** | `src/main/java/com/mpqh/opas/tiger/pas/presentation/services/` | **MODERN ✓** |
| **Legacy code** | `src/main/java/com/mpqh/pas/presentation/...` | Legacy |
| **Spring config** | `WebRoot/WEB-INF/*.xml` | Legacy |
| **JSP pages** | `WebRoot/*.jsp` or `WebRoot/include/` | Legacy |
| **CSS/JavaScript** | `WebRoot/css/` and `WebRoot/js/` | Mixed |
| **Test code** | `src/test/java/` | Current |
| **Legacy JARs** | `WebRoot/WEB-INF/lib/` | Legacy |

### Development Guidelines: Tiger = Modern Code

**For all new development:**

1. **Create code in the `tiger` package:**
   ```
   src/main/java/com/mpqh/opas/tiger/pas/presentation/[feature]/YourNewClass.java
   ```

2. **Organize by feature or domain:**
   ├── controllers/    ← REST/servlet controllers (if needed)
   ├── utils/          ← Utility functions
   └── models/         ← Data models/entities
   ```

3. **Keep legacy code isolated:**
   - Do not modify classes in pre-Tiger packages (e.g., `presentation`)
   - If refactoring legacy code, create a new tiger equivalent and deprecate the legacy version

4. **Configuration for Tiger code:**
   - Add new Spring beans in `src/main/resources/` (Maven standard location)
   - Keep legacy Spring XML config in `WebRoot/WEB-INF/` for backward compatibility

**Example: Adding a new feature**

**Before (Legacy approach):**
```java
// Location: src/main/java/com/mpqh/pas/presentation/beans/NewFeatureBean.java
package com.mpqh.pas.presentation.beans;
// [legacy code structure]
```

**After (Tiger/Modern approach):**
```java
// Location: src/main/java/com/mpqh/opas/tiger/pas/presentation/beans/NewFeatureBean.java
package com.mpqh.opas.tiger.pas.presentation.beans;
// [modern code structure, cleaner architecture]
```

### How Maven Handles Both Locations

**PasPresentation/pom.xml configuration:**
```xml
<!-- Maven location for Java source compilation -->
<source>${project.basedir}/src</source>

<!-- Legacy location for web resources and JSPs -->
<warSourceDirectory>${project.basedir}/WebRoot</warSourceDirectory>

<!-- Legacy JAR dependencies -->
<systemPath>${project.basedir}/WebRoot/WEB-INF/lib/spring-core-4.1.0.RELEASE.jar</systemPath>
```

**Build process:**
1. Maven compiles `src/main/java/` → `target/classes/`
2. Combines `target/classes/` + `WebRoot/` → `pas-presentation.war`
3. For executable JAR: unpacks both + embedded Tomcat → `pas-presentation-26.0.0-SNAPSHOT-runner.jar`
4. For external Tomcat: copies both `src/` classes and `WebRoot/` to Tomcat webapps

### Finding Your Tomcat Installation

Use these commands to locate Tomcat's webapps directory:

**macOS (Homebrew):**
```bash
brew --cellar tomcat
# Example output: /usr/local/Cellar/tomcat/9.0.89/libexec/webapps
```

**macOS (Manual Installation):**
```bash
ls -la ~/Library/Tomcat/webapps  # Common location
# or find it:
find /Library -type d -name "tomcat" 2>/dev/null
```

**Linux:**
```bash
# Common locations:
ls /opt/tomcat/webapps
ls /usr/local/tomcat/webapps
ls /var/lib/tomcat/webapps

# or check CATALINA_HOME:
echo $CATALINA_HOME
```

**Windows (cmd.exe):**
```cmd
dir "C:\Program Files\Apache Software Foundation"
# or PowerShell:
Get-ChildItem "C:\Program Files" -Filter "*Tomcat*"
```

---

## Deployment Path 1: Executable JAR (Embedded Tomcat 9)

### Build the JAR

**From the OPAS root directory:**

```bash
mvn -Ptiger clean package
```

**Output:**
- `PasPresentation/target/pas-presentation-26.0.0-SNAPSHOT-runner.jar` (executable JAR)
- `PasPresentation/target/pas-presentation.war` (standard WAR, also created)

### Run the JAR

#### Setup Environment Variables

**macOS/Linux:**
```bash
export OPAS_DB_URL='jdbc:oracle:thin:@[DATABASE IP]:1521:[DB SCHEMA]'
export OPAS_DB_USERNAME='[PAS APP USERNAME]'
export OPAS_DB_PASSWORD='[PASSWORD]'
```

**Windows (PowerShell):**
```powershell
$env:OPAS_DB_URL = "jdbc:oracle:thin:@[DATABASE IP]:1521:[DB SCHEMA]"
$env:OPAS_DB_USERNAME = "[PAS APP USERNAME]"
$env:OPAS_DB_PASSWORD = "[PASSWORD]"
```

**Windows (cmd.exe):**
```cmd
set OPAS_DB_URL=jdbc:oracle:thin:@[DATABASE IP]:1521:[DB SCHEMA]
set OPAS_DB_USERNAME=[PAS APP USERNAME]
set OPAS_DB_PASSWORD=[PASSWORD]
```

#### Execute from OPAS Root Directory

**macOS/Linux:**
```bash
cd /path/to/OPAS
java -Dopas.contextPath=/pas-presentation \
     -Dopas.docBase=$(pwd)/PasPresentation/target/pas-presentation \
  -jar PasPresentation/target/pas-presentation-26.0.0-SNAPSHOT-runner.jar
```

**Windows (PowerShell):**
```powershell
cd C:\path\to\OPAS  # or whatever your OPAS directory is
$projectDir = (Get-Location).Path
java -Dopas.contextPath=/pas-presentation `
     -Dopas.docBase="$projectDir\PasPresentation\target\pas-presentation" `
  -jar PasPresentation\target\pas-presentation-26.0.0-SNAPSHOT-runner.jar
```

**Windows (Git Bash):**
```bash
cd C:/path/to/OPAS
# Git Bash requires MSYS2_ARG_CONV_EXCL to prevent path rewriting
MSYS2_ARG_CONV_EXCL='*' java -Dopas.contextPath=/pas-presentation \
     -Dopas.docBase="$(pwd -W | sed 's/\//\\/g')/PasPresentation/target/pas-presentation" \
  -jar PasPresentation/target/pas-presentation-26.0.0-SNAPSHOT-runner.jar
```

**Windows (cmd.exe):**
```cmd
cd C:\path\to\OPAS
set DOCBASE=%CD%\PasPresentation\target\pas-presentation
java -Dopas.contextPath=/pas-presentation -Dopas.docBase=%DOCBASE% -jar PasPresentation\target\pas-presentation-26.0.0-SNAPSHOT-runner.jar
```

### Verify Startup
Look for these success indicators in the console log:

```
INFO org.apache.coyote.http11.Http11NioProtocol: Initializing ProtocolHandler ["http-nio-8080"]
INFO org.apache.catalina.startup.Catalina: Server startup in [XX] ms
```

**Access the application:**
- URL: `http://localhost:8080/pas-presentation`
- Login: Use LDAP credentials (if on VPN) or test credentials if LDAP bypassed

---

## Deployment Path 2: External Tomcat 9 Deploy

### Prerequisites

This deployment method copies compiled classes directly to a Tomcat installation. 

**Step 1: Identify your Tomcat location:**

- macOS (Homebrew): `/usr/local/Cellar/tomcat/9.0.x/libexec/webapps`
- macOS (manual): `/Library/Tomcat/webapps` or your custom location
- Linux: `/opt/tomcat/webapps`, `/usr/local/tomcat/webapps`, or custom
- Windows: `C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps` or custom

**Step 2: Verify Tomcat path exists:**

```bash
# macOS/Linux
ls -la /path/to/tomcat/webapps

# Windows
dir C:\path\to\tomcat\webapps
```

**Step 3: Ensure Tomcat is NOT running during deployment**

### Build and Deploy

The build uses a configurable `tomcat.deploy.path` property. Override it with your actual Tomcat installation path:

**macOS/Linux:**
```bash
mvn -pl PasPresentation -am package \
  -DexternalTomcatDeploy \
  -Dtomcat.deploy.path=/usr/local/Cellar/tomcat/9.0.x/libexec/webapps \
  -DskipTests
```

**Windows (cmd.exe):**
```cmd
mvn -pl PasPresentation -am package -DexternalTomcatDeploy -Dtomcat.deploy.path="C:\Tomcat 9.0\webapps" -DskipTests
```

**Windows (PowerShell):**
```powershell
mvn -pl PasPresentation -am package `
  -DexternalTomcatDeploy `
  -Dtomcat.deploy.path="C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps" `
  -DskipTests
```

**What this does:**
1. Builds PasPresentation and dependencies (PasBusiness, PasIntegration)
2. Copies compiled WebRoot files to `$TOMCAT_PATH/pas-presentation/`
3. Copies compiled classes to `$TOMCAT_PATH/pas-presentation/WEB-INF/classes/`

**If successful, you should see:**
```
[INFO] Copying resources to .../webapps/pas-presentation
[INFO] BUILD SUCCESS
```

### Persist Your Tomcat Path (Optional)

To avoid repeating `-Dtomcat.deploy.path` every time, create a Maven settings profile:

**Edit:** `~/.m2/settings.xml` (macOS/Linux) or `%USERPROFILE%\.m2\settings.xml` (Windows)

```xml
<profile>
  <id>local-tomcat</id>
  <properties>
    <tomcat.deploy.path>/usr/local/Cellar/tomcat/9.0.x/libexec/webapps</tomcat.deploy.path>
  </properties>
</profile>

<activeProfiles>
  <activeProfile>local-tomcat</activeProfile>
</activeProfiles>
```

Then build without the property:
```bash
mvn -pl PasPresentation -am package -DexternalTomcatDeploy -DskipTests
```

### Configure Application Datasource

Application uses JDBC datasource. Database credentials are read from environment variables during startup. Ensure your environment is set before starting Tomcat (see Prerequisites section above).

### Start Tomcat

**macOS/Linux:**
```bash
# If CATALINA_HOME is set:
$CATALINA_HOME/bin/catalina.sh start

# Otherwise, from your Tomcat installation:
/usr/local/Cellar/tomcat/9.0.x/bin/catalina.sh start
```

**Windows (cmd.exe):**
```cmd
C:\Program Files\Apache Software Foundation\Tomcat 9.0\bin\startup.bat
```

**Windows (PowerShell):**
```powershell
& 'C:\Program Files\Apache Software Foundation\Tomcat 9.0\bin\startup.bat'
```

### Verify Deployment

- Tomcat console should show: `Deployment of web application directory [pas-presentation] has finished`
- Check Tomcat webapps directory contains: `pas-presentation/` with subdirectories
- Access: `http://localhost:8080/pas-presentation`

### Troubleshooting External Tomcat Deploy

**Issue:** `[ERROR] Cannot create resource output directory`
- **Cause:** Path does not exist or incorrect
- **Solution:** Verify path with `-Dtomcat.deploy.path` matches actual Tomcat location

**Issue:** Files not deployed
- **Cause:** Wrong path or Tomcat in use during build
- **Solution:** Stop Tomcat first, re-run build, verify `-Dtomcat.deploy.path`

---

## Deployment Path 3: Manual WAR Deployment

### Build the WAR

**From OPAS root directory:**

```bash
mvn -Ptiger clean package
```

**Output:** `PasPresentation/target/pas-presentation.war`

### Deploy to Tomcat

**Step 1: Stop Tomcat** (if running)

macOS/Linux:
```bash
$CATALINA_HOME/bin/catalina.sh stop
```

Windows (cmd.exe):
```cmd
C:\Program Files\Apache Software Foundation\Tomcat 9.0\bin\shutdown.bat
```

**Step 2: Copy WAR file to webapps**

**macOS/Linux:**
```bash
cp PasPresentation/target/pas-presentation.war $CATALINA_HOME/webapps/
```

**Windows (cmd.exe):**
```cmd
copy PasPresentation\target\pas-presentation.war C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps\
```

**Windows (PowerShell):**
```powershell
Copy-Item PasPresentation\target\pas-presentation.war -Destination 'C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps\'
```

**Step 3: (Optional) Rename WAR for custom context path**

If you want a different context path than `pas-presentation`, rename the WAR before starting Tomcat.

**macOS/Linux:**
```bash
mv $CATALINA_HOME/webapps/pas-presentation.war $CATALINA_HOME/webapps/myapp.war
# Now accessible at http://localhost:8080/myapp
```

**Windows (cmd.exe):**
```cmd
ren "C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps\pas-presentation.war" "myapp.war"
```

**Windows (PowerShell):**
```powershell
Rename-Item 'C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps\pas-presentation.war' 'myapp.war'
```

**Step 4: Start Tomcat**

**macOS/Linux:**
```bash
$CATALINA_HOME/bin/catalina.sh start
```

**Windows (cmd.exe):**
```cmd
C:\Program Files\Apache Software Foundation\Tomcat 9.0\bin\startup.bat
```

**Windows (PowerShell):**
```powershell
& 'C:\Program Files\Apache Software Foundation\Tomcat 9.0\bin\startup.bat'
```

**Step 5: Verify**

- Tomcat will extract the WAR during startup
- Access: `http://localhost:8080/pas-presentation` (or your custom name)

---

## Configuration

### Database Connection
Database credentials are passed via environment variables:
- `OPAS_DB_URL` – JDBC connection string
- `OPAS_DB_USERNAME` – Oracle username
- `OPAS_DB_PASSWORD` – Oracle password

These are read by the application during Spring context initialization and pooled via Tomcat DBCP2 BasicDataSourceFactory (configured in context.xml).

### LDAP Authentication
LDAP configuration is hardcoded in:
- `PasPresentation/WebRoot/WEB-INF/nscSecurityContext.xml` (Spring Security LDAP provider)
- Current: `ldap://marspd3ap:1389/dc=mars,dc=mpqhf,dc=org`

**VPN Required:** The LDAP server is only accessible from the corporate network. If not on VPN, authentication will fail with `UnknownHostException: marspd3ap`.

### Application Properties
- Context path: `/pas-presentation` (set via `-Dopas.contextPath=`)
- Document base: `PasPresentation/target/pas-presentation` (set via `-Dopas.docBase=`)

---

## Troubleshooting

### Issue: `ORA-01017: invalid username/password; logon denied`
**Cause:** Incorrect Oracle credentials or account locked
- Verify `OPAS_DB_USERNAME` and `OPAS_DB_PASSWORD` environment variables
- Check Oracle account status with DBA
- Confirm connection string (`OPAS_DB_URL`) points to active database

### Issue: `UnknownHostException: marspd3ap`
**Cause:** LDAP server not reachable (VPN disconnected or local network)
- Connect to corporate VPN
- Verify VPN is active: `nslookup marspd3ap` should resolve to an IP
- If not on VPN, LDAP authentication will always fail

### Issue: `NoSuchMethodError: 'java.lang.ClassLoader javax.servlet.ServletContext.getClassLoader()'`
**Cause:** Legacy servlet API JAR conflicts with Tomcat 9
- Ensure `packagingExcludes` in pom.xml strips: `servlet-api.jar`, `jsp-api-2.0.jar`, `el-api.jar`, `el-api-2.2.jar`
- Clean and rebuild: `mvn -Ptiger clean package`

### Issue: MSYS2 Path Rewriting (Git Bash only)
**Symptom:** Paths like `/pas-presentation` converted to `C:/Program Files/Git/pas-presentation`
**Solution:** Use `MSYS2_ARG_CONV_EXCL='*'` environment variable when invoking java command
```bash
MSYS2_ARG_CONV_EXCL='*' java -jar ...
```

### Issue: Port 8080 Already in Use
**Cause:** Another application is using the port
- Find process: `netstat -ano | findstr :8080` (Windows)
- Kill process or start on different port: `-Dopas.port=8081`

### Issue: Out of Memory
**Solution:** Increase heap size:
```bash
java -Xmx1024m -jar pas-presentation-26.0.0-SNAPSHOT-runner.jar
```

---

## Build Artifacts

### Standard Outputs
| File | Purpose | Path |
|------|---------|------|
| pas-presentation.war | Standard WAR for any Tomcat | `PasPresentation/target/` |
| pas-presentation-26.0.0-SNAPSHOT-runner.jar | Executable JAR with embedded Tomcat | `PasPresentation/target/` |

### Source Configuration
| File | Purpose |
|------|---------|
| `PasPresentation/pom.xml` | Maven config; maven-assembly-plugin creates runner JAR |
| `PasPresentation/src/assembly/runner.xml` | Assembly descriptor for executable JAR |
| `PasPresentation/WebRoot/META-INF/context.xml` | Tomcat context + datasource config |
| `PasPresentation/WebRoot/WEB-INF/nscSecurityContext.xml` | Spring Security LDAP configuration |

---

## Quick Start Summary

**All commands run from OPAS root directory**

### Executable JAR (Fastest for Development)

```bash
# 1. Build
mvn -Ptiger clean package

# 2. Run (macOS/Linux)
java -Dopas.contextPath=/pas-presentation \
     -Dopas.docBase=$(pwd)/PasPresentation/target/pas-presentation \
  -jar PasPresentation/target/pas-presentation-26.0.0-SNAPSHOT-runner.jar

# 2. Run (Windows PowerShell)
$projectDir = (Get-Location).Path
java -Dopas.contextPath=/pas-presentation `
     -Dopas.docBase="$projectDir\PasPresentation\target\pas-presentation" `
  -jar PasPresentation\target\pas-presentation-26.0.0-SNAPSHOT-runner.jar
```

### External Tomcat (Production-like)

```bash
# 1. Find your Tomcat webapps path (see deployment section)

# 2. Build and deploy with your Tomcat path
# macOS/Linux:
mvn -pl PasPresentation -am package \
  -DexternalTomcatDeploy \
  -Dtomcat.deploy.path=/usr/local/Cellar/tomcat/9.0.x/libexec/webapps \
  -DskipTests

# Windows:
mvn -pl PasPresentation -am package -DexternalTomcatDeploy ^
  -Dtomcat.deploy.path="C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps" -DskipTests

# 3. Start Tomcat from your TOMCAT_HOME
$CATALINA_HOME/bin/catalina.sh start     # macOS/Linux
# or C:\...\Tomcat 9.0\bin\startup.bat   # Windows
```

### Manual WAR

```bash
# 1. Build
mvn -Ptiger clean package

# 2. Stop Tomcat
$CATALINA_HOME/bin/catalina.sh stop

# 3. Copy
cp PasPresentation/target/pas-presentation.war $CATALINA_HOME/webapps/

# 4. Start Tomcat
$CATALINA_HOME/bin/catalina.sh start
```

---

## Cross-Platform Compatibility

This section confirms all deployment paths work across macOS, Linux, and Windows.

### What We Fixed

✅ **Removed hardcoded user paths** – No more `C:\Users\dapplegate\...`
✅ **Removed OS-specific Tomcat paths** – All paths are now configurable properties
✅ **Parameterized database URLs** – Use environment variables, not hardcoded IPs
✅ **Platform-agnostic build commands** – Maven works identically on all platforms
✅ **Cross-platform shell examples** – Separate examples for Bash, PowerShell, cmd.exe

### Configurable Properties

| Property | Default | Override | Usage |
|----------|---------|----------|-------|
| `tomcat.deploy.path` | `C:/Program Files/.../Tomcat 9.0/webapps` | `-Dtomcat.deploy.path=/your/path` | External Tomcat deploy profile |
| `OPAS_DB_URL` | (none) | `export OPAS_DB_URL=...` | Database connection |
| `OPAS_DB_USERNAME` | (none) | `export OPAS_DB_USERNAME=...` | Database login |
| `OPAS_DB_PASSWORD` | (none) | `export OPAS_DB_PASSWORD=...` | Database password |

### Platform-Specific Notes

**macOS:**
- Homebrew Tomcat: `/usr/local/Cellar/tomcat/9.0.x/libexec/webapps`
- Use `$(pwd)` for absolute paths in commands
- Shells: bash, zsh

**Linux:**
- Tomcat commonly at: `/opt/tomcat/webapps` or `/usr/local/tomcat/webapps`
- Use `$(pwd)` for absolute paths
- Shells: bash, sh, zsh

**Windows:**
- Path separator: `\` in cmd.exe, `/` preferred in Maven/Git Bash
- Forward slashes `/` work in Maven properties
- Tomcat default: `C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps`
- Shells: cmd.exe, PowerShell, Git Bash

**Git Bash on Windows:**
- MSYS2 rewrites Unix paths to Windows paths
- Use `MSYS2_ARG_CONV_EXCL='*'` to disable path rewriting (only for embedded JAR)
- For Tomcat paths, use forward slashes and let Maven handle conversion

### Sharing Configuration

To share your configuration with team members:

1. **Document your Tomcat path:**
   ```
   Your Tomcat location: /usr/local/Cellar/tomcat/9.0.89/libexec/webapps
   ```

2. **Share build command:**
   ```bash
   mvn -pl PasPresentation -am package \
     -DexternalTomcatDeploy \
     -Dtomcat.deploy.path=/usr/local/Cellar/tomcat/9.0.89/libexec/webapps \
     -DskipTests
   ```

3. **Or create team `.m2/settings.xml` profile** (see External Tomcat section above)

### Testing Cross-Platform Compatibility

When deploying to a new machine:

1. **Verify prerequisites:**
   ```bash
   mvn --version    # Should work
   java -version    # Should work
   ```

2. **Test build (no deployment):**
   ```bash
  mvn -Ptiger clean package    # Should succeed
   ```

3. **Test JAR deployment:**
   ```bash
  java -jar PasPresentation/target/pas-presentation-26.0.0-SNAPSHOT-runner.jar
   ```

4. **Only then use external Tomcat** with `-Dtomcat.deploy.path`

To auto-detect your OS and run appropriate commands:

**macOS/Linux Bash:**
```bash
#!/bin/bash
OS=$(uname -s)
TOMCAT_PATH=${CATALINA_HOME:-/usr/local/Cellar/tomcat/9.0.x}

case "$OS" in
  Darwin|Linux)
    echo "Detected: $OS"
    java -Dopas.contextPath=/pas-presentation \
         -Dopas.docBase=$(pwd)/PasPresentation/target/pas-presentation \
         -jar PasPresentation/target/pas-presentation-26.0.0-SNAPSHOT-runner.jar
    ;;
  *)
    echo "Unsupported OS: $OS"
    ;;
esac
```

**Windows PowerShell:**
```powershell
$projectDir = (Get-Location).Path
$jarPath = "$projectDir\PasPresentation\target\pas-presentation-26.0.0-SNAPSHOT-runner.jar"

if (Test-Path $jarPath) {
    Write-Host "Running from $projectDir"
    java -Dopas.contextPath=/pas-presentation `
         -Dopas.docBase="$projectDir\PasPresentation\target\pas-presentation" `
         -jar $jarPath
} else {
  Write-Error "JAR not found at $jarPath. Did you run 'mvn -Ptiger clean package'?"
}
```

---

## Local Secret Setup (VS Code)

Use this setup to keep database credentials out of tracked config files.

### Files

- Tracked template: `/opas.profile.env.example`
- Local secret file used by debug launch: `/.vscode/opas.profile.local.env`

### One-Time Setup

1. Copy template to local file:

```bash
cp opas.profile.env.example .vscode/opas.profile.local.env
```

Windows PowerShell:

```powershell
Copy-Item opas.profile.env.example .vscode\opas.profile.local.env
```

2. Edit `/.vscode/opas.profile.local.env` and replace placeholders:

```text
OPAS_DB_URL=jdbc:oracle:thin:@[DATABASE IP]:1521:[DB SCHEMA]
OPAS_DB_USERNAME=[PAS APP USERNAME]
OPAS_DB_PASSWORD=[PASSWORD]
```

3. Reload VS Code window.

### Verification

Run `OPASMain - Launch Embedded Tomcat (Use This)` from Run and Debug.

If launch command output still shows placeholder text like `[PASSWORD]`, the local secret file was not updated or not reloaded.

### Security Rules

- Never put real credentials in `.vscode/launch.json`.
- Keep real credentials only in `/.vscode/opas.profile.local.env`.
- Rotate credentials if they were ever committed or pasted into chat/terminal history.

---

## VS Code Run and Debug with OPASMain

Use this when you want to run embedded Tomcat from VS Code and debug Java code in the OPAS process.

### Prerequisites

1. Java Extension Pack installed in VS Code
2. Project built at least once:
```bash
mvn -Ptiger -pl PasPresentation -am compile
```
3. Database env values ready (use your local values):
- `OPAS_DB_URL`
- `OPAS_DB_USERNAME`
- `OPAS_DB_PASSWORD`

### Option 1: Run/Debug OPASMain directly (recommended)

Create or update `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "java",
      "name": "OPASMain (Embedded Tomcat)",
      "request": "launch",
      "mainClass": "com.mpqh.opas.tiger.pas.presentation.OPASMain",
      "projectName": "pas-presentation",
      "cwd": "${workspaceFolder}",
      "vmArgs": "-Dopas.contextPath=/pas-presentation -Dopas.docBase=${workspaceFolder}/PasPresentation/target/pas-presentation",
      "envFile": "${workspaceFolder}/.vscode/opas.profile.local.env",
      "env": {
        "MSYS2_ARG_CONV_EXCL": "*"
      },
      "console": "integratedTerminal"
    }
  ]
}
```

How to use:

1. Set breakpoints in Java classes (services/beans/controllers).
2. Open Run and Debug in VS Code.
3. Select `OPASMain (Embedded Tomcat)`.
4. Press F5.
5. Browse to `http://localhost:8080/pas-presentation`.

### Option 2: Attach debugger to a running jar process

Start OPASMain in terminal with remote debug enabled:

```bash
OPAS_DB_URL='jdbc:oracle:thin:@[DATABASE IP]:1521:[DB SCHEMA]' \
OPAS_DB_USERNAME='[PAS APP USERNAME]' \
OPAS_DB_PASSWORD='[PASSWORD]' \
java -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=5005 \
  -Dopas.contextPath=/pas-presentation \
  -Dopas.docBase=PasPresentation/target/pas-presentation \
  -jar PasPresentation/target/pas-presentation-26.0.0-SNAPSHOT-runner.jar
```

Then add an attach config in `.vscode/launch.json`:

```json
{
  "type": "java",
  "name": "Attach to OPASMain (5005)",
  "request": "attach",
  "hostName": "localhost",
  "port": 5005
}
```

### Debugging Notes

- Entry point class is `com.mpqh.opas.tiger.pas.presentation.OPASMain`.
- Embedded runtime files are created under `target/embedded-tomcat/`.
- If breakpoints are not hit, run `mvn clean compile` and restart debug.
- If startup fails with LDAP host errors, confirm VPN access to `marspd3ap`.
- If startup fails with DB auth/connect errors, verify `OPAS_DB_*` values.

---

## See Also

- **[OPAS_BUILD_REFERENCE.md](OPAS_BUILD_REFERENCE.md)** — Structured reference for build automation, environment variables, scripting patterns, and CI/CD integration
