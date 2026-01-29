# Manual Secret Creation Instructions
# Run these commands manually after applying the kustomization

## Setup Steps

**ArgoCD will apply the gitops resources automatically. You only need to create the secrets manually:**

1. **Generate a secure admin token:**
   ```bash
   openssl rand -base64 48
   # Example output: jiQ1QYoUSsnJy2532Cq+Bk3AnL3bAEYYpv8tsIyEsY2Kma3LJ6K6d/bRBj+5zO4l
   ```

2. **Create PostgreSQL user secret:**
   ```bash
   kubectl create secret generic vaultwarden-postgres-user \
     --from-literal=username=vaultwarden \
     --from-literal=password=YOUR_SECURE_PASSWORD \
     --namespace personal
   ```

3. **Create admin token secret:**
   ```bash
   kubectl create secret generic vaultwarden-admin-token \
     --from-literal=adminToken=YOUR_GENERATED_TOKEN_HERE \
     --namespace personal
   ```

4. **Restart vaultwarden to pick up secrets:**
   ```bash
   kubectl delete pod -n personal -l app.kubernetes.io/name=vaultwarden
   ```

5. **Access vaultwarden:**
   - **LAN access:** `http://vaultwarden.personal.svc.cluster.local:8080`
   - **Port forward:** `kubectl port-forward -n personal svc/vaultwarden 8080:8080`
   - **Admin panel:** `http://your-access-url/admin/?token=YOUR_GENERATED_TOKEN`

## Security Notes

- ⚠️  Replace `YOUR_SECURE_PASSWORD` with a strong password
- ⚠️  Replace `YOUR_GENERATED_TOKEN_HERE` with the token from step 2
- ✅  Secrets are NOT stored in git repository
- 🔄  Can be migrated to a secret manager later