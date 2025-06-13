# NFS setup

This guide ensures proper NFS sharing between:

- NFS Server: 192.168.1.110 (hosting /mnt/sdc/mongo-NFS-server)
- Clients:
  - Docker Host (masternode.k8s.com: 192.168.1.11)
  - Kubernetes Worker Nodes (workernode1, workernode2 / 192.168.1.12, 192.168.1.13)

## Step 1 :  Configure the NFS Server (192.168.1.110)

### 1. Install NFS Server
```bash
sudo apt update && sudo apt install nfs-kernel-server -y
```
### 2. Create & Export the Shared Directory
```bash
sudo mkdir -p /mnt/sdc/mongo-NFS-server
sudo chown -R 999:999 /mnt/sdc/mongo-NFS-server
```

### 3. Configure /etc/exports
```bash
sudo nano /etc/exports
```

Add:

    /mnt/sdc/mongo-NFS-server      192.168.1.0/24(rw,sync,no_subtree_check,no_root_squash,anonuid=999,anongid=999)

- rw = Read/Write
- sync = Write changes immediately
- no_subtree_check = Better performance
- no_root_squash = Allow root access (if needed)
- anonuid=999,anongid=999 = Map accesses to MongoDB’s UID/GID

### 4. Apply Exports & Restart NFS

```bash
sudo exportfs -ra
sudo systemctl restart nfs-kernel-server
```

### 5. Open Firewall (If Enabled)
```bash
sudo ufw allow from 192.168.1.0/24 to any port nfs
sudo ufw enable
```

<img src="https://raw.githubusercontent.com/sumanb007/Labs/main/img/nfsServer.png" alt="nfsServer" width="700"/>

## Step 2 : Configure NFS Clients 

### 1. Install nfs-common on All Clients

Run on all client masternode:
```bash
sudo apt update && sudo apt install nfs-common -y
```

### 2. Test Manual Mount (Optional)
```bash
sudo mkdir -p /mnt/test-nfs
sudo mount -t nfs 192.168.1.110:/mnt/sdc/mongo-NFS-server /mnt/test-nfs
```

- verify
  ```bash
  df -h | grep nfs
  ls /mnt/test-nfs
  ```

<img src="https://raw.githubusercontent.com/sumanb007/Labs/main/img/nfsClient.png" alt="nfsClient" width="700"/>

## * Optional - Configure Docker to Use NFS
First ensures that nfs is installed on the host before we run container.

### 1. Create volume 
```bash
docker volume create \
  --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.110,rw,soft,timeo=30 \
  --opt device=:/mnt/sdb2-partition/mongo-NFS-server/docker \
  mongodb_data
```

### 2. Verify the Volume
```bash
docker volume inspect mongodb_data
```
Expected Output:
```json
{
  "Driver": "local",
  "Options": {
    "device": ":/mnt/sdb2-partition/mongo-NFS-server/docker",
    "o": "addr=192.168.1.110,rw,soft,timeo=30",
    "type": "nfs"
  },
  ...
}
```
### 3. Run a Container with the NFS Volume
```bash
docker run -d \
  --name web-mongodb \
  -v mongodb_data:/data/db \
  mongo:latest
```

### 4. on the NFS server (192.168.1.110):
```bash
ls /mnt/sdb2-partition/mongo-NFS-server/docker
```

## Alternately

### 1. Update docker-compose.yml
```yaml
volumes:
  mongodb_data:
    driver: local
    driver_opts:
      type: nfs
      o: addr=192.168.1.110,rw,soft,timeo=30
      device: ":/mnt/sdb2-partition/mongo-NFS-server/docker"  # Separate subdir
```

### 2. Restart Docker Containers
```bash
docker compose down && docker compose up -d
```

### 3. Check Docker Container
```bash
docker exec -it web-mongodb ls /data/db
```

## * Optional - Configure Kubernetes to Use NFS
First ensures that nfs is installed on all the cluster host before we run pods.

### 1. Update mongodb manifest (mongodb-pod.yaml)
```yaml
volumes:
  - name: mongo-storage
    nfs:
      server: 192.168.1.110
      path: /mnt/sdb2-partition/mongo-NFS-server/kubernetes  # Separate subdir
```

### 2. Apply Changes
```yaml
kubectl delete -f mongodb-pod.yaml --force
kubectl apply -f mongodb-pod.yaml
```

### 3. Check Pod
```bash
kubectl get pods -o wide
kubectl logs web-mongodb-<pod-id>
```

