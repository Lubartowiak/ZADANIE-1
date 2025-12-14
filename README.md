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


Zastosowanie NetworkPolicy
kubectl apply -f network/

kubectl get networkpolicy -A


Screen: frontend-policy oraz backend-policy

Konfiguracja ResourceQuota
kubectl create -f limits/frontend-quota.yaml --validate=false
kubectl create -f limits/backend-quota.yaml --validate=false

kubectl get resourcequota -A


Screen (kluczowy): limity dla namespace frontend i backend

Konfiguracja HPA dla frontend
kubectl autoscale deployment frontend \
  -n frontend \
  --cpu-percent=50 \
  --min=1 \
  --max=10

kubectl get hpa -n frontend


Screen: HPA z zakresem 1–10 replik

Test obciążeniowy
kubectl get hpa -n frontend
kubectl get resourcequota -A


Screen: status HPA oraz ResourceQuota
