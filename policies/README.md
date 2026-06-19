# Image Signature Policy

Policy Controller only enforces namespaces labeled with:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true
```

Only add the label after GitHub Actions has built and signed the image with Cosign. If the label is added before the image is signed, the current API pods may be rejected during the next rollout.

This lab uses key-based Cosign signing:

```powershell
cosign generate-key-pair
```

Commit only `signing/cosign.pub`. Put the private key and password in GitHub repository secrets:

- `COSIGN_PRIVATE_KEY`: full content of `signing/cosign.key`
- `COSIGN_PASSWORD`: password used when generating the key pair

The public key from `signing/cosign.pub` is embedded in `policies/cluster-image-policy.yaml`.
