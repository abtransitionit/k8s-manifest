# Contributing to This Project

Thank you for considering contributing! Here are a few guidelines to follow:

## Reporting Issues

- Use [GitHub Issues](../../issues) to report bugs.
- Provide a clear title and description.
- Include steps to reproduce, expected vs. actual behavior.

## Suggesting Features

- Open an issue and label it as a **feature request**.
- Describe your idea clearly and explain the value it would bring.

## Making Changes

- Fork the repository.
- Create a new branch for your feature/fix.
- Follow consistent coding style (if any).
- Test your changes if applicable.

## Submitting a Pull Request

- Ensure your changes are properly described in the PR.
- Reference related issues in your pull request description.
- Be kind and respectful in your communication.

---

## Create a Release
- This project adopts a release workflow adapted from [RootlessKit](https://github.com/rootless-containers/rootlesskit), implemented via [GitHub Actions](https://github.com/rootless-containers/rootlesskit/actions/runs/14899716761/workflow). 
- Source Implementation :`release.yml` Workflow from [RootlessKit](https://github.com/rootless-containers/rootlesskit/blob/master/.github/workflows/release.yml).

**Key Steps**:
1. **Trigger**:   Push a version tag (e.g., `v1.0.0`) initiates the workflow  
2. **Artifacts**: Automatically published to [GitHub Releases](https://github.com/your/repo/releases)  
3. **Changelog**: Manually update [`CHANGELOG.md`](CHANGELOG.md) pre-release  


### Step 1: Create Version Tag
Run these commands:
```bash
# commit changes to CHANGELOG.md
git add CHANGELOG.md
git commit -m "Prepare v1.0.0 release"

# create a TAG
git tag -a v1.0.0 -m "Release v1.0.0"

# push to main
git push origin v1.0.0
```

### Step 2: Automatic Release
Let GitHub Actions Do the Rest. The workflow (using GitHub Actions) will automatically:  
 ✓ Build the project  
 ✓ Create a GitHub Release  
 ✓ Create a Release page  
 ✓ Upload the files or assets

*(Find your release under "Releases" in your repo sidebar)*

### Step 3: Update Changes
Edit `CHANGELOG.md` following this format:
```markdown
## [v1.0.0] - 2025-05-20
### Added
- New feature X
### Fixed
- Bug in module Y
```

### 🚨 Troubleshooting
If the release fails:
1. Check [Actions tab](https://docs.github.com/assets/cb-33827/images/help/repository/actions-tab.png) for errors  
2. Verify your tag matches the version in `CHANGELOG.md`  

### Example Release
See how it should look:  
[v1.0.0 Demo Release](https://github.com/rootless-containers/rootlesskit/releases/tag/v1.0.0)


## Additional Notes

- Read our [Code of Conduct](CODE_OF_CONDUCT.md).
- All contributions must follow the [license terms](../LICENSE).
