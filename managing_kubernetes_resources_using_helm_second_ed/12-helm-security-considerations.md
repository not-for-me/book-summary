# 12. Helm Security Considerations

## 챕터 개요 (3줄 요약)
- digital signature(PGP/GPG)로 Helm binary·chart의 data provenance·integrity를 검증하는 방법을 다룬다.
- 안전한 chart 개발(secure image, resource request/limit, secret 처리)의 best practice를 설명한다.
- RBAC(least privilege)와 secure chart repository(인증·TLS) 구성을 다룬다.

---

## 1. Data provenance & integrity
> provenance(출처)와 integrity(변조 여부)는 PGP/GPG digital signature로 검증하며, private key로 서명·public key로 검증한다.

- PGP: asymmetric(private 서명, public 검증). 서명=메시지 digest를 private key로 암호화.
- GPG key pair 생성: `gpg --full-generate-key` (Helm은 ECC 미지원 → RSA 사용).
- **Helm binary 검증**: `.asc` 다운로드 → `gpg --recv-key <fingerprint>` → `gpg --verify helm-*.tar.gz.asc helm-*.tar.gz`. `gpg --sign-key`로 certify 시 warning 제거.

---

## 2. Signing & verifying charts
> helm package --sign으로 chart를 서명해 .prov 파일을 만들고, 소비자는 public key로 helm verify/install --verify한다.

```
gpg --export > ~/.gnupg/pubring.gpg          # GPG v2+는 .gpg 레거시 포맷 export 필요
gpg --export-secret-keys > ~/.gnupg/secring.gpg
helm package --sign --key <name> --keyring ~/.gnupg/secring.gpg <chart>
helm verify --keyring ~/.gnupg/pubring.gpg guestbook-0.1.0.tgz
helm install guestbook <repo>/guestbook --verify --keyring ~/.gnupg/pubring.gpg
```
- .prov: chart metadata + sha256 hash + PGP signature. .tgz와 .prov 둘 다 publish.
- public key는 `gpg --send-key`로 key server 배포, 소비자는 `gpg --recv-key`.

---

## 3. Developing secure & stable charts
> secure image(digest·scan·non-root), resource request/limit, secret 처리를 통해 안전한 chart를 작성한다.

- **image**: tag(가변) 대신 **digest**(`@sha256:...`, 불변·MITM 방지). registry/Vuls/OpenSCAP로 취약점 scan, root·privileged 금지(필요 시 capability만 부여).
- **resource**: requests(최소)/limits(최대) 기본값 제공. namespace 차원은 LimitRange(container/pod/PVC)·ResourceQuota(namespace 총량).
- **secret**: 기본값 제공 금지, `required` 함수로 강제(randAlphaNum은 upgrade마다 재생성 주의). ConfigMap 아닌 Secret 사용. values 파일은 Git 노출 주의 → --set 또는 SOPS/git-crypt/Vault로 암호화.

---

## 4. RBAC & secure chart repositories
> least privilege RBAC로 user·service account 권한을 최소화하고, repository는 인증·TLS로 보호한다.

**RBAC** (least privilege):
```
kubectl create role pod-viewer --resource=pods --verb=get,list -n chapter12
kubectl create sa example -n chapter12
kubectl create rolebinding pod-viewers --role=pod-viewer --serviceaccount=chapter12:example -n chapter12
kubectl auth can-i list pods --as=system:serviceaccount:chapter12:example -n chapter12
```
- built-in role(view 등)보다 필요한 권한만 가진 custom Role/ClusterRole 권장. 앱 전용 SA 생성.

**Repository 보안**:
- HTTP(S): `helm repo add --username --password`(필요 시 --pass-credentials), cert 인증 --cert-file/--key-file, 사설 CA --ca-file.
- OCI: `helm registry login --username --password-stdin`(stdin으로 bash history 노출 방지).
- TLS 암호화(ChartMuseum --tls-cert/--tls-key 등), GitHub Pages는 TLS 기본.

---

## Summary (핵심 정리)
- provenance·integrity는 PGP/GPG 서명으로 검증하며, Helm binary는 .asc, chart는 .prov 파일로 verify한다.
- 안전한 chart는 image digest·취약점 scan·non-root, resource request/limit(LimitRange/ResourceQuota), Secret·암호화된 secret 처리를 따른다.
- RBAC는 least privilege로 전용 SA·custom role을 부여하고, kubectl auth can-i로 검증한다.
- chart repository는 basic/cert 인증과 TLS로 보호하며, OCI는 --password-stdin으로 자격 노출을 막는다.
