# Workspace

## Motivation

Without a workspace, multi-repo development is a sprawl of independent clones. Each clone is an island: no shared conventions, no shared documentation, no way to see all active work at a glance. When you start a new project, you clone into wherever makes sense that day. When you return a month later, you reconstruct context from scratch. AI assistants face the same problem: every session starts cold, with no record of how this collection of repos is supposed to work together.

A workspace solves this by giving the collection of repositories a home.

## Mental Model

A workspace is a single git repository that acts as the organizational hub for all your development work. It contains:

- References to external repositories (tracked repositories), stored as bare clones
- Isolated working areas for each branch you are actively developing
- Documentation, conventions, and work tracking that apply across all tracked repositories

The workspace itself is a versioned git repository. That versioning matters: conventions and documentation have history, can be reviewed, and can be shared. When you change how the workspace operates, you can see when that happened and why.

The key structural idea is that external repositories are not cloned in the usual sense. They are tracked as bare clones with working areas created per branch. Each branch gets its own directory with a full checkout of that branch's state. Multiple branches can be active simultaneously without any stashing or switching. Git worktrees are the recommended mechanism for this, though any approach that provides branch isolation satisfies the requirement.

The workspace is where you `cd` to start working. Not into a specific repository, not into a specific branch directory. The workspace is the entry point, and navigation from there is a matter of convention.

## Capabilities

A workspace implementation supports:

- **Adding a tracked repository**: Bring an external repository under the workspace's management. After this, the workspace knows the repository exists, where to find it, and where working areas for it live.
- **Creating a working area**: For any tracked repository and any branch, create an isolated checkout that does not interfere with other branches. Start development immediately without disturbing other active working areas.
- **Listing and navigating**: See all tracked repositories and all active working areas across the entire workspace at once. Move between them without hunting for paths or reconstructing state.
- **Removing a repository or working area**: Clean up when a branch is merged and its working area is no longer needed, or stop tracking a repository entirely.

## In Practice

Say you work on three services: an API, a frontend, and a shared library. You add all three to your workspace. The result looks like this:

```
workspace/
  repos/
    api/
      bare.git/          # bare clone of the API repo
      worktrees/
        main/            # working area for the main branch
        gt-pw2k--add-oauth/  # working area for a feature branch
    frontend/
      bare.git/
      worktrees/
        main/
    shared-lib/
      bare.git/
      worktrees/
        main/
  docs/                  # conventions and documentation for the workspace
  projects/              # cross-repo project folders
  .claude/               # AI assistant configuration
```

When you need to start a feature on the API, you create a working area for a new branch. You get a fresh directory at `repos/api/worktrees/your-branch/` with a full checkout. The `main` working area in `repos/api/worktrees/main/` is untouched. You can have a PR open on the API, another on the shared library, and a hotfix in progress, all simultaneously, all in separate directories, with no context switching overhead.

The workspace documentation and AI conventions travel with all of this. Whatever rules and context you have written into the workspace apply every time an AI assistant starts a session. You write them once; they apply everywhere.

## Design Considerations

**Bare clone plus worktree is the recommended approach**, but the workspace concept does not require it. The key property is branch isolation: multiple branches active simultaneously in separate directories. Implementations could use shallow clones, symlinks, or other mechanisms. The directory layout described above is opinionated, and conforming to it makes tooling and conventions portable across towns.

**Working area naming** carries information. Including the branch name (and optionally a work tracking ID) in the directory name makes it easy to find the right working area without a lookup.

**The workspace directory layout is the convention.** `repos/<name>/` for tracked repositories, `worktrees/<branch>/` for working areas. AI assistants and CLI tools can navigate by convention rather than by querying a database.

**The town is the thing you version and share.** The tracked repositories have their own git histories. The workspace repository captures the organizational layer: what repos are tracked, what conventions apply, what work is in flight. That separation keeps the workspace useful even when the tracked repositories change hands or URLs.
