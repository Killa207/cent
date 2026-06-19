**Repository Summary**
- **What it is:** `cent` collects community Nuclei templates (YAML) by cloning repositories and organizing templates on disk.
- **Primary packages:** CLI entry in `cmd/` (uses `cobra`), core logic in `pkg/jobs/`, small helpers in `internal/utils/`.

**How to build & run (dev)**
- **Build:** `go build ./...` (module path `github.com/xm1k3/cent/v2`, Go 1.24)
- **Run locally:** `go run ./` or `go run ./main.go` — the CLI is implemented with `cobra` in `cmd/`.
- **Install (user):** `go install github.com/xm1k3/cent/v2@latest` (matches README)

**Quick runtime notes**
- The app requires `git` in `PATH` (checked at runtime in `pkg/jobs.Start`).
- The tool expects a YAML config at `~/.config/cent/.cent.yaml` by default (viper config name: `.cent`).
- External integration: users typically run `nuclei` separately against the produced `cent-nuclei-templates` folder.

**Key files and patterns**
- `cmd/root.go` — orchestrates flags and high-level flow. Flags: `--path` (`-p`, default `cent-nuclei-templates`), `--threads` (`-t`, default `10`), `--timeout` (`-T`, default `2` minutes), `--keep-folders` (`-k`), `--by-repo` (`-r`), `--console` (`-C`). The command action calls `jobs.Start`, `jobs.RemoveEmptyFolders`, `jobs.UpdateRepo`, `jobs.RemoveDuplicates`.
- `pkg/jobs/*.go` — core behavior:
  - `ParseRepoEntries()` reads `viper` key `community-templates` and supports either a plain string (URL) or a map with `url`, `commit`, and `exclude` entries.
  - `Start(path, console, threads, timeout, keepFolders, byRepo)` — clones repos into `os.TempDir()` (folder `cent{timestamp}/repo{index}`), filters by `exclude` entries, then copies `.yaml` files into the target path.
  - `cloneRepo()` uses `git clone` with `--depth 1` when no commit is specified and a context timeout controlled by the `--timeout` flag.
  - `RemoveDuplicates` computes SHA256 hashes and deletes duplicate templates (keeps the lexicographically first file).
- `internal/utils/utils.go` — small helpers: `GetDataDir()` (returns `~/.config/cent/` on unix), `CopyFile`, `DownloadFile` (used by `init`), `RemoveStringFromSlice`.

**Config shape (discoverable)**
- Config key: `community-templates` is an array. Entries can be:
  - String form: `- https://github.com/user/repo`
  - Map form: `- url: https://github.com/user/repo\n  commit: abc123\n  exclude: ["workflows/", "docs/"]`
- Other keys: `exclude-dirs` and `exclude-files` used by `jobs.UpdateRepo` to remove unwanted paths/files when updating.

**Developer workflows & debugging tips**
- To see `git` output and clone-time errors, run CLI with `-C` (`--console`) — `pkg/jobs.cloneRepo` forwards `Stdout`/`Stderr` when `console` is true.
- To reproduce how config is loaded, inspect `cmd/initConfig()` in `cmd/root.go` — `--config` can override the default data dir location returned by `internal/utils.GetDataDir()`.
- Temporary clones are created under the OS temp directory; cleanup is done by `DeleteFromTmp()` — if debugging clones, inspect `os.TempDir()` for `cent{timestamp}` folders before the program removes them.

**Where to change core behavior**
- Add/change repository parsing or per-repo filtering in `pkg/jobs.ParseRepoEntries`.
- Modify cloning behavior (shallow clone vs full, clone args, retry behavior) in `pkg/jobs.cloneRepo` and `worker`.
- Template filtering, deduplication, and organization lives in `pkg/jobs.Start` and `RemoveDuplicates`.

**Conventions & expectations**
- CLI implemented with `cobra` — add subcommands in `cmd/` and wire them in `cmd.Execute()`.
- Configuration uses `viper` — read from `~/.config/cent/.cent.yaml` unless `--config` is passed.
- Files of interest for audits: `pkg/jobs` (file operations), `internal/utils` (IO), and `cmd` (flags & config).

**Known external deps & system requirements**
- Requires `git` executable in PATH. `nuclei` is a common downstream consumer but not required to run the program.
- Go module deps are declared in `go.mod` (e.g., `cobra`, `viper`, `go-pretty`, `go-retryablehttp`).

**Small examples**
- Example `community-templates` entry that pins a commit:
  ```yaml
  - url: https://github.com/user/repo
    commit: abc123f
    exclude:
      - "workflows/"
  ```
- Sample command to run an update with deletion of excluded directories:
  ```bash
  cent update -p cent-nuclei-templates -d
  ```

**If you modify the code**
- Run `gofmt -w .` or `go vet ./...` before submitting changes.
- Validate behavior locally by running `go run ./` and exercise the flags (`-p`, `-t`, `-T`, `-k`, `-r`, `-C`) to ensure end-to-end cloning/copying.

Please review — tell me which areas need more detail (examples, config keys, or call-sites) and I will iterate.
