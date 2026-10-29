# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 專案定位

這是 Gelman、Vehtari、McElreath 等人 *Bayesian Workflow*（CRC Press, 2026）配套程式碼與網站的 fork。使用者把它當成**讀書實作空間**：跑書中案例、改模型、做習題。

- `origin` = `SCgeeker/Bayesian-Workflow`（自己的 fork，可推送）
- `upstream` = `avehtari/Bayesian-Workflow`（作者原始 repo；有 `final-book-version` tag，另有 `vouchers` 分支）
- 同步作者更新（主要是 `errata.qmd` 與案例修正）：`git fetch upstream && git merge upstream/main`

- 讀書筆記入口（Obsidian）：`D:\core\workout\cardsbox\Ideaverse_Growing\Atlas\Library\Books\01_input\Bayesian workflow.md`
- 查書原文：`/query-book` skill，書籍別名 `bw`（例：`query_book.py search bw "prior predictive" --max 5`）；另有 NotebookLM 筆記本（全書依章節切成 32 份 PDF），連結在上述筆記中。
- 線上版網站：https://avehtari.github.io/Bayesian-Workflow/

## 練習檔存放位置：`practice/`

作者的原始檔視為上游參考，**不直接修改**；所有自己的實作放在 `practice/`（上游沒有這個目錄，merge 不會衝突）：

```
practice/
├── ch04_multiple_choice/     # 依「章節_案例或主題」命名，對應書中章節
│   ├── mc_variant.R          # 自己的腳本；要改作者案例時先複製過來再改
│   ├── mc_variant_1.stan     # 沿用作者的遞增編號習慣
│   └── data/                 # 自己產生或額外下載的資料
├── exNN_<主題>/              # 書中習題（資料在 exercises/chapter_NN/）
└── nano4/<job>/              # 送 Nano4 的批次作業：slurm + upload/submit/fetch（見下方）
```

- 練習腳本同樣用 `root("practice", "ch04_multiple_choice", "mc_variant_1.stan")` 組路徑；引用作者資料直接 `root("multiple_choice", "data", ...)`，不要複製資料。
- `.gitignore` 已設定忽略 `practice/` 下**沒有副檔名**的檔案（cmdstanr 編譯出的模型執行檔），不必逐一列。
- 練習腳本不需要維持 knitr spin 格式，除非想用 Quarto 渲染成筆記。

## 環境與指令

- R：`E:\App\R\R-4.6.1\bin\x64\Rscript`；Quarto 用 RStudio 內附版本（`E:\APP\RStudio\resources\app\bin\quarto\bin\quarto`）。
- Stan 一律透過 **cmdstanr**（brms 也設 `options(brms.backend = "cmdstanr")`）；少數案例另用 rstanarm（`loo_comparison`、`roaches`、`sleep_study`、`variable_selection` 等）。需要 CmdStan 已安裝（`cmdstanr::cmdstan_path()` 可檢查）。
- 常用套件：`rprojroot`、`cmdstanr`、`brms`、`posterior`、`bayesplot`、`loo`、`ggplot2`、`dplyr`、`patchwork`、`ggdist`、`priorsense`、`tinytable`、`SBC`。

```sh
# 互動執行單一案例（工作目錄可在 repo 內任何位置）
Rscript bioassay/bioassay.R

# 渲染單一案例為 HTML（.R 檔是 knitr spin 格式）
quarto render bioassay/bioassay.R

# 渲染整個網站到 docs/（freeze: auto，已凍結的結果不會重算）
quarto render

# Draft profile（隱藏未公開內容，見 _quarto-draft.yml）
QUARTO_PROFILE=draft quarto render
```

一般案例在本機跑即可。取樣量大的（例如 `sbc/` 的大量重複擬合、`birthdays/` 的 GP 模型、大量 `loo` 重算）才送 Nano4，見下方「計算資源：Nano4」。

本 repo 沒有測試套件，也沒有 Makefile（`_quarto-draft.yml` 註解提到的 `make DRAFT=true` 在此 repo 不存在）。

## 架構

### 路徑解析：`rprojroot` + `.Bayesian-Workflow-root`

所有腳本開頭都用：

```r
library(rprojroot)
root <- has_file(".Bayesian-Workflow-root")$make_fix_file()
stan_file <- root("bioassay", "bioassay0.stan")
```

根目錄的空檔 `.Bayesian-Workflow-root` 是錨點，**不可刪除或改名**。新寫的腳本也要用 `root(...)` 組路徑，不要用 `setwd()` 或相對路徑。

### 案例研究（case studies）

每個頂層目錄（`bioassay/`、`golf/`、`world_cup/` …）是一章或一節的完整案例，結構一致：

- `<name>/<name>.R`：主腳本，同時是 Quarto 文件。用 knitr spin 語法：`#'` 開頭是 markdown 文字（含 YAML 標頭），`#+` / `#|` 是 chunk 選項，其餘為程式碼。編輯時要維持這個格式，否則網站渲染會壞。
- `<name>/*.stan`：模型通常以遞增版本命名（`dogs_0.stan` → `dogs_5.stan`、`park_1.stan` → `park_4.stan`），對應書中「逐步擴充模型」的工作流程；R 腳本依序編譯、比較。
- `<name>/data/`：該案例資料。
- 部分腳本內定義 `print_stan_file()` 輔助函式，在渲染時把 `.stan` 內容以 stan code block 輸出。

章節對照表在 `_quarto.yml` 的 `sidebar` 與 `casestudies.qmd`（例：Ch 23 = `world_cup`、Ch 25 = `golf`、Ch 31 = `sbc`）。

### 其他目錄

- `misc/chapter_XX/section_XX_YY/`：非完整案例的小範例與圖形重現程式，依章節編號放置（`other_code.qmd` 為入口）。
- `exercises/chapter_XX/`：習題所需資料，各有 README 說明；習題本身列在 `exercises.qmd`。
- `errata.qmd`：書籍勘誤，上游最常更新的檔案。
- `social-cards/`：網站社群預覽圖的產生腳本，與學習無關。

### 產物與忽略規則

cmdstanr 編譯出的模型執行檔會產生在 `.stan` 同目錄、同名無副檔名（如 `golf/golf_logistic`），`.gitignore` 逐一列出這些路徑；新增 Stan 模型時，若不想提交執行檔，記得把它加入 `.gitignore`。`stan_output/`、`*_cache`、`*_files`、`.quarto/` 也已忽略。網站輸出目錄 `docs/` 預設不存在。

## 計算資源：Nano4（國網中心晶創26）

連線與環境的權威文件在 `D:\core\personal\working_projects\NCHC_cloud_computing\`（`NANO4-guide.md` 日常用法；`twcc-ollama-proxy\nano4\README.md`、`ONBOARDING.md` 細節與設計取捨）。下面只摘要跑 Stan／R 需要的部分；有衝突時以該處為準。

### 連線（需要使用者本人輸 OTP）

- 只有 **WSL（Ubuntu）+ ControlMaster** 一條路：伺服器不提供金鑰認證，每條新連線都要密碼＋OTP；WSL 的 master 連線讓 OTP 降到「每次開機一次」。Claude 無法代輸 OTP，master 斷了要請使用者自己建立。
- 只跑 R、不要 Ollama 時用 `nano4_wsl.sh up`（**不要**用 `nano4-start.ps1`，它是給 LLM 推論用的、會提交 ollama job 計費）：
  ```powershell
  wsl -d Ubuntu -- bash /mnt/d/core/personal/working_projects/NCHC_cloud_computing/twcc-ollama-proxy/nano4/nano4_wsl.sh up
  ```
  之後 `wsl -d Ubuntu -- ssh nano4 <指令>` 皆免 OTP（查餘額：`ssh nano4 wallet`）。
- 會斷線需重輸 OTP 的情況：重開機、`wsl --shutdown`、睡眠、換網路／VPN。僅限台灣境內 IP。

### 已知的坑

- **不能用 `scp`**（`:22` 沒有 sftp-server）。傳檔用 `ssh nano4 "cat > 遠端路徑" < 本機檔案`，傳完逐檔比對 `wc -c`；範本是 `nano4/upload_via_master.sh`。大檔走 `:2222`（`nano4-sftp`，另需 OTP）。
- **從 PowerShell 下 `wsl -- bash -c '...$VAR...'` 會被中間層提前展開**（`$USER` 會變成 WSL 使用者名稱，安靜地讀寫錯路徑）。含變數的邏輯一律寫成 `.sh` 檔再 `wsl -d Ubuntu -- bash <路徑>`。
- 儲存：程式、R 套件、結果放 `/home/$USER`；`/work` 會自動清除 28 天未使用的檔案且無備份。
- 外網是白名單：CRAN（`cloud.r-project.org`）、`stan-dev.r-universe.dev`、PyPI、`github.com` 可達，但 **GitHub release 下載不可達**，所以 `cmdstanr::install_cmdstan()` 會失敗——CmdStan 已改用 conda-forge 裝在 `~/cmdstan-env/bin/cmdstan`。

### R + CmdStan 環境（已實測可用，含 brms threading）

```bash
module load miniconda3/26.1.1 biology/R/4.6.1 gcc/13.2
export R_LIBS_USER=$HOME/R/4.6
export CMDSTAN=$HOME/cmdstan-env/bin/cmdstan
export MAKEFLAGS=-j8
# 不需 LD_LIBRARY_PATH/LDFLAGS：libstdc++ 的 CXXABI 問題已由 $CMDSTAN/make/local 的 rpath 修好（cmdstan_fix.sh）
```

本 repo 的腳本不會自己設 CmdStan 路徑，在 Nano4 上要先 `cmdstanr::set_cmdstan_path(Sys.getenv("CMDSTAN"))`。已裝套件：`cmdstanr brms glmmTMB posterior loo bayesplot emmeans future.apply pwr`；本書另需的（`rprojroot`、`ggdist`、`patchwork`、`priorsense`、`tinytable`、`rstanarm` 等）尚未裝，要先在登入節點裝進 `~/R/4.6`。

### Slurm 批次

- **五個佇列全是 GPU 佇列**，沒有 CPU-only 分區：Stan 只用 CPU，但仍必須 `--gres=gpu:1`，每張 GPU 最多 12 cores、200 GB。按 GPU 小時計費（約 60 元／GPU 小時），`-c` 設 chains × threads 即可，不要為了 Stan 多要 GPU。
- `dev` 佇列每個 job 上限 4 小時。
- **一律 `sbatch -A "$NANO4_ACCOUNT"`**：帳號下有兩個計畫，不帶 `-A` 可能扣錯；`NANO4_ACCOUNT` 來自 `/work/$USER/scripts/nano4_env.sh`（gitignore，不要把計畫編號寫進本 repo）。
- Nano4 沒有 `/usr/bin/time`，用 bash 內建 `time`；峰值記憶體事後查 `sacct`。
- 登入節點只做安裝、編譯、單元測試等輕量工作，正式取樣一律送 job。

既有作業範本是 `twcc-ollama-proxy\nano4\r_bfda1\`，本 repo 的 `practice/nano4/<job>/` 照同一組四件式：

| 檔案 | 在哪跑 | 做什麼 |
|---|---|---|
| `upload_<job>.sh` | WSL（共用 master） | 逐檔 `cat >` 上傳到 `/home/<帳號>/<job>/` 並比對大小；記得連同 `.Bayesian-Workflow-root` 一起傳，`rprojroot` 才找得到根目錄 |
| `<job>.slurm` | Nano4 | `#SBATCH -p dev --gres=gpu:1 -c N -t HH:MM:SS -o slurm_logs/%x_%j.out`，載入上面的環境後 `Rscript` |
| `submit_<job>.sh` | Nano4 登入節點 | 先跑 testthat，全過才 `sbatch -A "$NANO4_ACCOUNT"`；缺帳號或仍是佔位符就拒絕提交 |
| `fetch_<job>.sh` | WSL | 以 `ssh nano4 "cat ..."` 取回結果、比對大小、不覆寫既有檔 |
