# Release Strategy

GitHub Flow + Release Branch dengan SemVer per change.

Approval via **GitHub Issues** — tidak perlu setup GitHub Environments.

## Struktur Workflow

```
.github/
├── approvers.yml                         # Daftar approver per environment
└── workflows/
    ├── create-release-candidate.yml      # Manual — buat release branch
    ├── create-release.yml                # Manual — buat tag + GitHub Release + trigger deploy
    ├── release.yml                       # Auto — build image, mulai deployment chain
    ├── on-issue-comment.yml              # Auto — approval gate via /approve / /reject
    ├── deploy-development.yml            # Auto — deploy ke development (no approval)
    ├── deploy-staging.yml                # Auto — deploy ke staging (butuh approval)
    ├── deploy-production.yml             # Auto — deploy ke production (butuh approval)
    └── cleanup.yml                       # Auto — hapus release branch lama
```

## Deployment Chain

| Source | Dev | Staging | Production |
|---|---|---|---|
| `main` + regular (preview) | ✅ Auto | ✅ Perlu approval | ❌ Skip |
| `release/vX.X.x` + regular | ✅ Auto | ✅ Perlu approval | ✅ Perlu approval |
| Hotfix | ❌ Skip | ✅ Perlu approval | ✅ Perlu approval |

## Approval

Approval dilakukan via komentar di GitHub Issue yang otomatis dibuat workflow:

| Komentar | Aksi |
|---|---|
| `/approve` | Deploy ke environment yang dituju |
| `/reject [alasan]` | Batalkan deployment |

Siapa yang boleh approve dikonfigurasi di `.github/approvers.yml`.

## Flow Normal (Sprint)

```
1. Developer merge PR ke main

2. Code freeze → buat release branch
   Actions → Create Release Candidate
   bump_type : minor
   → branch: release/v1.5.x

3. QA final di release branch

4. Siap rilis → Actions → Create Release
   release_branch : release/v1.5.x
   bump_type      : minor
   → tag v1.5.0 dibuat
   → dev: auto-deploy
   → issue pending-staging dibuka → engineer /approve → deploy staging
   → issue pending-production dibuka → tech lead /approve → deploy production
```

## Flow Hotfix

```
1. Fix di main dulu (PR seperti biasa)

2. Cherry-pick ke release branch aktif
   git checkout release/v1.5.x
   git cherry-pick -x <commit-hash>    ← wajib pakai -x
   git push origin release/v1.5.x

3. Actions → Create Release
   release_type   : hotfix
   release_branch : release/v1.5.x
   → tag v1.5.3 dibuat
   → dev: SKIP (fix sudah ada di main)
   → issue pending-staging dibuka → engineer /approve → deploy staging
   → issue pending-production dibuka → tech lead /approve → deploy production
```

## SemVer

| Bump | Kapan | Contoh |
|---|---|---|
| `patch` | Bug fix / hotfix | v1.5.2 → v1.5.3 |
| `minor` | Fitur baru | v1.5.x → v1.6.0 |
| `major` | Breaking change | v1.x.x → v2.0.0 |

Tag bisa dibuat dari **release branch** maupun **main** (default jika `release_branch` dikosongkan).
Setelah sprint release, main lanjut ke minor berikutnya (v1.5.x → v1.6.x) agar v1.5.x reserved untuk hotfix di release branch.

## Branch Lifecycle

- Simpan **2 release branch terakhir** (untuk rollback & hotfix)
- `cleanup.yml` otomatis hapus yang lama setiap kali tag baru di-push
