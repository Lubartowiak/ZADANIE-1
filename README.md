# Zadanie 1 – Kubernetes (część obowiązkowa)
Środowisko:
Ubuntu 24.04
Minikube v1.37.0
Kubernetes v1.34.0
Docker
CNI: Calico

# Utworzenie klastra Kubernetes (3 węzły)
```minikube start \
  --nodes=3 \
  --cni=calico \
  --network-plugin=cni \
  --container-runtime=docker
```
```
kubectl get nodes
```
<img width="543" height="112" alt="image" src="https://github.com/user-attachments/assets/e8cc9370-2b35-441c-84fc-21e175d2f5cd" />


# Utworzenie namespace’ów
```
kubectl create namespace frontend
kubectl create namespace backend
```
<img width="541" height="66" alt="image" src="https://github.com/user-attachments/assets/d1eafd5d-2c28-4d89-871e-36327e63fe6b" />

# Oznaczenie węzłów etykietami
```
kubectl label node minikube role=backend
kubectl label node minikube-m02 role=frontend
kubectl label node minikube-m03 role=frontend

kubectl get nodes --show-labels
```

<img width="647" height="216" alt="image" src="https://github.com/user-attachments/assets/83c83d5b-b37f-4eb1-814a-958c3f73869f" />

# Utworzenie struktury katalogów projektu
```
mkdir -p k8s/{frontend,backend,network,limits}
cd k8s
```
# Wdrożenie backend
```
kubectl apply -f backend/backend-deployment.yaml

kubectl apply -f backend/mysql-pod.yaml

kubectl apply -f backend/backend-service.yaml
kubectl apply -f backend/mysql-service.yaml

kubectl get pods -n backend -o wide
kubectl get svc -n backend
```

<img width="791" height="433" alt="image" src="https://github.com/user-attachments/assets/0a37d06c-7a0a-4743-bb0f-1ce235131794" />
<img width="792" height="172" alt="image" src="https://github.com/user-attachments/assets/5c4c3ee2-4f42-4b17-a733-78655f72845c" />

# Wdrożenie frontend
```
kubectl apply -f frontend/frontend-deployment.yaml

kubectl apply -f frontend/frontend-service.yaml

kubectl get pods -n frontend -o wide
kubectl get svc -n frontend
```

<img width="732" height="128" alt="image" src="https://github.com/user-attachments/assets/0ffff047-6399-4d6e-a93c-8f865d49d7e2" />
<img width="742" height="91" alt="image" src="https://github.com/user-attachments/assets/b5396e60-e2e5-4692-bd91-0989e07c2384" />
<img width="796" height="196" alt="image" src="https://github.com/user-attachments/assets/d3e336c2-72dc-4c81-bbb5-60cbc65d0dda" />


# Zastosowanie NetworkPolicy
```
kubectl apply -f network/

kubectl get networkpolicy -A
```

<img width="572" height="112" alt="image" src="https://github.com/user-attachments/assets/4b98be10-d045-4aab-93d9-6276e1b4e2fc" />


# Konfiguracja ResourceQuota
```
kubectl create -f limits/frontend-quota.yaml --validate=false
kubectl create -f limits/backend-quota.yaml --validate=false

kubectl get resourcequota -A
```

<img width="767" height="245" alt="image" src="https://github.com/user-attachments/assets/8040ff41-0e5c-432d-97f5-8f72c57c4a31" />
<img width="786" height="63" alt="image" src="https://github.com/user-attachments/assets/5f68ea77-98e5-4475-9949-677a68266ce5" />


# Konfiguracja HPA dla frontend
```
kubectl autoscale deployment frontend \
  -n frontend \
  --cpu-percent=50 \
  --min=1 \
  --max=10

kubectl get hpa -n frontend
```

<img width="795" height="185" alt="image" src="https://github.com/user-attachments/assets/b8936e19-4a0f-47b3-be15-0a0e33779535" />


# Test obciążeniowy
```
Uruchomienie testu obciążeniowego
while true; do wget -qO- http://frontend-svc.frontend.svc.cluster.local; done
kubectl get hpa -n frontend
kubectl get resourcequota -A
```

<img width="795" height="207" alt="image" src="https://github.com/user-attachments/assets/e1a79d14-670e-43c0-9ac9-f1b19be9ab2e" />
<img width="795" height="156" alt="image" src="https://github.com/user-attachments/assets/d814dc15-84ff-49bb-93eb-2759e7707299" />
<img width="818" height="491" alt="image" src="https://github.com/user-attachments/assets/3c167192-4d15-41b6-ac45-c8bc58a820b9" />


# Zadanie 1 – część nieobowiązkowa
# 1. Czy możliwa jest aktualizacja aplikacji frontend, gdy działa HPA?

Tak, możliwa jest aktualizacja aplikacji frontend (np. zmiana wersji obrazu kontenera), nawet gdy Deployment jest objęty autoskalerem HPA.

HPA skaluje jedynie liczbę replik Deploymentu w odpowiedzi na metryki (np. CPU), natomiast sam proces aktualizacji obrazu kontenera jest realizowany przez mechanizm Deployment oraz strategię rollingUpdate. Oba mechanizmy działają niezależnie i są ze sobą kompatybilne.

# Potwierdzenie w dokumentacji Kubernetes:
https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/#how-does-a-horizontalpodautoscaler-work

# 2. Strategia rollingUpdate dla Deploymentu frontend
Przykładowa strategia rollingUpdate

Dla Deploymentu frontend można zastosować następujące parametry strategii aktualizacji:
```
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
    maxSurge: 1
```
a) Gwarancja co najmniej 2 aktywnych Podów
Przy początkowej liczbie 3 replik:
-maxUnavailable: 1 oznacza, że w trakcie aktualizacji maksymalnie jeden Pod może być niedostępny,
-zapewnia to, że co najmniej 2 Pody frontend są zawsze aktywne.

b) Brak przekroczenia limitów namespace frontend
Dla przestrzeni nazw frontend obowiązują limity:
-maksymalnie 10 Podów,
-1 CPU,
-1.5 GiB RAM.

Parametr maxSurge: 1 powoduje, że podczas aktualizacji może powstać tylko jeden dodatkowy Pod, co:
-nie powoduje przekroczenia limitu liczby Podów,
-nie powoduje przekroczenia limitów CPU i RAM, ponieważ każdy Pod ma zdefiniowane niskie requests zasobów.

c) Korelacja strategii rollingUpdate z HPA

Nie ma konieczności modyfikowania konfiguracji autoskalera HPA w związku z zastosowaną strategią rollingUpdate, ponieważ:
-HPA skaluje Deployment w granicach 1–10 replik,
-strategia rollingUpdate generuje maksymalnie jedną dodatkową replikę,
-oba mechanizmy nie powodują konfliktu ani przekroczenia limitów ResourceQuota.

W związku z tym konfiguracja HPA może pozostać bez zmian.

# Uzasadnienie doboru parametrów

Zastosowana strategia rollingUpdate zapewnia nam ciągłość działania aplikacji frontend podczas aktualizacji, nie powodując przerw w dostępności usługi. Jednocześnie dobrane parametry nie naruszają wcześniej zdefiniowanych ograniczeń zasobów oraz są w pełni kompatybilne z działającym autoskalerem HPA.

