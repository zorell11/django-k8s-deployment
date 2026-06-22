# Dokumentácia projektu: Django na Kubernetes (AWS)

Tento dokument sumarizuje postup nasadenia Django aplikácie do Kubernetes klastra na AWS inštancii.

---

## 1. Postup nasadenia

### Fáza A: Lokálny vývoj (MacBook)

1.  **Vytvorenie aplikácie:** Django aplikácia s `Dockerfile`.
2.  **Build a Push:** Stavba obrazu pre cieľovú architektúru a nahratie na Docker Hub.
    - `docker build --platform linux/amd64 -t tvoje_meno/django-k8s:v1 .`
    - `docker push tvoje_meno/django-k8s:v1`
3.  **Lokálne testovanie:** Overenie cez `kubectl port-forward`.

### Fáza B: Nasadenie na AWS (Cloud)

1.  **Infraštruktúra:** AWS EC2 inštancia s nainštalovaným MicroK8s.
2.  **Konfigurácia:** Aplikovanie súborov `deployment.yaml` a `service.yaml`.
3.  **Network:** Otvorenie portu 30000 v AWS Security Groups (Inbound rules).

---

## 2. Zoznam príkazov (Cheat Sheet)

### Docker

- **Build (pre AWS/amd64):** `docker build --platform linux/amd64 -t tvoje_meno/django-k8s:v1 .`
- **Login:** `docker login`
- **Push:** `docker push tvoje_meno/django-k8s:v1`

### Kubernetes (na AWS VM)

- **Alias pre kubectl (aby si nepísal microk8s):**
  `sudo snap alias microk8s.kubectl kubectl`
- **Aplikovanie zmien:** `kubectl apply -f .`
- **Reštart deploymentu:** `kubectl rollout restart deployment django-app`

### Debugging a logy

- **Zoznam podov:** `kubectl get pods -A`
- **Detailný stav podu:** `kubectl describe pod <názov-podu>`
- **Logy (aktívne):** `kubectl logs <názov-podu>`
- **Logy (predchádzajúce - ak padlo):** `kubectl logs <názov-podu> --previous`

### Údržba

- **Zmazanie zombie podov:** `kubectl delete pods --field-selector status.phase!=Running`
- **Reštart klastra:** `sudo microk8s stop && sudo microk8s start`

---

## 3. Riešenie kritických problémov (Troubleshooting)

| Problém              | Príčina                               | Riešenie                                     |
| :------------------- | :------------------------------------ | :------------------------------------------- |
| `exec format error`  | Zlá architektúra (ARM64 vs x86_64)    | Použiť `--platform linux/amd64` pri builde   |
| `DiskPressure`       | Nedostatok miesta/zdrojov na disku    | `docker system prune -a -f` a reštart K8s    |
| `Connection refused` | MicroK8s nie je prístupný ako kubectl | Alias: `snap alias microk8s.kubectl kubectl` |
| `Pending` stav       | Cluster nie je pripravený             | `microk8s status --wait-ready`               |

---

## 4. Konfigurácia (YAML)

### Deployment (`deployment.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: django-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: django-app
  template:
    metadata:
      labels:
        app: django-app
    spec:
      containers:
        - name: django-app
          image: zorell/django-k8s:v1
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
```
