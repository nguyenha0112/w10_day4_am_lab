# Image Signature Policy

Policy Controller only enforces namespaces labeled with:

```powershell
kubectl label namespace demo policy.sigstore.dev/include=true
```

Only add the label after GitHub Actions has built and signed the image with Cosign. If the label is added before the image is signed, the current API pods may be rejected during the next rollout.

The policy accepts images signed by GitHub Actions from this workflow:

```text
https://github.com/nguyenha0112/w10_day4_am_lab/.github/workflows/build-push.yml@refs/heads/main
```
