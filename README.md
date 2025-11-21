# 注音測驗系統 (Zhuyin Quiz)

基於 Vue 3 + Vite 的注音測驗單頁應用程式。

## 開發

```bash
# 安裝依賴
npm install

# 啟動開發伺服器
npm run dev

# 建置生產版本
npm run build

# 預覽生產版本
npm run preview
```

## 部署到 GitHub Pages

### 方法一：使用 GitHub Actions 自動部署（推薦）

1. **將程式碼推送到 GitHub**
   ```bash
   git init
   git add .
   git commit -m "Initial commit"
   git branch -M main
   git remote add origin https://github.com/你的用戶名/zhuyin-quiz.git
   git push -u origin main
   ```

2. **啟用 GitHub Pages**
   - 前往 GitHub 倉庫的 **Settings** → **Pages**
   - 在 **Source** 選擇 **GitHub Actions**
   - 儲存設定

3. **修改 base path（如果倉庫名稱不是 `zhuyin-quiz`）**
   - 編輯 `vite.config.js`
   - 將 `base: '/zhuyin-quiz/'` 改為你的倉庫名稱，例如：`base: '/你的倉庫名稱/'`

4. **自動部署**
   - 每次推送到 `main` 分支時，GitHub Actions 會自動建置並部署
   - 部署完成後，網站會出現在：`https://你的用戶名.github.io/zhuyin-quiz/`

### 方法二：手動部署

1. **建置專案**
   ```bash
   npm run build
   ```

2. **推送到 gh-pages 分支**
   ```bash
   # 安裝 gh-pages（如果還沒安裝）
   npm install --save-dev gh-pages
   
   # 在 package.json 添加部署腳本
   # "deploy": "gh-pages -d dist"
   
   # 執行部署
   npm run deploy
   ```

3. **在 GitHub 設定 Pages**
   - 前往 **Settings** → **Pages**
   - 選擇 **gh-pages** 分支作為來源

## 注意事項

- 如果您的 GitHub 倉庫名稱不是 `zhuyin-quiz`，請記得修改 `vite.config.js` 中的 `base` 路徑
- 如果倉庫名稱是 `你的用戶名.github.io`（個人或組織頁面），請將 `base` 設為 `'/'`
