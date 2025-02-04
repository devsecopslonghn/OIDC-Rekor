# OIDC-Rekor
# **Checklist Triển Khai OIDC (Fulcio) và Transparency Log (Rekór) Trong Mạng Nội Bộ**

## **1. Chuẩn bị môi trường**
- [ ] Máy chủ Linux (Ubuntu/CentOS) với quyền sudo.
- [ ] Đảm bảo Docker và Docker Compose đã được cài đặt:
  ```bash
  sudo apt update && sudo apt install -y docker.io docker-compose
  ```
- [ ] Mở các cổng cần thiết:
  - `8080` cho Keycloak (OIDC)
  - `5555` cho Fulcio
  - `3000` cho Rekór
  - `8090` cho Trillian

## **2. Cài đặt OIDC Provider (Keycloak)**
- [ ] Chạy Keycloak với Docker:
  ```bash
  docker run -d --name keycloak -p 8080:8080 \
    -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD=admin \
    quay.io/keycloak/keycloak:latest start-dev
  ```
- [ ] Cấu hình Keycloak:
  - Tạo **Realm**: `sigstore`
  - Tạo **Client**: `fulcio`
  - Cho phép **Service Accounts** và kích hoạt `openid-connect`
  - Thêm Mapper để cung cấp `email`

## **3. Cài đặt Fulcio (CA OIDC)**
- [ ] Clone repository:
  ```bash
  git clone https://github.com/sigstore/fulcio.git && cd fulcio
  ```
- [ ] Chỉnh sửa `config.yaml` để sử dụng Keycloak:
  ```yaml
  OIDCIssuers:
    - issuer: "http://keycloak.local:8080/realms/sigstore"
      clientID: "fulcio"
      type: "email"
  ```
- [ ] Chạy Fulcio:
  ```bash
  docker run -d --name fulcio -p 5555:5555 \
    -v $(pwd)/config.yaml:/etc/fulcio-config.yaml \
    ghcr.io/sigstore/fulcio:latest --config /etc/fulcio-config.yaml
  ```

## **4. Cài đặt Rekór (Transparency Log)**
- [ ] Cài đặt Trillian Backend:
  ```bash
  docker run -d --name trillian-log-server -p 8090:8090 gcr.io/trillian-map-server
  ```
- [ ] Chạy Rekór:
  ```bash
  docker run -d --name rekor -p 3000:3000 \
    -e REKOR_STORAGE_TYPE="trillian" \
    -e TRILLIAN_LOG_SERVER="http://trillian-log-server:8090" \
    ghcr.io/sigstore/rekor-server:latest
  ```

## **5. Cấu hình Cosign để sử dụng dịch vụ nội bộ**
- [ ] Thiết lập biến môi trường:
  ```bash
  export SIGSTORE_FULCIO_URL=http://localhost:5555
  export SIGSTORE_REKOR_URL=http://localhost:3000
  export SIGSTORE_OIDC_ISSUER=http://keycloak.local:8080/realms/sigstore
  ```
- [ ] Ký Image mà không cần Key On-Prem:
  ```bash
  cosign sign myregistry.local/myimage:latest
  ```
- [ ] Xác minh chữ ký:
  ```bash
  cosign verify --certificate-identity=me@example.com myregistry.local/myimage:latest
  
