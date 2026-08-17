# OCI Annotations Implementation Guide

## Overview

The Docker Bake reusable workflow now automatically manages OCI annotations for all container images. The 4 standard OCI annotations are computed from GitHub Actions context and automatically applied at all manifest levels, while custom annotations from the calling repository are preserved.

## Key Features

✅ **Automatic** - No additional workflow inputs or configuration needed  
✅ **Standard** - Uses official OCI Container Image Spec annotations  
✅ **Multi-level** - Annotations at all 3 levels for multi-arch images (index, descriptors, image manifests)  
✅ **Override** - Managed annotations always take precedence over calling repo values  
✅ **Compatible** - Merges seamlessly with custom annotations from bake files  

## Managed Annotations

The workflow **automatically computes and applies** these 4 OCI annotations to every build:

```
org.opencontainers.image.created   → Build timestamp (RFC 3339 UTC)
org.opencontainers.image.source    → Repository URL
org.opencontainers.image.version   → Git ref name (branch/tag)
org.opencontainers.image.revision  → Git commit SHA
```

These annotations:
- Are computed from GitHub Actions context (`github.server_url`, `github.repository`, `github.sha`, `github.ref_name`)
- Are set at **build time** and **merge time**
- **Always override** any values in the calling repository's bake file
- Ensure consistency across all images built by the workflow

## For Calling Repository Users

### No Changes Needed!

The workflow is fully backward compatible. Existing users don't need to change anything - the annotations are added automatically.

### To Add Custom Annotations

Simply add an `annotations` field to your bake targets:

```hcl
target "my-image" {
  context = "."
  dockerfile = "Dockerfile"
  tags = ["myregistry/myimage:latest"]
  platforms = ["linux/amd64", "linux/arm64"]
  
  # Optional: add custom annotations
  annotations = [
    "custom.org.app=myapp",
    "custom.org.environment=production",
    "custom.org.team=platform"
  ]
}
```

### Annotation Merging Behavior

When you define annotations in your bake file:

1. **Custom annotations** (any except the 4 managed ones) are preserved
2. **Managed annotations** from workflow override bake file values:
   ```hcl
   annotations = [
     # These will be OVERRIDDEN by workflow
     "org.opencontainers.image.created=2000-01-01T00:00:00Z",   # ❌ Ignored
     "org.opencontainers.image.source=https://old.url",         # ❌ Ignored
     "org.opencontainers.image.version=v0.0.0",                 # ❌ Ignored
     "org.opencontainers.image.revision=0000000000000000000000000000000000000000",  # ❌ Ignored
     
     # These will be KEPT
     "custom.org.app=myapp",                                    # ✅ Preserved
     "custom.org.description=My custom annotation"              # ✅ Preserved
   ]
   ```

3. **Final annotations** include all 4 managed + all custom:
   ```json
   {
     "org.opencontainers.image.created": "2026-08-17T14:32:45Z",
     "org.opencontainers.image.source": "https://github.com/user/repo",
     "org.opencontainers.image.version": "v1.0.0",
     "org.opencontainers.image.revision": "abc123def456abc123def456abc123def456abc1",
     "custom.org.app": "myapp",
     "custom.org.description": "My custom annotation"
   }
   ```

## Implementation Architecture

### Prepare Job

```
┌─────────────────────────────────────────┐
│ Compute OCI Annotations                 │
│ ┌─────────────────────────────────────┐ │
│ │ org.opencontainers.image.created    │ │  ← date -u +'%Y-%m-%dT%H:%M:%SZ'
│ │ org.opencontainers.image.source     │ │  ← github.server_url/github.repository
│ │ org.opencontainers.image.version    │ │  ← github.ref_name
│ │ org.opencontainers.image.revision   │ │  ← github.sha
│ └─────────────────────────────────────┘ │
│                                          │
│ Output: annotations_json (all 4)        │
└─────────────────────────────────────────┘
```

### Build Job

```
annotations_json (from prepare)
    ↓
Convert to Buildx format: *.annotations=key=value
    ↓
Pass to docker/bake-action via 'set' parameter
    ↓
Buildx merges with bake file annotations
    ↓
Result: Image manifests with annotations
```

### Merge Job (Multi-Arch Only)

```
annotations_json (from prepare)
    ↓
Convert to imagetools format: --annotation key=value
    ↓
Pass to docker buildx imagetools create
    ↓
Result: OCI Index + Descriptors + Image Manifests (all with annotations)
```

## Annotation Levels

### Multi-Architecture Images

Multi-arch images created by `docker buildx imagetools` have annotations at **3 levels**:

1. **OCI Index** (Top Level)
   - Referenced by the image tag
   - Contains `mediaType`, `manifests` array, `annotations`
   - Type: `application/vnd.oci.image.index.v1+json`

2. **Descriptors** (Per Platform)
   - Objects in the `manifests` array
   - One descriptor per architecture
   - Contains `mediaType`, `digest`, `platform`, `annotations`

3. **Image Manifests** (Architecture-Specific)
   - Individual manifest for each platform
   - Referenced via descriptor's digest
   - Contains `config`, `layers`, `annotations`
   - Type: `application/vnd.oci.image.manifest.v1+json`

### Single-Architecture Images

Single-arch images have annotations at **1 level** only:

1. **Image Manifest**
   - The only manifest (no index or descriptors)
   - Contains `config`, `layers`, `annotations`

## Annotation Flow Diagram

```
Calling Repo Bake File
(custom annotations)
       ↓
       ├─→ docker buildx bake
       │   (build for each platform)
       │
       └─→ Image Manifests + custom annotations
            ↓ (per platform)
            │
            └─→ Workflow-managed annotations override
                 ↓
                 Multi-platform images?
                 ├─→ YES: docker buildx imagetools create
                 │        (merge with annotations)
                 │        ↓
                 │        OCI Index
                 │        + Descriptors
                 │        + Image Manifests (all with annotations)
                 │
                 └─→ NO: Single-arch
                        (annotations from build phase only)
                        ↓
                        Image Manifest (with annotations)
```

## Code Changes

### Reusable Workflow (gha-workflows)

**File**: `.github/workflows/docker-bake.yaml`

**Changes**:
1. Added `annotations_json` output to `prepare` job
2. Added "Compute OCI annotations" step in `prepare`
3. Added "Prepare OCI annotations for build" step in `build`
4. Updated `docker/bake-action` call to include `${{ env.ANNOTATION_ARGS }}`
5. Added "Prepare OCI annotations for merge" step in `merge`
6. Updated `docker buildx imagetools create` call to include `$ANNOTATION_FLAGS`

**No new inputs** - workflow is fully backward compatible

### Test Repository (gha-test)

**File Changes**:
- `docker-bake.hcl` - Added annotations to test targets
- `docker-bake-multi-target-single-arch.hcl` - Added annotations to test scenarios
- `docker-bake-single-target-single-arch.hcl` - Added managed override test
- `testdir/testsubdir/docker-bake-multi-target-mixed-arch.hcl` - Added mixed scenarios
- `ANNOTATIONS_VERIFICATION.md` - Comprehensive testing guide

**No workflow changes needed** - existing test workflow works as-is

## Workflow Execution Timeline

### Prepare Job
```
├─ Compute OCI annotations
│  ├─ created    = 2026-08-17T14:32:45Z
│  ├─ source     = https://github.com/k-candidate/gha-test
│  ├─ version    = annotations (git ref)
│  └─ revision   = abc123def456... (git SHA)
│
└─ Output: annotations_json
```

### Build Job (Per Platform)
```
├─ Prepare OCI annotations
│  └─ Convert: *.annotations=org.opencontainers.image.created=2026-08-17T14:32:45Z
│              *.annotations=org.opencontainers.image.source=...
│              *.annotations=org.opencontainers.image.version=...
│              *.annotations=org.opencontainers.image.revision=...
│
└─ Build (docker buildx bake)
   └─ Result: Image manifest with all annotations
```

### Merge Job (If Multi-Arch)
```
├─ Prepare OCI annotations
│  └─ Convert: --annotation org.opencontainers.image.created=2026-08-17T14:32:45Z
│              --annotation org.opencontainers.image.source=...
│              --annotation org.opencontainers.image.version=...
│              --annotation org.opencontainers.image.revision=...
│
└─ Create manifest lists (docker buildx imagetools create)
   └─ Result: OCI Index with annotations
              ├─ Descriptor 1 (amd64) with annotations
              └─ Descriptor 2 (arm64) with annotations
```

## Benefits

1. **Zero Configuration** - Works automatically without any additional inputs
2. **Consistency** - All images get the same 4 standard annotations
3. **Traceability** - Source URL, version, revision always correct and consistent
4. **Override Protection** - Managed annotations can't be accidentally incorrect
5. **Flexibility** - Custom annotations still fully supported
6. **Standards Compliant** - Uses official OCI Image Spec annotations
7. **Multi-Level** - Annotations at all manifest levels for discoverability

## Version Compatibility

- **Backward Compatible**: Existing workflows work without changes
- **No New Inputs**: Annotation handling is transparent to users
- **Docker/Buildx**: Requires recent version with annotation support
  - `docker buildx` with `--annotation` flag support
  - `docker buildx imagetools` with `--annotation` flag support

## Troubleshooting

### Issue: Annotations Not in Built Image

**Check**:
1. Image was pushed to registry (not just local)
2. Using `docker buildx imagetools inspect image:tag --raw` (requires `--raw` flag)
3. Docker/Buildx version supports annotations

**Debug**:
```bash
# Check workflow logs for "Compute OCI annotations" step output
# Verify annotations_json contains expected 4 annotations
```

### Issue: Managed Annotations Have Wrong Values

**Check**:
1. GitHub Actions context is correct (SHA, ref name)
2. Workflow has proper `needs:` dependencies
3. `annotations_json` output from prepare job is passed to build/merge jobs

**Debug**:
```bash
# In workflow logs, check Compute OCI annotations output
# Verify org.opencontainers.image.revision matches git SHA
# Verify org.opencontainers.image.version matches branch/tag name
```

### Issue: Custom Annotations Missing

**Check**:
1. Annotations syntax in bake file is correct: `annotations = ["key=value", ...]`
2. Annotation values don't contain special characters that need escaping
3. Target is included in the build group/targets

**Debug**:
```bash
# Verify bake file parses correctly
docker buildx bake --file docker-bake.hcl --print | jq '.target.my-target.annotations'
```

## References

- [OCI Image Spec - Annotations](https://github.com/opencontainers/image-spec/blob/main/annotations.md)
- [Docker Buildx Annotations](https://docs.docker.com/build/metadata/annotations/)
- [OCI Image Index](https://github.com/opencontainers/image-spec/blob/main/image-index.md)
- [GitHub Actions Context](https://docs.github.com/en/actions/learn-github-actions/contexts)
