# Splunk Web Access Security Lab

使用 Splunk Enterprise 分析合成 Apache Web access log，練習資料匯入、欄位擷取、SPL 統計、異常探索與儀表板建置。

> **資料說明：**本專案使用 19 筆自行建立的合成日誌，不含真實使用者或生產環境資料。偵測結果僅展示分析方法，不代表實際攻擊或正式安全告警。

## 專案成果

- 將 `data/access.log` 上傳至 Splunk，使用 `index=main`、`sourcetype=access_combined`。
- 驗證 Splunk 自動擷取 `clientip`、`method`、`uri`、`status`、`bytes`、`useragent` 等欄位。
- 建立儀表板 **Web Access Security Overview**，包含 HTTP 狀態碼分布、HTTP 錯誤率、重複登入失敗與可疑路徑探測四個面板。
- 將查詢保存於 [`spl/`](spl/)，可在 Splunk Search & Reporting 重現。

## 環境與匯入設定

- Splunk Enterprise 10.6.1.1（依實作畫面記錄）
- 資料來源：Apache combined 格式的合成 access log
- Index：`main`
- Sourcetype：`access_combined`
- 日誌時間：2026-11-02 09:00–09:05，時區 `+0800`
- 事件數：19

在 Splunk Web 開啟 **Settings → Add Data → Upload**，上傳 `data/access.log`，確認 sourcetype 為 `access_combined`、index 為 `main` 後提交。搜尋時選擇 **All time**，避免相對時間範圍排除這批舊事件。

## 分析結果

| 分析 | 結果 | 解讀 |
| --- | ---: | --- |
| HTTP 錯誤率（status ≥ 400） | 9 / 19 = **47.37%** | 樣本特意包含錯誤回應，比例不能當成真實服務基準。 |
| HTTP 200 | 10 | 成功回應數。 |
| HTTP 401 | 3 | 三次登入失敗，皆來自 `203.0.113.12`，路徑為 `/login`。 |
| 可疑路徑探測 | 4 | `198.51.100.24` 請求 `/admin`、`/wp-admin`、`/.env` 與 `/../../etc/passwd`。 |

## SPL 查詢

| 檔案 | 用途 |
| --- | --- |
| [`spl/01_field_validation.spl`](spl/01_field_validation.spl) | 檢視擷取欄位與事件明細 |
| [`spl/02_http_error_rate.spl`](spl/02_http_error_rate.spl) | 計算 4xx/5xx 錯誤數與錯誤率 |
| [`spl/03_status_distribution.spl`](spl/03_status_distribution.spl) | 統計 HTTP 狀態碼分布 |
| [`spl/04_repeated_login_failures.spl`](spl/04_repeated_login_failures.spl) | 找出至少 3 次 401 的來源 IP |
| [`spl/05_suspicious_path_probing.spl`](spl/05_suspicious_path_probing.spl) | 彙整管理頁、敏感檔案與路徑穿越探測 |

### 指標邏輯與實務注意

- 錯誤率以 HTTP status `>=400` 定義，按所有事件計算比例。
- 登入失敗門檻 `>=3` 是為這份小型示範資料設定的固定門檻。正式環境應用時間窗口、帳號維度、流量基準和告警降噪策略校準。
- 可疑 URI 規則是簡化示範，可能有誤報與漏報；正式偵測需考慮 URL 編碼、代理伺服器、應用路由與 WAF 情境。
- 合成樣本只有 19 筆且刻意植入異常，錯誤率與探測比例不具統計代表性。

## 儀表板

儀表板名稱：**Web Access Security Overview**

1. **HTTP Status Code Distribution**：依 status 統計請求數，長條圖。
2. **HTTP Error Rate**：單一數值顯示 47.37%。
3. **Repeated Login Failures**：顯示來源 IP、失敗次數與目標路徑。
4. **Suspicious Path Probing**：顯示來源 IP、探測數、路徑與回應碼。

建議將儀表板總覽與各分析面板截圖放入 [`screenshots/`](screenshots/)。截圖中避免保留個人帳號或其他本機資訊。

## 專案結構

```text
splunk-web-access-security/
├── README.md
├── data/
│   └── access.log
├── screenshots/
│   └── README.md
└── spl/
    ├── 01_field_validation.spl
    ├── 02_http_error_rate.spl
    ├── 03_status_distribution.spl
    ├── 04_repeated_login_failures.spl
    └── 05_suspicious_path_probing.spl
```

## 後續延伸

- 增加較長時間跨度與正常流量比例，建立有代表性的基準。
- 將重複登入失敗偵測改為滑動時間窗口，並評估告警門檻。
- 擴充不同日誌來源（例如 Syslog 或認證日誌），比較跨來源事件。
- 將 dashboard、搜尋與欄位設定整理成可部署的 Splunk app 配置。
