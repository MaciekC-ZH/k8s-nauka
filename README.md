# Kubernetes (K8s) – Fundamenty i Notatki Inżynierskie

Repozytorium dokumentujące proces nauki orkiestracji kontenerów, architektury klastrów oraz wdrażania aplikacji w ekosystemie Kubernetes.

---

## 1. Dlaczego Kubernetes? (Docker vs K8s)

* **Pojedynczy Docker:** Świetny do uruchamiania kontenerów na jednej maszynie (lokalnie / w testach).
* **Problem skali produkcyjnej:** Gdy aplikacja działa na 10–50 maszynach wirtualnych (np. AWS EC2), Docker sam z siebie nie poradzi sobie z:
  * **Autonaprawą:** Jeśli serwer ulegnie awarii, kontenery muszą zostać natychmiast przeniesione na inną maszynę.
  * **Skalowaniem:** Zwiększaniem i zmniejszaniem liczby kontenerów w zależności od obciążenia (CPU/RAM/ruch).
  * **Routingiem ruchu:** Balansowaniem zapytań użytkowników pomiędzy wieloma kopiami tej samej aplikacji.
  * **Zero-Downtime Deployment:** Płynną aktualizacją aplikacji wersja po wersji bez przestojów w działaniu.

**Kubernetes (K8s)** to orkiestrator kontenerów. Traktuje pulę maszyn jak jeden wielki komputer i zarządza cyklem życia kontenerów.

---

## 2. Kluczowa Zasada: Stan Pożądany (*Desired State*)

Kubernetes działa deklaratywnie w ciągłej pętli kontrolnej:
1. Użytkownik deklaruje stan pożądany w pliku YAML (np. *„Chcę dokładnie 3 kopie kontenera X”*).
2. K8s bezustannie porównuje **stan aktualny** ze **stanem pożądanym**.
3. W razie rozbieżności (np. padnięcie kontenera), K8s natychmiast podejmuje akcję naprawczą.

---

## 3. Architektura Klastra

Klaster dzieli się na dwa typy węzłów (*nodes*):

```text
[ Ty / kubectl / CI/CD ]
       │ (HTTPS / port 6443)
       ▼
┌───────────────────────── CONTROL PLANE ─────────────────────────┐
│                                                                 │
│   kube-apiserver  ◄────►  etcd (baza stanu klucz-wartość)       │
│        ▲                                                        │
│        ├──────────────►  kube-scheduler (wybór węzła)           │
│        └──────────────►  kube-controller-manager (strażnik)     │
└────────┬────────────────────────────────────────────────────────┘
         │
         │ (zarządzanie węzłami)
         ▼
┌──────────────────────── WORKER NODE ────────────────────────────┐
│  kubelet (agent lokalny) ──► containerd (silnik kontenerów)     │
│  kube-proxy (reguły sieciowe i routing IP)                      │
│                                                                 │
│  ┌─────────────────────── POD ───────────────────────────────┐  │
│  │  [Kontener App] (wspólne IP Poda, porty i wolumeny)       │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### A. Control Plane (Mózg klastra)
* **`kube-apiserver`:** Jedyny punkt wejścia do klastra. Obsługuje zapytania API, autoryzuje i przetwarza polecenia `kubectl`.
* **`etcd`:** Niezwykle szybka, rozproszona baza danych klucz-wartość. Zapisuje kompletny stan klastra.
* **`kube-scheduler`:** Przydziela nowo utworzone Pody do konkretnych Worker Nodes na podstawie dostępnych zasobów (CPU/RAM).
* **`kube-controller-manager`:** Pętla kontrolna pilnująca stanu pożądanego (np. reaguje na śmierć węzła lub Poda).

### B. Worker Nodes (Węzły wykonawcze)
* **Container Runtime (np. `containerd`):** Silnik odpowiedzialny za pobieranie obrazów i uruchamianie kontenerów.
* **`kubelet`:** Agent działający na węźle, odbiera polecenia z `kube-apiserver` i nadzoruje stan kontenerów.
* **`kube-proxy`:** Odpowiada za reguły sieciowe (firewall, IP tables) i trasowanie ruchu do odpowiednich Podów.

---

## 4. Czym jest Pod?

* **Najmniejsza, niepodzielna jednostka wykonawcza** w Kubernetesie (K8s nie zarządza bezpośrednio czystym kontenerem).
* **Kapsuła / otoczka** dla jednego lub rzadziej kilku ściśle powiązanych kontenerów.
* **Wspólne zasoby:** Kontenery wewnątrz jednego Poda dzielą ten sam adres IP, przestrzeń portów (`localhost`) oraz wolumeny dyskowe.
* **Śmiertelność (*Ephemeral*):** Pody nie są naprawiane. Gdy Pod ulegnie awarii, zostaje zniszczony, a w jego miejsce powstaje nowy (z nowym adresem IP).

### Anatomia manifestu YAML (4 filary)

Każdy plik w K8s opiera się na 4 polach:
1. `apiVersion` – wersja API obsługująca dany zasób.
2. `kind` – typ obiektu (np. `Pod`, `Deployment`, `Service`).
3. `metadata` – metadane identyfikacyjne (`name`, `labels`, `namespace`).
4. `spec` – techniczna specyfikacja stanu pożądanego (obraz, porty, limity, środowisko).

---

## 5. Podstawowe komendy diagnostyczne (CLI)

```bash
# Sprawdzenie dostępnych węzłów w klastrze
kubectl get nodes

# Wdrożenie manifestu YAML do klastra
kubectl apply -f sciezka/do/pliku.yaml

# Lista uruchomionych Podów
kubectl get pods

# Szczegółowa inspekcja Poda (Events, status, błędy kontenera)
kubectl describe pod <nazwa-poda>

# Usunięcie Poda
kubectl delete pod <nazwa-poda>
```
## 📌 Etap 2: Deploymenty, Skalowanie i Zarządzanie Wydaniami

### 1. Kluczowe koncepcje

* **Dlaczego nie używamy „nagich” Podów:** 
  Pojedynczy Pod (`kind: Pod`) jest jednostką śmiertelną (ephemeral) bez mechanizmu samoleczenia. Po awarii procesu lub usunięciu węzła Pod umiera na stałe.
* **Rola Deploymentu:**
  Zarządza deklaratywnym stanem aplikacji (*Desired State*). Zapewnia **samoleczenie** (odtworzenie uszkodzonych instancji) oraz **skalowanie**.
* **Deployment vs ReplicaSet:**
  Deployment zarządza wersjami i strategią wdrożeń (Rolling Update), tworząc pod spodem obiekt **ReplicaSet**, który bezpośrednio odpowiada za pilnowanie stałej liczby działających Podów (`replicas: N`).
* **Rolling Update (Wdrożenie bez przestojów):**
  Domyślna strategia w K8s: najpierw uruchamiany jest nowy Pod z nową wersją obrazu, czeka na osiągnięcie gotowości, a dopiero potem ubijany jest stary Pod. Użytkownik nie odczuwa przerwy w działaniu.
* **Configuration Drift (Rozjazd stanu):**
  Użycie doraźnego cofnięcia `kubectl rollout undo` zmienia stan w klastrze, ale nie zmienia pliku `.yaml` na dysku. Kolejne wywołanie `kubectl apply` nadpisze cofniętą wersję wartością z pliku. W metodyce GitOps po awaryjnym cofnięciu w klastrze zawsze należy wykonać `git revert` w repozytorium.

---

### 2. Przykładowy manifest Deploymentu

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: moj-deployment
  labels:
    app: moja-aplikacja
spec:
  replicas: 3
  selector:
    matchLabels:
      app: moja-aplikacja
  template:
    metadata:
      labels:
        app: moja-aplikacja
    spec:
      containers:
        - name: app-kontener
          image: nginx:alpine
          ports:
            - containerPort: 80
```
| Polecenie                                     | Zastosowanie                                                      |
|-----------------------------------------------|-------------------------------------------------------------------|
| kubectl apply -f <plik.yaml>                  | Wdrożenie lub aktualizacja deklaratywna zasobów z pliku           |
| kubectl get deployments                       | Lista aktywnych Deploymentów, liczba żądanych i dostępnych replik |
| kubectl get pods                              | Lista Podów (zwracaj uwagę na kolumny STATUS oraz AGE)            |
| kubectl delete pod <nazwa-poda>               | Ręczne usunięcie Poda (test mechanizmu samoleczenia)              |
| kubectl port-forward pod/<nazwa> 8080:80      | Tymczasowy tunel diagnostyczny z localhost do portu w Podzie      |
| kubectl scale deployment <nazwa> --replicas=5 | Doraźne wyskalowanie liczby Podów w locie                         |
| kubectl rollout status deployment <nazwa>     | Śledzenie procesu wymiany Podów na żywo podczas aktualizacji      |
| kubectl rollout history deployment <nazwa>    | Sprawdzenie historii wdrożonych rewizji                           |
| kubectl rollout undo deployment <nazwa>       | Awaryjne wycofanie do poprzedniej rewizji                         |