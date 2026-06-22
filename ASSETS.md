# Assets

Large assets are not committed to GitHub. They should be downloaded separately and placed under the repository paths below.

## Baidu Netdisk

TODO: add Baidu Netdisk share link after upload.

Suggested package to upload:

```text
cr_eval_release_v7.tar.gz
```

SHA256:

```text
9aca70099d4d66d4a253315151585280b6a9b05f5c580f995521bf7d13b9d0fb  cr_eval_release_v7.tar.gz
```

## Contents

| Asset | Size | Count / file | Expected path after extraction |
|---|---:|---:|---|
| HOI-Edit L1/L2 original images | 476 MB | 499 images | `data_v7/CR/data_v7_L12/` |
| HOI-Edit L3 original images | 126 MB | 143 images | `data_v7/CR/data_v7_L3/` |
| GroundingDINO weight | 694 MB | `groundingdino_swint_ogc.pth` | `third_party/GroundingDINO/weights/` |
| SAM2 checkpoint | 898 MB | `sam2.1_hiera_large.pt` | `sam2/checkpoints/` |

Checkpoint SHA256:

```text
3b3ca2563c77c69f651d7bd133e97139c186df06231157a64c507099c52bc799  groundingdino_swint_ogc.pth
2647878d5dfa5098f2f8649825738a9345572bae2d4350a2468587ece47dd318  sam2.1_hiera_large.pt
```

## Notes

- The GitHub repository keeps code and annotation JSON files only.
- `data_v7/CR/data_v7_L12/`, `data_v7/CR/data_v7_L3/`, `third_party/GroundingDINO/weights/`, and `sam2/checkpoints/` are ignored by git.
- Uploading to Baidu Netdisk requires a logged-in Baidu Netdisk client or CLI authorization. This environment currently has no `bypy` or Baidu Netdisk CLI configured.
