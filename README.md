# VM para deploy de postgre en local y procesos de carga desde telegram

Preparación de la VM de Ubuntu (en VirtualBox)

    Configura la VM:
        RAM: 8GB
        CPU: 4 núcleos
        Adaptador de Red (Adaptador 1): NAT (para reenvío de puertos a localhost)
        Configura IP estática: 
        Edita /etc/netplan/01-network-manager-all.yaml 
        - check ip con comando: ip a
        - check de puerta de enlace con: ip r | grep default
        sudo netplan apply
# En la Terminal de la VM de Ubuntu

sudo apt update && sudo apt install -y docker.io
sudo usermod -aG docker $USER
newgrp docker

# Instalar kubectl:
sudo apt install curl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Instalar minikube

curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube-linux-amd64 /usr/local/bin/minikube

# Iniciar minikube

minikube start --driver=docker --memory=4000mb --cpus=3

kubectl get nodes (espera Ready)

# Instalar ArgoCD

curl -sSL -o argocd-linux-amd64 https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
sudo install -m 555 argocd-linux-amd64 /usr/local/bin/argocd

# Desplegar ArgoCD

kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd (espera todos Running).

# Acceder a Argo CD UI y obtener contraseña:

minikube service argocd-server -n argocd # Abre UI en navegador
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo # Muestra contraseña

admin
yREDJrgXqKjsCOGZ

# Crear Aplicación Argo CD (Web UI):

Nombre: postgres-db
Proyecto: default
Sync Policy: Automatic (con Auto-PRUNE, SELF HEAL)
Repo URL: Tu URL de Git.
Revision: HEAD o main.
Path: my-kubernetes-manifests (o la carpeta que contenga postgres-k8s.yaml).
Cluster: in-cluster.
Namespace: default.
Haz clic en CREATE. Espera a que esté Healthy.

# Derivar puertos para conectar argocd con el cluster

kubectl port-forward argocd-server-66ff958769-tgr6g 8080:8080 -n argocd
kubectl port-forward postgres-deployment-58764db864-n2969 5432:5432

# Configurar el fichero postgresql.conf y pg_hba.conf para permitir conexiones externas

Crea un fichero de config(manifests/postgres-config.yaml) con el contenido:

apiVersion: v1
kind: ConfigMap
metadata:
  name: postgres-config
data:
  postgresql.conf: |
    # PostgreSQL configuration file for Kubernetes
    listen_addresses = '*' # Escuchar en todas las IPs
    max_connections = 100
    # SSL Configuration (desactivado para simplificar localmente)
    ssl = off
    #ssl_cert_file = 'server.crt' # Comentado, ya que SSL está off
    #ssl_key_file = 'server.key'  # Comentado, ya que SSL está off
    password_encryption = md5 # Usar MD5 para la contraseña

    # Logging
    log_connections = on
    log_disconnections = on
    log_directory = 'pg_log'
    log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
    log_line_prefix = '%m [%p] %q%u@%d '
    log_timezone = 'UTC'

    # Resource usage (basic tuning)
    shared_buffers = 128MB
    effective_cache_size = 512MB

  pg_hba.conf: |
    # TYPE  DATABASE        USER            ADDRESS                 METHOD
    # Regla por defecto para localhost (dentro del pod)
    local   all             all                                     trust
    host    all             all             127.0.0.1/32            scram-sha-256
    host    all             all             ::1/128                 scram-sha-256

    # Permite conexiones desde cualquier IP interna de K8s sin SSL (para simplificar)
    host    all             all             0.0.0.0/0               md5

    # OPCIONAL: Si quisieras requerir SSL para alguna conexión, sería así:
    # hostssl criptodb        admincar        0.0.0.0/0               md5


Modifica el postgres-k8s.yaml para que lo use:

# --- 1. Secret para la contraseña de PostgreSQL ---
apiVersion: v1
kind: Secret
metadata:
  name: postgres-credentials
type: Opaque
stringData:
  POSTGRES_USER: admincar
  POSTGRES_PASSWORD: TU_CONTRASEÑA_SEGURA # <-- ¡CAMBIA ESTO A UNA CONTRASEÑA MÁS SEGURA!
  POSTGRES_DB: criptodb

---
# --- 2. PersistentVolumeClaim (PVC) para los datos de PostgreSQL ---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi # <--- Ajusta esto si necesitas más o menos espacio
  storageClassName: standard # Minikube usa 'standard' por defecto

---
# --- 3. Deployment para el servidor PostgreSQL (con ConfigMap para configuración) ---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres-deployment
  labels:
    app: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:16-alpine
        ports:
        - containerPort: 5432
        envFrom:
        - secretRef:
            name: postgres-credentials
        volumeMounts: # <--- MONTAR VOLÚMENES DEL PVC Y CONFIGMAP
        - name: postgres-storage
          mountPath: /var/lib/postgresql/data # Aquí se guardan los datos de la DB
        - name: postgres-config-volume # Montaje del ConfigMap para los archivos de configuración
          mountPath: /etc/postgresql/postgresql.conf # Monta postgresql.conf
          subPath: postgresql.conf # Especifica que es el archivo 'postgresql.conf' del ConfigMap
        - name: postgres-config-volume # Montaje del ConfigMap para pg_hba.conf
          mountPath: /etc/postgresql/pg_hba.conf # Monta pg_hba.conf
          subPath: pg_hba.conf # Especifica que es el archivo 'pg_hba.conf' del ConfigMap
        readinessProbe: # Para que Kubernetes sepa cuándo está listo para recibir tráfico
          exec:
            command: ["pg_isready", "-U", "admincar", "-d", "criptodb"]
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 5
        livenessProbe: # Para que Kubernetes sepa cuándo la app está funcionando (o muerta)
          exec:
            command: ["pg_isready", "-U", "admincar", "-d", "criptodb"]
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
      volumes: # <--- DEFINICIÓN DE LOS VOLÚMENES UTILIZADOS
      - name: postgres-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
      - name: postgres-config-volume # Definición del volumen que proviene del ConfigMap
        configMap:
          name: postgres-config # Nombre del ConfigMap que debes crear
          items:
            - key: postgresql.conf
              path: postgresql.conf
            - key: pg_hba.conf
              path: pg_hba.conf

---
# --- 4. Service para exponer PostgreSQL dentro del clúster (ClusterIP) ---
apiVersion: v1
kind: Service
metadata:
  name: postgres-service
  labels:
    app: postgres
spec:
  selector:
    app: postgres
  ports:
    - protocol: TCP
      port: 5432 # Puerto del servicio
      targetPort: 5432 # Puerto del contenedor

