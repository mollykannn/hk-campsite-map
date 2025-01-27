# 香港營地地圖

互動式香港營地地圖，提供即時天氣資訊和營地詳細資料。

## 功能特點

- 📍 顯示所有香港營地位置
- 🌤️ 即時天氣資訊（溫度、雨量）
- 💨 即時風速資料
- 🔍 營地搜尋功能
- 📱 響應式設計，支援手機瀏覽
- 📍 即時位置追蹤
- 🔗 可分享營地連結
- 🗺️ 使用 OpenStreetMap 開源地圖

## 安裝

### 前置需求

- Node.js 16.x 或以上
- pnpm 8.x 或以上

### 安裝步驟

1. 克隆專案

```bash
git clone https://github.com/your-username/hk-campsite-map.git
cd hk-campsite-map
```

2. 安裝依賴

```bash
pnpm install
```

3. 啟動開發伺服器

```bash
pnpm dev
```

4. 建置生產版本

```bash
pnpm build
```

## 使用方法

### URL 參數
可以使用 URL 參數直接訪問特定營地：
```
/?place=TaiTanCampsite    # 顯示大灘營地
/?place=WongShekCampsite  # 顯示黃石營地
```