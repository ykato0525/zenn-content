---
title: "ClinicalTrials.gov APIをPythonで叩いて、がん領域のモダリティ別臨床試験を可視化する"
emoji: "🧬"
type: "tech"          # tech or idea
topics: ["python", "lifescience", "api", "bioinformatics", "datascience"]
published: true       # false にするとドラフト保存
---

# はじめに
「自分の研究しているモダリティが、今どれくらい臨床に進んでいるのか」——論文を追っていても、全体像はなかなか見えてこない。
ClinicalTrials.gov は世界最大の臨床試験登録データベースで、40万件以上の試験情報が公開されている。2023年にv2 APIがリリースされ、構造化されたJSONで大量データを取得できるようになった。

この記事では、Pythonを使って以下3点を実装したいと思います。
- ClinicalTrials.gov v2 APIからがん領域の臨床試験データを取得
- 介入名からモダリティを分類（抗体・ADC・低分子・核酸・細胞療法・遺伝子療法）
- モダリティ × Phase でヒートマップと積み上げ棒グラフを可視化

本記事のコードはgoogle colabで動作を確認しました。

### 環境・依存ライブラリ
```
pip install requests pandas matplotlib seaborn
````

# 1. ClinicalTrials.gov v2 APIの基本
エンドポイント
https://clinicaltrials.gov/api/v2/studies
v1との主な違い：

レスポンスがフラットなJSONになり扱いやすい
pageSize最大1000件（v1は100件）
nextPageTokenによるページネーション

主要クエリパラメータ
パラメータ説明例query.cond疾患・状態cancerquery.intr介入名antibodyfilter.overallStatus試験ステータスRECRUITING,COMPLETEDfields取得フィールドを絞るNCTId,Phase,InterventionName,...pageSize1ページあたりの件数（最大1000）1000pageToken次ページトークン（レスポンスから取得）

# 2. データ取得
## 2-1. 基本的なAPI呼び出し
```
import requests

BASE_URL = "https://clinicaltrials.gov/api/v2/studies"

def fetch_trials(query_cond: str, page_size: int = 1000, max_pages: int = 10) -> list[dict]:
    """
    ClinicalTrials.gov v2 APIからデータを取得する。
    ページネーションを処理して全件返す。
    """
    params = {
        "query.cond": query_cond,
        "filter.overallStatus": "RECRUITING,ACTIVE_NOT_RECRUITING,COMPLETED",
        "fields": (
            "NCTId,BriefTitle,Phase,OverallStatus,"
            "InterventionName,InterventionType,"
            "StartDate,PrimaryCompletionDate,Condition"
        ),
        "pageSize": page_size,
        "format": "json",
    }

    all_studies = []
    page_token = None
    page_count = 0

    while page_count < max_pages:
        if page_token:
            params["pageToken"] = page_token

        response = requests.get(BASE_URL, params=params, timeout=30)
        response.raise_for_status()
        data = response.json()

        studies = data.get("studies", [])
        all_studies.extend(studies)

        page_token = data.get("nextPageToken")
        page_count += 1

        print(f"  Page {page_count}: {len(studies)} 件取得（累計 {len(all_studies)} 件）")

        if not page_token:
            break

    return all_studies

```

## 2-2. がん領域のデータを取得
```
print("がん領域の臨床試験データを取得中...")
raw_studies = fetch_trials(query_cond="cancer OR neoplasm OR tumor OR carcinoma OR leukemia OR lymphoma", max_pages=20)
print(f"\n取得完了: {len(raw_studies)} 件")
````

# 3. DataFrameへの整形
APIレスポンスはネストされた構造なので、必要なフィールドをフラットに展開する。

```
import pandas as pd

def extract_fields(study: dict) -> dict:
    """1試験分のJSONから必要フィールドを抽出する"""
    proto = study.get("protocolSection", {})
    id_mod = proto.get("identificationModule", {})
    status_mod = proto.get("statusModule", {})
    design_mod = proto.get("designModule", {})
    arms_mod = proto.get("armsInterventionsModule", {})

    # Phase：リストで返る場合がある
    phases = design_mod.get("phases", [])
    phase = phases[0] if phases else "UNKNOWN"

    # 介入名：複数ある場合はすべて結合
    interventions = arms_mod.get("interventions", [])
    intervention_names = " | ".join([
        i.get("name", "") for i in interventions
    ])
    intervention_types = " | ".join([
        i.get("type", "") for i in interventions
    ])

    return {
        "nct_id": id_mod.get("nctId", ""),
        "title": id_mod.get("briefTitle", ""),
        "status": status_mod.get("overallStatus", ""),
        "phase": phase,
        "intervention_name": intervention_names,
        "intervention_type": intervention_types,
    }


df_raw = pd.DataFrame([extract_fields(s) for s in raw_studies])
print(df_raw.shape)
df_raw.head()
```

# 4. モダリティ分類
介入名のキーワードマッチングでモダリティを判定する。優先順位を設けて多重分類を避ける。

```
import re

MODALITY_RULES = [
    ("ADC",          r"antibody.drug conjugate|ADC|\-ADC\b|emtansine|deruxtecan"
                     r"|ozogamicin|vedotin|ravtansine|deruxtecan"),

    ("CAR-T/Cell",   r"CAR.T|chimeric antigen|cell therapy|TIL|NK cell|dendritic cell"),

    ("Gene Therapy", r"gene therapy|gene editing|CRISPR|AAV|lentiviral|retroviral|oncolytic"),

    ("Nucleic Acid", r"\bsiRNA\b|\bASO\b|\bshRNA\b|\bmiRNA\b|\bmRNA\b|antisense|oligonucleotide"),

    # 放射線：独立カテゴリ化
    ("Radiation",    r"radiation|radiotherapy|stereotactic|SBRT|IMRT|proton|brachytherapy"),

    # ホルモン療法：独立カテゴリ化
    ("Hormone",      r"fulvestrant|tamoxifen|letrozole|anastrozole|exemestane"
                     r"|enzalutamide|abiraterone|leuprolide|goserelin|bicalutamide"),

    # 抗体：語尾 -mab で一括対応
    ("Antibody",     r"mab\b|monoclonal antibody|bispecific|checkpoint inhibitor"
                     r"|anti.PD|anti.CTLA|anti.CD\d+|anti.HER"),

    # 低分子：語尾パターン + 個別薬剤名 
    ("Small Mol.",   r"inhibitor|kinase|small molecule"
                 r"|nib\b|parib\b|ciclib\b|lisib\b|rafenib\b"  # 語尾パターン拡張
                 r"|taxel|platin|fluorouracil|gemcitabine|temozolomide|capecitabine|pemetrexed"
                 r"|bortezomib|venetoclax|azacitidine|vorinostat|topotecan|irinotecan"
                 r"|everolimus|lenalidomide|RAD001|ixabepilone|celecoxib"),
]

def classify_modality(name: str) -> str:
    """介入名からモダリティを分類する"""
    if not isinstance(name, str) or name.strip() == "":
        return "Other"
    name_lower = name.lower()
    for modality, pattern in MODALITY_RULES:
        if re.search(pattern, name, re.IGNORECASE):
            return modality
    return "Other"


df_raw["modality"] = df_raw["intervention_name"].apply(classify_modality)

# 分類結果の確認
print(df_raw["modality"].value_counts())
```

# 5. Phase の整理
APIから返るPhaseは表記が揺れるため正規化する。

```
PHASE_ORDER = ["EARLY_PHASE1", "PHASE1", "PHASE2", "PHASE3", "PHASE4"]
PHASE_LABEL = {
    "EARLY_PHASE1":  "Early Ph1",
    "PHASE1":        "Phase 1",
    "PHASE2":        "Phase 2",
    "PHASE3":        "Phase 3",
    "PHASE4":        "Phase 4",
}

df = df_raw[df_raw["phase"].isin(PHASE_ORDER)].copy()
df["phase_label"] = df["phase"].map(PHASE_LABEL)
df["phase"] = pd.Categorical(df["phase"], categories=PHASE_ORDER, ordered=True)

print(f"Phase情報あり: {len(df)} 件 / 全{len(df_raw)} 件")
```

# 6. 可視化
## 6-1. モダリティ × Phase ヒートマップ
```
import matplotlib.pyplot as plt
import seaborn as sns
import matplotlib.ticker as ticker

# ピボットテーブル作成
pivot = df.groupby(["modality", "phase"]).size().unstack(fill_value=0)
pivot = pivot.reindex(columns=PHASE_ORDER, fill_value=0)
pivot.columns = [PHASE_LABEL[p] for p in pivot.columns]

# モダリティを合計件数で降順ソート
pivot = pivot.loc[pivot.sum(axis=1).sort_values(ascending=False).index]

fig, ax = plt.subplots(figsize=(12, 5))

sns.heatmap(
    pivot,
    annot=True,
    fmt="d",
    cmap="Blues",
    linewidths=0.5,
    linecolor="#e0e0e0",
    cbar_kws={"label": "Trial Count"},
    ax=ax,
)

ax.set_title(
    "Cancer Clinical Trials by Modality × Phase\n(ClinicalTrials.gov, Recruiting/Active/Completed)",
    fontsize=13,
    pad=14,
)
ax.set_xlabel("Phase", fontsize=11)
ax.set_ylabel("Modality", fontsize=11)
ax.tick_params(axis="x", rotation=30)
ax.tick_params(axis="y", rotation=0)

plt.tight_layout()
plt.savefig("heatmap_modality_phase.png", dpi=150, bbox_inches="tight")
plt.show()
```

## 6-2. 積み上げ棒グラフ（モダリティ別・Phase内訳）
```
PHASE_COLORS = {
    "Early Ph1": "#d0e8ff",
    "Phase 1":   "#93c4f0",
    "Ph 1/2":    "#5fa3e0",
    "Phase 2":   "#3a7fc1",
    "Ph 2/3":    "#2a5fa0",
    "Phase 3":   "#1a3f7a",
    "Phase 4":   "#0d1f40",
}

phase_labels_ordered = [PHASE_LABEL[p] for p in PHASE_ORDER]

fig, ax = plt.subplots(figsize=(12, 6))

bottom = pd.Series([0] * len(pivot), index=pivot.index)

for phase_label in phase_labels_ordered:
    if phase_label not in pivot.columns:
        continue
    values = pivot[phase_label]
    ax.bar(
        pivot.index,
        values,
        bottom=bottom,
        label=phase_label,
        color=PHASE_COLORS[phase_label],
        edgecolor="white",
        linewidth=0.5,
    )
    bottom += values

ax.set_title(
    "Cancer Clinical Trials by Modality (Phase breakdown)\n(ClinicalTrials.gov, Recruiting/Active/Completed)",
    fontsize=13,
    pad=14,
)
ax.set_xlabel("Modality", fontsize=11)
ax.set_ylabel("Number of Trials", fontsize=11)
ax.legend(title="Phase", bbox_to_anchor=(1.01, 1), loc="upper left", fontsize=9)
ax.yaxis.set_major_locator(ticker.MultipleLocator(500))
ax.tick_params(axis="x", rotation=15)

plt.tight_layout()
plt.savefig("bar_modality_phase.png", dpi=150, bbox_inches="tight")
plt.show()
```

# 7. 課題
Otherの大きさがやや問題です。Phase 1だけで2085件はSmall Mol.の887件を大きく上回っており、未分類の中に意味のある介入が混在している可能性があります。
記事に載せる場合は2つの選択肢があります：
① Otherを除外して掲載
主要6モダリティの比較に集中。図がクリーンになる。


# 8. 参考
- ClinicalTrials.gov API v2 Documentation
- ClinicalTrials.gov API Explorer