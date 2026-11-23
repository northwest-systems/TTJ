# TTJ Runbook

## リリース手順

Git・GitHub CLI（`gh`）・Python・uvを利用する。Gitは変更とタグの管理、`gh` はPRとGitHub Releaseの操作、Pythonとuvは文書・Skillの検証に使う。以下のコマンドはPowerShellでリポジトリのルートから実行する。

### 1. PRで変更を反映する

mainへの変更はPR経由で取り込む。作業ブランチで規格・JSON参照形式・Skillの整合性をレビューし、README、規格本文、JSON参照形式、Skillの版と `protocol` をリリース対象版へ揃える。正式版ではREADMEのDraft表記を外し、規格本文の文書状態を `stable` にする。

Job計画の正式作成前を表すDraftや、JSON例の識別値に含まれるdraftは変更しない。

### 2. リリース候補を検証する

```powershell
git diff --check
```

JSON参照形式のすべてのJSONコードブロックをパースできることを確認する。

```powershell
$ttjJsonText = Get-Content skills/ttj/references/json-format.md -Raw
$ttjJsonBlocks = [regex]::Matches($ttjJsonText, '(?s)```json\r?\n(.*?)\r?\n```')
if ($ttjJsonBlocks.Count -eq 0) { throw 'JSON例がありません' }
foreach ($ttjJsonBlock in $ttjJsonBlocks) {
    $null = ConvertFrom-Json -InputObject $ttjJsonBlock.Groups[1].Value -ErrorAction Stop
}
```

文書リンクの参照先、JSON例の `protocol`、`owner_id`・実行権・`depends`/`refs` 区分・状態遷移・終了判定が規格本文と一致することも確認する。

Skillの形式は、skill-creatorの `scripts/quick_validate.py` で検証する。`$ttjValidator` には利用環境にあるスクリプトの実際のパスを指定する。uvは検証スクリプトが必要とするPyYAMLを一時環境に用意する。

```powershell
$ttjValidator = '<skill-creatorの配置先>/scripts/quick_validate.py'
uv run --no-project --with pyyaml python -X utf8 $ttjValidator skills/ttj
```

検証結果と変更内容をPRに記載し、レビュー後にマージする。非公開リポジトリでPR必須をGitHubの設定でも強制するには、ブランチ保護を利用できるプランが必要。

### 3. タグと正式リリースを作成する

```powershell
git switch main
git pull --ff-only origin main
git status --short
git log -1 --oneline
```

作業ツリーに未反映の変更がなく、HEADがレビュー済みPRのマージ結果であり、リモートmainと一致することを確認する。公開対象のコミットでも手順2を確認する。

対象版とリリース説明を設定し、同名のタグ・リリースが存在しないことを確認する。既存タグは付け替えない。

```powershell
$ttjVersion = '1.1.0'
$ttjTag = "v$ttjVersion"
$ttjNotes = '<変更内容と互換性への影響>'
$ttjCommit = git rev-parse HEAD
git tag -a $ttjTag $ttjCommit -m "TTJ $ttjVersion"
git push origin $ttjTag
gh release create $ttjTag --repo northwest-systems/TTJ --verify-tag --title "TTJ $ttjVersion" --notes $ttjNotes --latest
```

各コマンドの成功を確認してから次へ進む。正式リリースでは `--draft`・`--prerelease` を指定しない。

### 4. 公開結果を確認する

```powershell
gh release view $ttjTag --repo northwest-systems/TTJ --json tagName,isDraft,isPrerelease,url
gh api "repos/northwest-systems/TTJ/git/ref/tags/$ttjTag"
```

リリースの版が対象版と一致し、`isDraft`・`isPrerelease` がともに `false` であることを確認する。注釈付きタグは、返されたタグオブジェクトの参照先も取得し、公開対象のコミットSHAと一致することを確認する。
