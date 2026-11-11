# github-action-playground

Workflow manual untuk membuat PR sebagai `github-actions[bot]`, agar pengguna yang menjalankan workflow tetap dapat mereview dan approve PR sesuai ruleset/branch protection repo.

## Workflow utama: tanpa PAT

File: `.github/workflows/create-pr.yml` — **Create PR (GitHub Actions bot)**.

1. Untuk instalasi awal, merge workflow ke **default branch** (`main` di repo ini).
2. Pastikan **Settings → Actions → General → Workflow permissions → Allow GitHub Actions to create and approve pull requests** aktif. Policy organisasi dapat membatasi pengaturan ini. Workflow meminta `contents: read` dan `pull-requests: write`; tidak perlu memberi semua workflow akses tulis.
3. Buka **Actions → Create PR (GitHub Actions bot) → Run workflow**.
4. Pilih branch di **Use workflow from**, isi input jika perlu, lalu jalankan. Link PR muncul di summary run.

| Input | Default / perilaku |
| --- | --- |
| `base_repo` | Repo tempat workflow dijalankan. Pada versi bot, harus tetap repo ini. |
| `base_branch` | `main`, bisa diubah ke branch lain di repo tujuan. |
| `head_repo` | Repo tempat workflow dijalankan; bisa diisi `owner/fork` dari fork network yang sama. |
| `head_branch` | Branch yang dipilih di **Use workflow from**. Isi eksplisit untuk branch dari fork atau branch lain. |
| `title` | Subject commit pertama dalam hasil compare base → head: commit yang ada di head tetapi belum ada di base, dalam urutan kronologis API GitHub. Bukan commit pertama sepanjang sejarah repo. |
| `body` | Kosong. |
| `draft` | `false`. |

Contoh satu repo: pilih `feature/example` di **Use workflow from**, biarkan input lain kosong/default. Hasilnya PR `feature/example` → `main` di repo ini. Branch tersebut harus sudah memiliki file workflow.

Contoh source dari fork: jalankan di repo tujuan, isi `head_repo=someone/github-action-playground` dan `head_branch=feature/example`. Biarkan `base_repo` kosong dan `base_branch=main`.

PR terbuka yang sudah ada untuk source repo/branch dan target branch yang sama akan dipakai kembali; workflow tidak mengubah judul/body PR lama. Source tanpa commit tambahan akan ditolak. GitHub tetap menentukan apakah perubahan dapat dijadikan PR.

## Fork dan batasan token

`GITHUB_TOKEN` hanya berlaku untuk repository tempat workflow dijalankan. Memberi izin membuat/approve PR tidak memberinya akses tulis ke semua fork atau repo pengguna.

- Source dari fork publik dapat digunakan selama satu fork network. Source privat harus dapat dibaca oleh token; token bawaan tidak otomatis mendapat akses lintas repo privat.
- Untuk menjadikan fork lain sebagai **base repo** tanpa PAT, pasang dan jalankan workflow di fork tujuan tersebut. Aktifkan Actions jika masih dinonaktifkan pada fork itu.
- Kedua repo harus satu fork network; repo yang kebetulan punya isi sama tetapi tidak berhubungan sebagai fork tidak didukung.
- Workflow tidak melakukan checkout atau menjalankan kode source fork. Input diteruskan sebagai environment variables, bukan disisipkan ke kode JavaScript/shell.

## Dropdown branch

GitHub Actions tidak mendukung `workflow_dispatch.inputs` bertipe dropdown dengan opsi yang diambil dinamis dari API. `type: choice` hanya menerima daftar statis di YAML.

**Use workflow from** sudah menyediakan pemilih branch/tag repo saat ini. Workflow memakai pilihan branch ini sebagai default `head_branch`. Base branch dan branch fork menggunakan input teks. Workflow tidak menulis ulang YAML hanya untuk menyegarkan opsi dropdown.

## Mencoba perubahan sebelum merge

`workflow_dispatch` harus terdaftar di default branch terlebih dahulu. Bootstrap pertama perlu merge ke `main`; workflow tidak melakukan merge otomatis.

Setelah bootstrap:

1. Buat/push branch pengujian yang memuat perubahan YAML dan commit tambahan.
2. Jalankan workflow dari **Use workflow from → branch pengujian**. Versi jobs/steps dari ref itu yang dijalankan, sehingga perubahan implementasi dapat dites sebelum merge.
3. Biarkan `head_branch` kosong untuk membuat PR branch pengujian ke `main`, atau isi source secara eksplisit.

Pemilih workflow bukan dropdown source fork. Untuk fork, tetap pilih ref workflow dari repo tempat workflow dijalankan, lalu isi `head_repo` dan `head_branch`.

Form input di UI dapat mengikuti definisi pada default branch. Untuk perubahan skema/nama input, perbarui default branch agar form ikut sesuai; gunakan input yang sudah terdaftar saat mengetes implementasi di branch lain.

## Opsional: workflow PAT terpisah

File: `.github/workflows/create-pr-pat.yml` — **Create PR (PAT)**. Tidak ada fallback PAT tersembunyi pada workflow bot.

Tambahkan repository secret `PR_CREATOR_PAT`, lalu jalankan workflow ini. `base_repo` dapat diisi repo lain yang diizinkan token. Fine-grained PAT memerlukan **Pull requests: Read and write** di target serta akses membaca repo/branch sumber dan target. Pemilik token juga harus punya akses repo yang diperlukan. Cakupan resource owner dan policy organisasi dapat membatasi pilihan lintas fork; classic PAT memerlukan cakupan yang sesuai dengan visibilitas repo.

**PR dari PAT dibuat atas nama pemilik PAT.** Jika memakai PAT akunmu sendiri, kamu tetap tidak bisa approve PR itu sendiri. Untuk mempertahankan tujuan approval sambil memakai kredensial lintas repo, gunakan akun bot terpisah atau GitHub App dengan token instalasi dan akses repo yang sesuai; GitHub App bukan bagian dari implementasi ini.

## Review PR dan CI

Approval review PR dan approval menjalankan workflow CI adalah dua tindakan berbeda. Bot membuat PR, lalu manusia melakukan review sesuai aturan repo; workflow ini tidak melakukan approval atau merge otomatis.

Dokumentasi GitHub saat ini menjelaskan bahwa event `pull_request` tipe `opened`, `synchronize`, atau `reopened` yang ditimbulkan `GITHUB_TOKEN` dapat menghasilkan run berstatus **approval-required**. Pengguna dengan akses write dapat memilih **Approve workflows to run**. Pada host/versi yang belum mendukung perilaku tersebut, event dari token bawaan umumnya tidak memicu workflow lanjutan; `workflow_dispatch` dan `repository_dispatch` merupakan pengecualian. Jangan mengasumsikan CI sudah berjalan hanya karena PR berhasil dibuat.

Referensi:

- [Menjalankan workflow manual](https://docs.github.com/en/actions/how-tos/manage-workflow-runs/manually-run-a-workflow)
- [Input workflow_dispatch](https://docs.github.com/en/actions/reference/workflows-and-actions/workflow-syntax#onworkflow_dispatchinputs)
- [GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token)
- [Perilaku event dari GITHUB_TOKEN, sumber dokumentasi GitHub](https://github.com/github/docs/blob/main/data/reusables/actions/actions-do-not-trigger-workflows.md)
- [API create pull request dan cross-repo head](https://docs.github.com/en/rest/pulls/pulls#create-a-pull-request)
