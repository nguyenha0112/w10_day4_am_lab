# Payments app

Sample workload for the `payments` tenant.

The app uses `ghcr.io/nguyenha0112/w10-api:0.0.1` so the existing image signature policy can apply to the tenant namespace.

Before enforcing Sigstore on `payments`, make sure this image tag has been signed by the current key-pair based GitHub Actions workflow. Otherwise admission can reject the workload.
