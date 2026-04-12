## はじめに：企業分析の死角

製薬・バイオベンチャーを分析するとき、財務諸表（EDINET）やパイプライン情報（[ClinicalTrials.gov](http://ClinicalTrials.gov)）はよく使われる。しかし、こんな問いには答えにくい。

- この企業はAMEDからどの研究課題で資金を受けているか？
- 非上場バイオベンチャーの従業員数は？
- 楽天メディカルのような未上場企業でも何か公的な活動情報は取れないか？

これらはすべて **gBizINFO** で取得できる。しかも無料で、APIキーも数分で発行される。

この記事では、gBizINFO REST APIをPythonで叩き、取得したデータをBigQueryに格納して分析する手順を解説する。

---

## gBizINFOとは何か

gBizINFO（Gビズインフォ）は経済産業省が運営する法人情報プラットフォームだ。登記されている約400万社を対象に、政府が保有する法人活動情報をオープンデータとして提供している。

取得できるデータを分類するとこうなる。

| カテゴリ | 具体的な項目 | ライフサイエンスでの使い道 |
| --- | --- | --- |
| 法人基本情報 | 法人名・代表者名・資本金・従業員数・設立年 | organizationテーブルの補完 |
| 補助金情報 | AMED・経産省・厚労省の補助金・採択課題名・金額 | 研究資金の流れの可視化 |
| 調達情報 | 官公庁との契約実績・落札金額 | 政府との関係性把握 |
| 届出・認定 | 再生医療等提供計画の認定・薬事関連届出 | パイプライン前段階の活動把握 |
| 表彰情報 | 経産省・厚労省からの表彰歴 | 企業評価の間接指標 |

特に **補助金情報** が強力だ。楽天メディカルのような非上場企業も、AMED補助金の採択情報は公開されているため、財務情報がゼロの企業でも資金の流れが見える。

---

## APIトークンの取得（3分で完了）

1. 以下のURLにアクセスする
    
    `https://info.gbiz.go.jp/hojin/APIManual`
    
2. 「利用申請ページへ」をクリック
3. 利用者区分を選択
    - **個人利用者**：メールアドレス・利用目的のみでOK
    - **法人担当者**：法人番号・法人名等も入力
4. 申請後、登録メールアドレスにAPIトークンが届く（即日〜数時間）

取得したトークンをWorkbenchの環境変数に設定しておく。

```bash
export GBIZINFO_TOKEN="取得したトークン"
```

---

## STEP 1：法人番号でgBizINFO APIを叩く

エンドポイントはシンプルだ。法人番号（13桁）をパスに含めてGETするだけ。

```python
import requests
import json
import os

TOKEN = os.environ.get("GBIZINFO_TOKEN")
BASE_URL = "https://info.gbiz.go.jp/hojin/v1/hojin"

def fetch_hojin_basic(corporate_number: str) -> dict | None:
    """
    法人番号から法人基本情報を取得する。
    """
    url = f"{BASE_URL}/{corporate_number}"
    headers = {
        "Accept": "application/json",
        "X-hojinInfo-api-token": TOKEN
    }
    try:
        resp = requests.get(url, headers=headers, timeout=10)
        resp.raise_for_status()
        data = resp.json()
        infos = data.get("hojin-infos", [])
        return infos[0] if infos else None
    except requests.RequestException as e:
        print(f"[ERROR] {corporate_number}: {e}")
        return None

# 武田薬品工業で試す（法人番号: 4120001182080）
info = fetch_hojin_basic("4120001182080")
print(json.dumps(info, ensure_ascii=False, indent=2))
```

レスポンスの主要フィールド：

```json
{
  "name": "武田薬品工業株式会社",
  "corporate_number": "4120001182080",
  "representative_name": "クリストフ・ウェバー",
  "capital_stock": 955.50,
  "employee_number": 6738,
  "founded": "1781",
  "business_summary": "医薬品の製造・販売",
  "company_url": "https://www.takeda.com/ja-jp/"
}
```

---

## STEP 2：AMED補助金情報を取得・整形する

補助金情報は別エンドポイントで取得する。

```python
import time
import pandas as pd

def fetch_subsidy(corporate_number: str) -> list[dict]:
    """
    法人番号から補助金・助成金情報を取得する。
    AMEDや経産省からの採択情報が含まれる。
    """
    url = f"{BASE_URL}/{corporate_number}/subsidy"
    headers = {
        "Accept": "application/json",
        "X-hojinInfo-api-token": TOKEN
    }
    try:
        resp = requests.get(url, headers=headers, timeout=10)
        resp.raise_for_status()
        data = resp.json()
        return data.get("hojin-infos", [{}])[0].get("subsidy", [])
    except requests.RequestException:
        return []

def parse_subsidies(org_id: str, corporate_number: str, subsidies: list[dict]) -> list[dict]:
    """補助金データをgrantテーブル用に整形する。"""
    rows = []
    for s in subsidies:
        ministry_raw = s.get("joint_organizations", "")
        if "AMED" in ministry_raw or "日本医療研究開発" in ministry_raw:
            ministry = "amed"
        elif "経済産業" in ministry_raw:
            ministry = "meti"
        elif "厚生労働" in ministry_raw:
            ministry = "mhlw"
        elif "文部科学" in ministry_raw:
            ministry = "mext"
        else:
            ministry = "other"

        rows.append({
            "org_id": org_id,
            "corporate_number": corporate_number,
            "grant_program": s.get("subsidy_resource", ""),
            "project_title": s.get("title", ""),
            "ministry": ministry,
            "amount_jpy": s.get("amount", None),
            "fiscal_year": s.get("target_period_from", "")[:4] or None,
        })
    return rows

# 複数企業を一括取得
TARGETS = [
    ("ORG_TAK",  "4120001182080"),  # 武田薬品
    ("ORG_ANJS", "6010401044542"),  # AnGes
    ("ORG_HLIO", "4010401108271"),  # Healios
    ("ORG_SNBI", "7010401122467"),  # SanBio
]

all_grants = []
for org_id, corp_num in TARGETS:
    print(f"  取得中: {org_id} ({corp_num})")
    subsidies = fetch_subsidy(corp_num)
    grants = parse_subsidies(org_id, corp_num, subsidies)
    all_grants.extend(grants)
    print(f"    → {len(grants)}件の補助金情報")
    time.sleep(1.0)

df_grants = pd.DataFrame(all_grants)
print(df_grants)
```

---

## おわりに：公的資金の流れを可視化する意味

「この研究は本当に社会に届くのか」という問いに答えるには、技術の優劣だけでなく、資金・情報・組織の構造を合わせて見る必要がある。

gBizINFOは無料で、APIキーは数分で取れて、法人番号さえあればすべて引き出せる。使わない理由がない。

このプロジェクトの全体設計（BigQueryスキーマ・[ClinicalTrials.gov](http://ClinicalTrials.gov)同期スクリプト等）は別記事で公開予定。コードへの質問や分析サービスへの興味はプロフィールリンクから。