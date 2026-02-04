# 將 Podman Pod 複製到另一台機器

本指南說明如何將現有的 Podman pod 從您的 Bazzite 筆記本電腦複製（遷移/備份）到另一台 Linux 機器。

## 先決條件

*   **來源機器：** Bazzite（正在運行 Podman）
*   **目標機器：** 安裝了 Podman 的 Linux
*   **網路存取：** 建議機器之間可以透過 SSH 進行檔案傳輸。

## 步驟 1：確認您的 Pod

列出目前的 pod 以找到您想要複製的 pod 名稱。

```bash
podman pod ps
```

*在本指南中，我們將使用 `my-pod` 作為範例名稱。*

## 步驟 2：匯出 Pod 設定 (YAML)

Podman 可以產生一個與 Kubernetes 相容的 YAML 檔案，其中描述了您的 pod 及其容器。

```bash
podman generate kube my-pod > my-pod.yaml
```

*注意：如果您想包含憑證密鑰或其他特定設定，請查看 `podman generate kube --help` 以獲取更多選項。*

## 步驟 3：儲存容器映像檔 (Images)

目標機器需要您的 pod 所使用的映像檔。您可以將它們儲存為 tar 封存檔。

首先，列出映像檔以查看使用了哪些：
```bash
podman image ls
```

接著儲存相關的映像檔：
```bash
# 語法：podman save -o <輸出檔案.tar> <映像檔1> <映像檔2> ...
podman save -o my-pod-images.tar registry.fedoraproject.org/image1:latest docker.io/library/image2:tag
```

## 步驟 4：備份持久性資料 (Volumes)

如果您的 pod 使用了對應的卷 (volumes) 或綁定掛載 (bind mounts)，您必須手動複製目前的資料。

### 檢查 Volumes
檢查步驟 2 產生的 YAML 檔案 (`my-pod.yaml`) 或檢查 pod 容器以查看資料儲存位置。

```bash
podman pod inspect my-pod
```

### 選項 A：綁定掛載 (Host Directories)
如果您映射了主機目錄（例如 `-v /home/user/data:/data`），只需使用 `scp` 或 `rsync` 複製該目錄（參見步驟 5）。

### 選項 B：命名卷 (Named Volumes)
如果您使用了命名卷，則需要匯出其資料。
```bash
# 範例：將卷掛載到臨時容器並將內容打包成 tar
podman run --rm -v my-volume-name:/volume -v $(pwd):/backup alpine tar cvf /backup/my-volume-data.tar -C /volume .
```

## 步驟 5：傳輸檔案到目標機器

使用 `scp` 或 `rsync` 將檔案移動到新機器。

```bash
# 將 'user@target-machine-ip' 替換為您的實際目標詳細資訊
scp my-pod.yaml my-pod-images.tar my-volume-data.tar user@target-machine-ip:~/
```

## 步驟 6：在目標機器上還原

在 **目標機器** 上，執行以下步驟：

### 1. 載入映像檔
```bash
podman load -i my-pod-images.tar
```

### 2. 還原 Volumes
**對於綁定掛載 (Bind Mounts)：**
確保目錄結構存在於與來源相同的路徑，或編輯 `my-pod.yaml` 指向新位置。

**對於命名卷 (Named Volumes)：**
建立卷並還原資料：
```bash
podman volume create my-volume-name
podman run --rm -v my-volume-name:/volume -v $(pwd):/backup alpine tar xvf /backup/my-volume-data.tar -C /volume
```

### 3. 部署 Pod
使用 YAML 檔案重新建立 pod。

```bash
podman play kube my-pod.yaml
```

## 驗證

檢查 pod 是否在新機器上正常運作：

```bash
podman pod ps
podman pod logs my-pod
```
