# PSE Seminar Example (beta)

- PSE研究室向けのリポジトリです。
- このリポジトリを参考にすることで、LaTeXでゼミ用のレポートを作成し、自動で配信することができます。

## Procedure

### Step 1

- `main` ブランチと `distribution` ブランチを作成します。 
- `.github/workflows` にこのリポジトリと同様の `compile.yml` と `deploy.yml` を配置します。

<details>
<summary>compile.yml</summary>

```yaml
name: Compile

on:
  push:
    branches:
      - 'seminar/**'
  workflow_dispatch:

jobs:
  main:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout branch
        uses: actions/checkout@v2
      - name: Count the number of pull requests
        id: count
        run: echo "::set-output name=count::$(gh pr list --search "$GITHUB_REF_NAME" --state all | wc -l)"
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Create pull request
        if: ${{ steps.count.outputs.count == 0 }}
        run: gh pr create --base main --title "$GITHUB_REF_NAME" --body ""
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
      - name: Compile report
        uses: tarao1006/pse-seminar-report@main
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
```
</details>

<details>
<summary>deploy.yml</summary>

```yaml
name: Deploy

on:
  pull_request:
    branches:
      - main
    types: [closed]

jobs:
  main:
    runs-on: ubuntu-latest
    environment: email
    if: github.event.pull_request.merged == true
    steps:
      - name: Checkout Branch
        uses: actions/checkout@v2
        with:
          ref: distribution
      - name: Parse branch
        id: parse
        run: |
          IFS=/ ARR=(${{ github.head_ref }})
          echo "::set-output name=date::${ARR[1]}"
        shell: bash
      - name: Send email
        uses: tarao1006/pse-seminar-email@main
        with:
          ecs_id: ${{ secrets.ECS_ID }}
          password: ${{ secrets.PASSWORD }}
          to: ${{ secrets.TO }}
          from: ${{ secrets.FROM }}
          family_name: ${{ secrets.FAMILY_NAME }}
          given_name: ${{ secrets.GIVEN_NAME }}
          pdf: distribution/${{ github.head_ref }}/report.pdf
          date: ${{ steps.parse.outputs.date }}

```
</details>

### Step 2

- `seminar/%Y%m%d` ブランチを作成し、ゼミ用のレポートを作成します。( `%Y%m%d` は [strftime() と strptime() の書式コード](https://docs.python.org/ja/3/library/datetime.html#strftime-and-strptime-format-codes) に記されている書式コードを表します。例. `20211202` )
- **レポートはブランチ名と同じ名前のディレクトリに作成します。**

```shell
$ git switch -c seminar/20211202
$ makdir -p seminar/20211202
$ touch seminar/20211202/report.tex
```

### Step 3

- `seminar/%Y%m%d` ブランチにプッシュされるたびにレポートをコンパイルし、配信の準備をします。
- 初回プッシュ時には `seminar/%Y%m%d` ブランチから `main` ブランチへのプルリクエストを作成します。

### Step 4

- 作成されたプルリクエストがマージされる際に指定されたメールアドレス宛にPDFファイルが送信されます。 🎉

## Secrets

SMTPサーバの設定や送信先メールアドレスの設定にGitHubのSecrets機能を使用します。
`Settings > Secrets > New repository secret` から以下の6つの値を設定します。

| key | value |
| :-: | :-: |
| ECS_ID | ECS ID |
| PASSWORD | ESC IDに紐づくパスワード |
| TO | 送信先メールアドレス |
| FROM | 送信元メールアドレス |
| FAMILY_NAME | 性 |
| GIVEN_NAME | 名 |

<img width="900" alt="secrets" src="https://user-images.githubusercontent.com/32538736/144426635-618f8e0c-b8aa-449f-b7a6-53cb284ce4fa.png">

▲　Secretsを設定した例

## Actions

使用しているアクションは以下の二つです。

- Email配信: https://github.com/tarao1006/pse-seminar-email
- コンパイル: https://github.com/tarao1006/pse-seminar-report

これらを参考に `inputs` の値を変更、またはアクションの編集を行ってください。

## Dockerfile

latexのコンパイルに使用しているDockerイメージは以下の通りです。

- tarao1006/texlive: https://hub.docker.com/repository/docker/tarao1006/texlive

## License

[MIT License](LICENSE)
