# SampleArgoHelm

# 🚀 實戰指南：從零在 Kind 與 Argo CD 上部署官方 Grafana

本指南記錄了如何安裝 Kind、建立本地 Kubernetes 叢集、安裝 Argo CD、將官方 Grafana Helm Chart 納入自建 Git 倉庫並透過 Argo CD 部署，最後成功於本機指定 IP 存取服務的完整步驟。

---

## 一、 環境準備：安裝 Kind

在開始之前，請先確保本機已安裝 **Docker**、**kubectl** 與 **Helm**，接著透過以下指令安裝 Kind（Kubernetes in Docker）：

### Linux (amd64 / x86_64)
```bash
# 下載 Kind 執行檔
[ $(uname -m) = x86_64 ] && curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.31.0/kind-linux-amd64

# 賦予執行權限並移動至系統路徑
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

### macOS (Apple Silicon / Intel)
```bash
brew install kind
```
*Verification:* 執行 `kind version` 確認版本資訊正常顯示，且確認 `docker ps` 能正常執行（無權限報錯）。

---

## 二、 建立 Kubernetes (Kind) 叢集與安裝 Argo CD

### 1. 建立 Kind 叢集
```bash
kind create cluster --name argo-lab
```
*Verification:* 執行 `kubectl cluster-info` 確認叢集運作正常。

### 2. 安裝 Argo CD
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```
*Verification:* 執行 `kubectl get pods -n argocd` 確認所有 Argo CD 核心 Pod 均達到 `1/1 Running`。

---

## 三、 將官方 Grafana Helm Chart 放入本機專案與 GitHub

為了完全掌控設定並避免內建 SQLite 在 Kind 環境中的檔案鎖定問題，我們直接拉取官方 Helm Chart 並放入專案中管理。

### 1. 下載官方 Helm Chart
```bash
# 加入官方 Repo
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 拉取官方 Chart 並解壓縮
helm pull grafana/grafana --untar
```
*Verification:* 確認當前目錄下多出了一個名為 `grafana` 的資料夾。

### 2. 移動並整理至專案目錄
將解壓縮後的資料夾放到你的 Git 專案中的 `charts/` 目錄下（重新命名為 `grafana-app`）：
```bash
mkdir -p charts
mv grafana charts/grafana-app
```
*Verification:* 確認專案路徑下包含 `charts/grafana-app/values.yaml` 與 `templates/` 目錄。

### 3. 推送到 GitHub 倉庫
將整理好的 Chart 提交並推送到你的 GitHub 遠端倉庫：
```bash
git add charts/grafana-app
git commit -m "feat: add official grafana helm chart to repository"
git push origin main
```
*Verification:* 打開你的 GitHub 倉庫，確認 `charts/grafana-app` 檔案結構完整上傳。

---

## 四、 透過 Argo CD Web UI 部署 Grafana

透過 Argo CD 的網頁介面將專案內的 Grafana 部署到 Kubernetes 中。

### 1. 存取 Argo CD Web UI
透過 `port-forward` 將 Argo CD 介面導向本機：
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```
*Verification:* 瀏覽器開啟 `https://localhost:8080`（忽略憑證警告）。預設帳號為 `admin`，密碼可透過以下指令取得：
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 --decode
```

### 2. 在 Argo CD 建立應用程式 (Application)
1. 登入 Argo CD 後，點擊左上角 **+ New App**。
2. 填寫基本設定：
   * **Application Name**: `grafana-demo`
   * **Project**: `default`
   * **SYNC POLICY**: 可選 `Automatic` 或手動
3. 填寫 Source（來源）：
   * **Repository URL**: 你的 GitHub 倉庫網址
   * **Revision**: `HEAD`（或 `main`）
   * **Path**: `charts/grafana-app`
4. 填寫 Destination（目的地）：
   * **Cluster URL**: `https://kubernetes.default.svc`
   * **Namespace**: `default`
5. 點擊畫面上方的 **Create** 建立。

### 3. 執行同步 (Sync)
在 Argo CD 介面點擊剛建立的 `grafana-demo` 應用程式，點擊 **Sync** 並確認。
*Verification:* 稍等 1 分鐘，執行 `kubectl get pods` 確認 Grafana Pod 狀態為 `1/1 Running`。

---

## 五、 透過 Port-Forward 存取 Grafana

由於官方 Chart 預設的 Service 外部通訊埠（Port）為 `80`，我們需要將流量正確轉發出來。

### 1. 綁定指定 IP 進行 Port-Forward
若要將流量綁定至特定 IP（例如 `10.8.9.50`）並對應到本機的 `3000` 埠：
```bash
kubectl port-forward --address 10.8.9.50 svc/grafana 3000:80
```
*Verification:* 終端機顯示 `Forwarding from 10.8.9.50:3000 -> 80` 即代表通道建立成功。

### 2. 取得管理者密碼並登入
執行以下指令將系統自動產生的 `admin` 密碼解密：
```bash
kubectl get secret --namespace default grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```
*Verification:* 
1. 打開瀏覽器，輸入網址：`http://10.8.9.50:3000`
2. 輸入帳號 `admin` 與剛解密取得的密碼，即可順利登入 Grafana 主畫面。 test