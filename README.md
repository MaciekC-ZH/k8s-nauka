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

## 📌 Etap 3: Konfiguracja (ConfigMap & Secret) i Pamięć Trwała (PV & PVC)

### 1. Separacja konfiguracji (12-Factor App)
* **Zasada jednego obrazu:** Ten sam obraz kontenera działa na `dev`, `staging` i `prod`. Nigdy nie hardkodujemy zmiennych ani haseł w obrazie.
* **ConfigMap:** Przechowuje jawne parametry konfiguracyjne (np. URL-e, porty, flagi środowiskowe).
* **Secret:** Przechowuje dane poufne (hasła, tokeny, klucze API) – w manifeście podawane jako `stringData` (kodowane automatycznie do base64).
* **Wstrzykiwanie:** Wartości trafiają do kontenera przez sekcję `env` (`configMapKeyRef` / `secretKeyRef`) lub jako zamontowane pliki w wolumenie.

### 2. Pamięć trwała (Storage)
* **Problem ulotności:** Kontenery w Podach są domyślnie bezstanowe (*ephemeral*). Po awarii Poda wszystkie dane na dysku znikają.
* **PersistentVolume (PV):** Fizyczny zasób dyskowy (np. wolumen w chmurze, katalog na węźle).
* **PersistentVolumeClaim (PVC):** Deklaratywne żądanie pamięci przez aplikację (np. „potrzebuję 1Gi w trybie ReadWriteOnce”).
* **Cykl życia:** K8s automatycznie paruje PVC z PV (`Bound`). Gdy Deployment podepnie PVC jako wolumen, dane w zamontowanym katalogu przetrwają restarty, skalowanie i zabicie Poda.

---

### Podręczne komendy diagnostyczne

| Polecenie                                          | Zastosowanie                                                      |
| ---------------------------------------------------|-------------------------------------------------------------------|
| `kubectl get cm,secret`                            | Podgląd utworzonych obiektów ConfigMap i Secret                   |
| `kubectl exec deploy/<nazwa> -- env`               | Sprawdzenie zmiennych środowiskowych wewnątrz kontenera           |
| `kubectl get pvc,pv`                               | Weryfikacja stanu roszczeń dyskowych (`STATUS: Bound`)            |
| `kubectl exec deploy/<nazwa> -- sh -c "<komenda>"` | Wykonanie szybkiego polecenia w kontenerze (np. inspekcja plików) |

## 📌 Etap 4: Projekt Warsztatowy – Aplikacja Dwuwarstwowa (Adminer + PostgreSQL)

### 1. Architektura rozwiązania
Projekt łączy wszystkie podstawowe mechanizmy Kubernetesa w jeden działający system składający się z warstwy danych oraz bezstanowej warstwy aplikacyjnej:

```text
[ Przeglądarka: localhost:8081 ]
              │
              ▼
┌───────────────────────────────┐
│  Service: adminer-service     │  (type: LoadBalancer, port 8081 -> 8080)
└──────────────┬────────────────┘
               │ (Load Balancing)
       ┌───────┴───────┐
       ▼               ▼
┌─────────────┐ ┌─────────────┐
│ Pod Adminer │ │ Pod Adminer │  (Deployment: 2 repliki, env: postgres-service)
└──────┬──────┘ └──────┬──────┘
       └───────┬───────┘
               │ (Wewnętrzny DNS klastra: postgres-service:5432)
               ▼
┌───────────────────────────────┐
│  Service: postgres-service    │  (type: ClusterIP, port 5432 -> 5432)
└──────────────┬────────────────┘
               │
               ▼
┌───────────────────────────────┐
│  Pod PostgreSQL (1 replika)   │  (ConfigMap + Secret)
└──────────────┬────────────────┘
               │ montuje /var/lib/postgresql/data
               ▼
┌───────────────────────────────┐
│  PVC: postgres-pvc (1Gi)      │  (Trwały dysk przeżywający restarty Podów)
└───────────────────────────────┘
```
### 2. Zastosowane komponenty i wnioski z wdrożenia

* **Warstwa danych (`06-projekt-db/`):**
  * **PostgreSQL (`Deployment`):** Uruchomiony w 1 replice (`postgres:16-alpine`), aby uniknąć konfliktów zapisu na pojedynczym wolumenie `ReadWriteOnce`.
  * **Separacja konfiguracji (`ConfigMap` & `Secret`):** Nazwa bazy (`POSTGRES_DB`) i użytkownik (`POSTGRES_USER`) pobierane z `postgres-config` (`configMapKeyRef`), a hasło (`POSTGRES_PASSWORD`) wstrzykiwane bezpiecznie z `postgres-secret` (`secretKeyRef`).
  * **Trwałość danych (`PVC`):** Wolumen `postgres-pvc` podmontowany pod `/var/lib/postgresql/data`. Zapisane rekordy przetrwały całkowite usunięcie Poda bazy danych.
  * **Izolacja sieciowa (`ClusterIP`):** Serwis `postgres-service` udostępnia bazę na porcie `5432` wyłącznie wewnątrz klastra.

* **Warstwa aplikacyjna (`07-projekt-app/`):**
  * **Adminer (`Deployment`):** Uruchomiony w wielu replikach (`adminer:latest`), korzystający ze zmiennej `ADMINER_DEFAULT_SERVER=postgres-service`.
  * **Service Discovery (CoreDNS):** Aplikacja nie używa adresów IP Podów – łączy się z bazą po stałej nazwie serwisu `postgres-service`, którą wewnętrzny DNS Kubernetesa tłumaczy na aktualny adres IP.
  * **Dostęp zewnętrzny (`LoadBalancer`):** Serwis `adminer-service` mapuje zewnętrzny port `8081` na wewnętrzny port kontenera `8080`.

---

### 3. Przeprowadzone testy odporności (Chaos Engineering)

| Scenariusz testowy | Wykonana akcja | Rezultat w klastrze |
| :--- | :--- | :--- |
| **1. Przetrwanie danych po awarii** | Usunięcie wszystkich Podów bazy i aplikacji (`kubectl delete pod -l ...`) | Deploymenty natychmiast odtworzyły Pody; dane w bazie pozostały nienaruszone dzięki `PVC`. |
| **2. Skalowanie dynamiczne** | Zwiększenie replik aplikacji (`kubectl scale ... --replicas=4`) | `Service` automatycznie zarejestrował 4 nowe adresy IP Podów w puli `Endpoints`. |
| **3. Błąd konfiguracji DNS** | Podmiana nazwy serwera bazy na nieistniejącą i przywrócenie poprawnej | Natychmiastowa diagnoza błędu połączenia oraz płynny powrót do pracy po korekcie manifestu (`kubectl apply`). |
