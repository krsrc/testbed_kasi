# Update CANFAR pods

```bash
helm -n skaha-system upgrade --install --values my-posix-mapper-local-values-file.yaml posixmapper science-platform/posixmapper
helm -n skaha-system upgrade --install --values my-skaha-local-values-file.yaml skaha science-platform/skaha
helm -n skaha-system upgrade --install --dependency-update --values my-scienceportal-local-values-file.yaml scienceportal science-platform/scienceportal
helm -n skaha-system upgrade --install --dependency-update --values {my-storage-ui-values-file} storage-ui science-platform-client/storage-ui
```
