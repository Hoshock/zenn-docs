---
title: "GitHub Organizations に含まれるリポジトリの一覧を取得する"
emoji: "🔭"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["github", "graphql"]
published: true
---

## Motivation

GitHub の Organizations 内にあるリポジトリが増えてきたので、その一覧を markdown の表として出力したい。また、興味のあるリポジトリは特定のプレフィックス群から始まるので、それらでフィルタをかける必要がある。

## Script

- `OUTPUT_FILE`: 出力ファイル名
- `PREFIXES`: プレフィックス群
- `ORGANIZATION`: Organizations 名

```shell
#!/bin/bash

set -euCo pipefail

# constants
readonly OUTPUT_FILE="repo_table.txt"
readonly PREFIXES=("PREFIX_0" "PREFIX_1" "PREFIX_2")
readonly ORGANIZATION="YOUR_ORGANIZATION"

function list_repo() {
  local -r query='query($endCursor: String) {
    organization(login: "'"${ORGANIZATION}"'") {
      repositories(first: 100, after: $endCursor) {
        pageInfo {
          endCursor
          hasNextPage
        }
        nodes {
          name
          isArchived
        }
      }
    }
  }'

  local -r result=$(gh api graphql -F query="$query" --paginate)
  for prefix in "${PREFIXES[@]}"; do
    echo "${result}" |
      jq -r '.data.organization.repositories.nodes[] | select(.name | startswith("'"${prefix}"'")) | .name + " " + (.isArchived | tostring)' |
      sort
  done
}

function main() {
  echo "| Name | URL | Description | Archived |"
  echo "| ---- | --- | ----------- | -------- |"

  while read -r line; do
    read -r name archived <<<"${line}"
    if [[ "${archived}" == "true" ]]; then
      archived="o"
    else
      archived=""
    fi
    echo "| ${name} | https://github.com/${ORGANIZATION}/${name} | | ${archived} |"
  done < <(list_repo)
}

if [[ "${BASH_SOURCE[0]}" == "${0}" ]]; then
  main >"${OUTPUT_FILE}"
fi
```

## Future Prospect

- Description 列は手動で書くことを想定しているが、リポジトリの About から自動で取得したい
  - ただし、そのためにはリポジトリ作成のルール決めをする必要がある
- 一覧の更新には手動で上記スクリプトを叩く必要があるが、自動実行にしたい
