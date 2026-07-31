# Docker and custom-node operations

This fork builds ComfyUI and its custom nodes into one immutable Docker image.
The authoritative node list is [`custom-nodes.yaml`](../custom-nodes.yaml).
ComfyUI Manager remains available for browsing, but it cannot install, update,
disable, or remove nodes at runtime.

## Architecture

The image contains:

- Python 3.12.7 built with pyenv.
- PyTorch from the CUDA 13.0 (`cu130`) index.
- ComfyUI's `requirements.txt` dependencies.
- Every repository and commit in `custom-nodes.yaml`.
- Each node's root `requirements.txt`, optional manifest `pip` dependencies,
  and optional source patches.

The Compose configuration mounts models, input, and output. It deliberately
does not mount `/srv/app/custom_nodes`; custom nodes come only from the image.
The old `requirements-extra.txt` mechanism has been removed.

The installer is [`scripts/install_custom_nodes.py`](../scripts/install_custom_nodes.py).
For each manifest entry it:

1. Clones the repository.
2. Checks out the exact commit in detached-HEAD mode.
3. Verifies and applies configured patches with `git apply --check` and
   `git apply`.
4. Installs the node's root `requirements.txt`, when present.
5. Installs any additional packages declared in `pip`.

Docker BuildKit cache mounts preserve pip downloads and built wheels between
builds. The cache is local to the Docker builder and can be removed by
`docker builder prune`.

## Manifest format

The minimal entry is a repository and a full Git commit:

```yaml
nodes:
  - repo: https://github.com/example/ComfyUI-Example
    commit: 0123456789abcdef0123456789abcdef01234567
```

Never use a branch name, abbreviated commit, or tag in `commit`. A full commit
keeps builds reproducible even when a repository's default branch changes.

Optional node-specific dependencies and patches are stored with the node:

```yaml
nodes:
  - repo: https://github.com/example/ComfyUI-Example
    commit: 0123456789abcdef0123456789abcdef01234567
    patches:
      - patches/example-local-change.patch
    pip:
      - package-name>=1.2,<2
```

Patch paths are relative to the repository root containing the manifest and
must remain inside it. The node's image directory is derived from the final
component of its repository URL, with a trailing `.git` removed.

## Add a node

Obtain the repository URL and an exact commit. For the current default-branch
commit:

```bash
git ls-remote https://github.com/OWNER/REPOSITORY.git HEAD
```

Add the repository and returned 40-character commit to `custom-nodes.yaml`:

```yaml
  - repo: https://github.com/OWNER/REPOSITORY
    commit: FULL_COMMIT
```

Run the local checks:

```bash
python -m unittest tests-unit/scripts_test/custom_nodes_installer_test.py
python -m py_compile scripts/install_custom_nodes.py
git diff --check
docker compose config >/dev/null
```

Commit and push when ready:

```bash
git add custom-nodes.yaml
git commit -m "Add NAME node"
git push
```

Deploy on the NVIDIA server:

```bash
git pull
docker compose build
docker compose up -d --force-recreate
```

## Record an already-installed node

To identify a Git checkout's exact source and commit:

```bash
sudo git -C /srv/comfyui/custom_nodes/NODE remote get-url origin
sudo git -C /srv/comfyui/custom_nodes/NODE rev-parse HEAD
```

Use the resulting URL and full commit in the manifest. Root-owned repositories
must be inspected with `sudo` to avoid Git's dubious-ownership error.

Standalone files already tracked in this repository's `custom_nodes` directory,
such as `websocket_image_save.py`, are copied into the image by the normal
Docker build and do not need manifest entries.

## Update a node

Resolve the desired new commit and replace only that node's `commit` value.
Then run the checks, build, and deploy as described above.

If the node has patches, the build deliberately fails when a patch no longer
matches the new source. Recreate or refresh the patch against the new commit;
do not bypass the `git apply --check` failure.

ComfyUI Manager itself is pinned in the manifest. When updating it, confirm
that `patches/manager-manifest-only.patch` still applies and that node mutation
operations remain blocked.

## Remove a node

Delete its complete manifest entry, run the checks, rebuild, and recreate the
container. Because `custom_nodes` is not mounted, the removed repository will
not survive from an older container.

Do not delete the old host `/srv/comfyui/custom_nodes` directory merely to
remove a node from the image. It is no longer mounted and may be retained as a
migration backup.

## Add exceptional Python dependencies

The installer automatically installs a node's root `requirements.txt`. Use a
manifest `pip` list only when that file is incomplete or when the image needs a
specific compatibility constraint:

```yaml
    pip:
      - onnxruntime-gpu>=1.27
```

Keep exceptions beside the node that needs them. Do not recreate a global
`requirements-extra.txt`.

The image currently uses CUDA 13.0. CUDA-sensitive packages must match that
major version. For example, ONNX Runtime 1.27 and later use CUDA 13 wheels;
older CUDA 12 installations used an upper bound below 1.27.

## Patch a node

Check out the manifest's exact commit, make the required source change, and
create a patch:

```bash
git -C /path/to/node diff --binary > patches/descriptive-name.patch
```

Add it to the node entry:

```yaml
    patches:
      - patches/descriptive-name.patch
```

Validate against a clean checkout:

```bash
git -C /path/to/clean/node apply --check /path/to/patches/descriptive-name.patch
```

Patches are applied before Python dependencies are installed. Keep patches
small and focused so incompatibility with a future node update is obvious.

Current patches include:

- `reactor-disable-safety-check.patch`, which applies the locally required
  ReActor behavior change.
- `manager-manifest-only.patch`, which enforces image-managed custom nodes.

## ComfyUI Manager behavior

Manager is intentionally browse-only. Its patch rejects node install,
reinstall, uninstall, update, update-all, fix, enable/disable, Git URL install,
and direct pip-install requests before they enter Manager's task queue. The
response tells the user to update `custom-nodes.yaml` and rebuild the image.

Model downloads remain available because models are persistent mounted data,
not image-managed custom-node code.

If Manager gains a new node-mutation endpoint, add it to
`MANIFEST_ONLY_ROUTES` in `patches/manager-manifest-only.patch` and regenerate
the patch against the pinned Manager commit.

## Verify a deployment

Show only logs from the current container run:

```bash
docker compose logs \
  --since "$(docker inspect -f '{{.State.StartedAt}}' "$(docker compose ps -q comfyui)")" \
  comfyui
```

List packaged nodes:

```bash
docker compose exec comfyui ls -la /srv/app/custom_nodes
```

Check the GPU libraries:

```bash
docker compose exec comfyui python -c \
  "import torch, onnxruntime as ort; print(torch.__version__, torch.version.cuda); print(ort.__version__, ort.get_available_providers())"
```

Expected results include CUDA 13 for PyTorch and `CUDAExecutionProvider` for
ONNX Runtime. Test representative workflows after any large ComfyUI sync or
GPU-stack update.

## Troubleshooting

### Node missing from a workflow

Check the current-run logs for `IMPORT FAILED` and the first traceback. A node
can exist in `/srv/app/custom_nodes` but still be unavailable because a Python
or system dependency failed to load.

### Patch fails during build

The pinned node changed or the patch was generated against different source.
Confirm the manifest commit, recreate the patch against that exact commit, and
retain the build-time check.

### Dependency download is slow

The first build populates the BuildKit pip cache. Later builds reuse downloads
and wheels even when the custom-node layer changes. Changing the manifest still
reclones all nodes in that layer. Do not use `docker builder prune` unless the
cache should be discarded.

### Frontend legacy API warnings

Manager and some older custom nodes may import deprecated ComfyUI frontend
APIs. These warnings are non-fatal until upstream removes the compatibility
API. Updating a node only helps if its upstream project has migrated.

### Syncing upstream ComfyUI

Push local fork work first, use GitHub's **Sync fork** action, then pull and
rebuild:

```bash
git pull
docker compose build
docker compose up -d --force-recreate
```

Never choose an option that discards fork changes. If GitHub reports conflicts,
merge upstream locally so Docker and manifest conflicts can be reviewed before
pushing.

## Future Codex checklist

For a request such as “help me add this node: URL, commit SHA”:

1. Read this document and `AGENTS.md`.
2. Confirm the URL and full 40-character commit.
3. Add one entry to `custom-nodes.yaml`.
4. Add `pip` or `patches` only when the node actually requires them.
5. Run the focused unit test, Python compile check, `git diff --check`, and
   `docker compose config`.
6. Commit and push if requested.
7. Tell the operator to pull, build, recreate, and inspect current-run logs.
