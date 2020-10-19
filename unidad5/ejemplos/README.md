# Unidad 5: Almacenamiento en k8s

## Ejemplo 1: PV y PVC (estático)

Creamos el PersistantVolumen:

    kubectl create -f pv.yaml

    kubectl get pv
    NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
    pv1       5Gi        RWX               Recycle             Available         manual                    5s

Creamos el PersistantVolumenClaim:

    kubectl create -f pvc1.yaml

    kubectl get pv          
    NAME   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS   REASON   AGE
    pv1       5Gi         RWX               Recycle             Bound       default/pvc1  manual                    66s
    
    kubectl get pvc
    NAME   STATUS   VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    pvc1   Bound       pv1   5Gi          RWX            manual           31s

Escribimos un fichero index.html en el directorio correspondiente al volumen:

    minikube ssh                           
    $ sudo sh -c "echo 'Hello from Kubernetes storage' > /data/pv1/index.html"

Creamos un pod con el volumen

    kubectl create -f pod.yaml
    kubectl get pod
    kubectl describe pod task-pv-pod

Accedemos el pod, instalamos curl y probamos a acceder al servidor web

    kubectl exec -it task-pv-pod -- /bin/bash
    root@task-pv-pod:/# curl localhost
    Hello from Kubernetes storage


## Ejemplo 2: Gestión dinámica de volúmenes

En minikube, tenemos un provisionador de almacenamiento local del tipo hostPath. 

    kubectl get storageclass
    NAME                 PROVISIONER                 AGE
    standard (default)   k8s.io/minikube-hostpath   2d23h

Por la tanto si creamos un PersistantVolumenClaim, se creará de forma dinámica un PV:

    kubectl create -f pvc2.yaml

    kubectl get pv
    NAME                                   CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM          STORAGECLASS   
    pvc-fba7b6e9-817e-11e9-9dd3-080027b9a41d   1Gi        RWX            Delete           Bound    default/pvc2   standard            

    kubectl get pvc
    NAME   STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
    pvc2   Bound    pvc-fba7b6e9-817e-11e9-9dd3-080027b9a41d   1Gi           RWX            standard       15s


    kubectl delete pvc pvc2
    kubectl get pv


## Ejemplo 3: WordPress con almacenamiento persistente

    kubectl create -f wordpress-pv.yaml
    kubectl create -f wordpress-ns.yaml

    kubectl create -f wordpress-pvc.yaml 
    kubectl create -f mariadb-pvc.yaml 
    kubectl get pv,pvc -n wordpress    

    kubectl apply -f mariadb-srv.yaml
    kubectl apply -f wordpress-srv.yaml
    kubectl apply -f wordpress-ingress.yaml
    kubectl apply -f mariadb-secret.yaml
    kubectl apply -f mariadb-deployment.yaml
    kubectl apply -f wordpress-deployment.yaml

    kubectl get all -n wordpress

## Ejemplo 4: Gestión estática de almacenamiento en k3s

En este caso el storage class por defecto en k3s tiene el parámetro VOLUMEBINDINGMODE=WaitForFirstConsumer:

    kubectl get storageclass
    NAME                   PROVISIONER                                   RECLAIMPOLICY   VOLUMEBINDINGMODE      ALLOWVOLUMEEXPANSION   AGE
    local-path (default)   rancher.io/local-path                         Delete          WaitForFirstConsumer   false                  6d3h

Cuando creamos el PVC:

    kubectl apply -f pvc.yaml

No se crea el PV:

    kubectl get pv,pvc

Es decir el PV se creará cuando se utilice y se monte en un pod.

    kubectl apply -f pod.yaml
    kubectl get pod,pv,pvc

Para ver los detalles del volumen:

    kubectl describe persistentvolume/pvc-6ad5e59c-6b96-4e7f-b157-d48c4418c2c3

## Ejemplo 5: Gestión dinámica de almacenamiento en k3s

Vamos a trabajar con un cluster con tres workers desarrollado con k3s:

    export KUBECONFIG=~/.kube/k3s.yaml

Hemos creado un provisionador dinámicos de volúmenes nfs:

    kubectl config set-context default --namespace=nfs
    helm install my-nfs --set nfs.server=192.168.122.19 --set nfs.path=/srv/volumen helm-stable/nfs-client-provisioner
    kubectl config set-context default --namespace=default

También hemos instalado el paquete `nfs-common` en los workers para que puedan montar los volúmenes nfs.

Creamos el PVC:

    kubectl apply -f pvc.yaml

Creo el PVC:

    kubectl apply -f pvc.yaml 
    kubectl get pv,pvc

Y creo el despliegue y el servicio:

    kubectl apply -f deployment.yaml 
    kubectl apply -f service_np.yaml 

Escribo en el volumen:

    kubectl exec deployment.apps/nginx -- sh -c 'echo "Hello Word!!!" > /usr/share/nginx/html/index.html'

Y escalo el despliegue y sigo comprobando que todos los pods tienen montado el mismo volumen:

    kubectl scale deploy/nginx --replicas=4 

## Ejemplo 6: Liveness probe

    kubectl apply -f pod.yaml

Comprobamos que ha tenido reinicios, porque la prueba está fallando:

    kubectl get all                     
    NAME                READY   STATUS    RESTARTS   AGE
    pod/liveness-http   1/1     Running   2          45s

Podemos comprobar el estado del pod:

    kubectl describe pod/liveness-http
    ...

    Liveness:       http-get http://:80/healthz delay=3s timeout=3s period=3s #success=1 #failure=5
    ...
    Warning  Unhealthy  16s (x11 over 52s)  kubelet, minikube  Liveness probe failed: HTTP probe failed with statuscode: 404

Creamos un fichero, para que la prueba no falle:

    kubectl exec pod/liveness-http -- sh -c 'mkdir /usr/share/nginx/html/healthz;echo "Ok" > /usr/share/nginx/html/healthz/index.html' 

Comprobamos el funcionamiento del pod:

    kubectl port-forward pod/liveness-http 8081:80

## Ejemplo 7: Readlines probe

    kubectl apply -f depoly_service.yaml

Como no existe el fichero en ninguna replica, comprobamos que el servicio no tiene asociado ningún pod:

    kubectl describe service/nginx-service
    ...
    Endpoints:                

Creamos en el fichero en una replica, para que la prueba sea ok:

    kubectl exec nginx-deployment-fcf86974b-62622 -- sh -c "touch /tmp/iamready”

Y volvemos a comprobar el servicio:

    kubectl describe service/nginx-service
    ...
    Endpoints:   172.17.0.2:80