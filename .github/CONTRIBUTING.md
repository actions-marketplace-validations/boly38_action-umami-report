# How to contribute
You're not a dev ? just submit an issue (bug, improvements, questions).
* you could also fork, feature branch, then submit a pull request.

## HowTo setup

* Install Node.js and pnpm
* Clone
* Install deps
* setup your test environment (cf. [initenv.template.sh](./env/initenv.template.sh))

````bash
git clone https://github.com/boly38/action-umami-report.git
cd action-umami-report
pnpm install
# setup test env
cp ./env/initenv.template.sh ./env/initenv.dontpush.sh
# edit ./env/initenv.dontpush.sh then source it
. ./env/initenv.dontpush.sh
````

## HowTo test
* To run manual test

````bash
. ./env/initenv.dontpush.sh
pnpm day
pnpm showResults
# check other targets in package.json
pnpm run
````

* To run mocha test

````bash
. ./env/initenv.dontpush.sh
# run all tests
pnpm test
# run all tests with code coverage
pnpm ci-test
# run a single test file
tst=1_sinon_cloud_umami pnpm tst
````


## HowTo locally test GitHub action

* To run GitHub action workflow locally, cf [TESTING_ACTION](./TESTING_ACTION.md) 
* For GitHub workflows configuration and required secrets, cf [workflows/README](./workflows/README.md) 



## HowTo reproduce package

Packaging is defined in `.github/scripts/package.sh` 
````bash
pnpm install -g @vercel/ncc
ncc build index.js
# package result under dist/
````

## HowTo publish a new umami-server compatible branch ?

### Understanding the packaging mechanism

This project uses **orphan branches** to distribute packaged versions of the action (with all dependencies bundled in `dist/`).

**Two types of orphan branches exist:**

1. **`main-version`** - Auto-generated from the `main` branch
   - Used for QA and testing the latest development version
   - Always points to the latest packaged version of `main`

2. **`umami-server-X.Y.Z`** - Version-specific branches (defined in `package.json::orphanBranch`)
   - Used for stable releases compatible with specific Umami server versions
   - Example: `umami-server-2.17.0` for Umami v2.17.x compatibility

### The `to-package` branch mechanism

To trigger packaging, use the **`to-package` branch** as a pointer:

```bash
# To package the current main branch → creates/updates 'main-version'
git checkout main
git branch -f to-package
git push origin to-package --force

# To package a specific version → creates/updates orphan branch from package.json
git checkout v6.0.2
git branch -f to-package
git push origin to-package --force
```

**What happens when you push `to-package`:**

1. Workflow `.github/workflows/main_ci_and_package_action.yml` is triggered with `MUST_BE_PACKAGED=true`
2. The workflow determines the target orphan branch name:
   - If source is `main` → `ORPHAN_BRANCH=main-version`
   - Otherwise → `ORPHAN_BRANCH=$(jq -r .orphanBranch package.json)`
3. The code is packaged using `@vercel/ncc` → all deps bundled in `dist/`
4. A tag `last-<ORPHAN_BRANCH>` is created/updated to mark the source commit
5. The `dist/` content is force-pushed to the orphan branch

### Publishing a new version-specific branch

**Step-by-step process:**

1. **Update `package.json::orphanBranch`** to match the target Umami server version
   ```json
   "orphanBranch": "umami-server-2.17.0"
   ```

2. **Create a version** using patch/minor/major:
   ```bash
   pnpm version patch  # or minor/major
   git push --follow-tags
   ```

3. **Point `to-package` to the version you want to publish:**
   ```bash
   git checkout v6.0.2
   git branch -f to-package
   git push origin to-package --force
   ```

4. **Verify the workflow execution** in GitHub Actions
   - Check that `main_ci_and_package_action.yml` completes successfully
   - Verify the orphan branch was updated: `git fetch && git log origin/umami-server-2.17.0`
   - Verify the tag was created: `git fetch --tags && git show last-umami-server-2.17.0`

**Example:**
The `umami-server-2.17.0` dist orphan branch uses `last-umami-server-2.17.0` tag as its source reference.

## HowTo create a fresh version
- install GitHub client
- create automatically a draft release version using [gh client](https://cli.github.com/)

Example to create v6.0.1
```bash
gh release create v6.0.1 --draft --generate-notes
```

this will make a new draft release. Verify it in [releases list](https://github.com/boly38/action-mongo-tools/releases)

- ⚠️ the repository apply immutable releases since #8, so you can't modify a release once published
- publish the release when ready
