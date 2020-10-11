# Unidad 4: Despliegue de aplicaciones con k8s (II)


## Ejemplo 1: DNS

    kubectl create -f busybox.yaml
    kubectl exec -it busybox -- nslookup nginx
    kubectl exec -it busybox -- wget http://nginx

Comprobando el servidor DNS:

    kubectl get pods --namespace=kube-system -o wide
    kubectl get services --namespace=kube-system
    kubectl exec -it busybox -- cat /etc/resolv.conf

## Ejemplo 2: Balanceo de carga

Hemos creado una imagen docker que nos permite crear un contenedor con una aplicación PHP que muestra el nombre del servidor donde se ejecuta, el fichero index.php:

    <?php echo "Servidor:"; echo gethostname();echo "\n"; ?>

Si tenemos varios pod de esta aplicación, el objeto Service balancea la carga entre ellos:

    kubectl create deployment pagweb --image=josedom24/infophp:v1
    kubectl expose deploy pagweb --port=80 --type=NodePort
    kubectl scale deploy pagweb --replicas=3

Al acceder hay que indicar el puerto asignado al servicio:

    for i in `seq 1 100`; do curl http://192.168.99.100:32376; done
    Servidor:pagweb-84f6d54fb7-56zj6
    Servidor:pagweb-84f6d54fb7-mdvfn
    Servidor:pagweb-84f6d54fb7-bhz4p


## Ejemplo 3: Servicio para acceder a servidor remoto

Creamos el servicio:

    kubectl create -f service.yaml

Creamos el endpoint:

    kubectl create -f endpoint.yaml

Comprobamos que el servicio está apuntando al endpoint:

    kubectl describe service mariadb

Creamos un pod con un cliente de mariadb y accedemos usando el nombre del servicio:

    kubectl apply -f mariadb-deployment.yaml 

    kubectl exec -it pod/mariadb -- mysql -u prueba -p -h mariadb.default.svc.cluster.local

## Ejemplo 4: Despliegue canary

Creamos el servicio y el despliegue de la primera versión (5 réplicas):

    kubectl create -f service.yaml
    kubectl create -f deploy1.yaml

En un terminal, vemos los pods:

    watch kubectl get pod

En otro terminal, accedemos a la aplicación:

    service=$(minikube service my-app --url)
    while sleep 0.1; do curl "$service"; done

Desplegamos una réplica de la versión 2, para ver si funciona bien:

    kubectl create -f deploy2.yaml

Una vez que comprobamos que funciona bien, podemos escalar y eliminar la versión 1:

    kubectl scale --replicas=5 deploy my-app-v2
    kubectl delete deploy my-app-v1

## Ejemplo 5: ingress

    minikube addons enable ingress
    kubectl get pod -n kube-system
    
    kubectl create -f nginx-ingress.yaml 
    kubectl get ingress

## Ejemplo 6: LetsChat

    kubectl create -f letschat-deployment.yaml
    kubectl create -f mongo-deployment.yaml
    
    kubectl create -f letschat-srv.yaml
    kubectl create -f mongo-srv.yaml
    
    kubectl create -f ingress.yaml


## Ejemplo 7: Variables de entorno

Podemos definir un Deployment que defina un contenedor configurado por medio de variables de entorno.

Creamos el despliegue:

    kubectl create -f mariadb-deployment.yaml

O directamente ejecutando:

    kubectl run mariadb --image=mariadb --env MYSQL_ROOT_PASSWORD=my-password

Veamos el pod creado:

    kubectl get pods -l app=mariadb

Y probamos si podemos acceder, introduciendo la contraseña configurada:

    kubectl exec -it mariadb-deployment-fc75f956-f5zlt -- mysql -u root -p

## Ejemplo 8: ConfigMap

ConfigMap te permite definir un diccionario (clave,valor) para guardar información que puedes utilizar para configurar una aplicación.

    kubectl create cm mariadb --from-literal=root_password=my-password \
                              --from-literal=mysql_usuario=usuario     \
                              --from-literal=mysql_password=password-user \
                              --from-literal=basededatos=test

    kubectl get cm
    kubectl describe cm mariadb

Creamos un deployment indicando los valores guardados en el ConfigMap:

    kubectl create -f mariadb-deployment-configmap.yaml
    kubectl exec -it mariadb-deploy-cm-57f7b9c7d7-ll6pv -- mysql -u usuario -p

## Ejemplo 9: Secrets

Los Secrets nos permiten guardar información sensible que será codificada. Por ejemplo,nos permite guarda contraseñas, claves ssh, …
Al crear un Secret los valores se pueden indicar desde un directorio, un fichero o un literal.

    kubectl create secret generic mariadb --from-literal=password=root
    kubectl get secret
    kubectl describe secret mariadb

Creamos el despliegue y probamos el acceso:

    kubectl create -f mariadb-deployment-secret.yaml
    kubectl exec -it mariadb-deploy-secret-f946dddfd-kkmlb -- mysql -u root -p

## Ejemplo 10: Desplegando WordPress con MariaDB


mariadb

    kubectl create secret generic mariadb-secret \
                            --from-literal=dbuser=user_wordpress \
                            --from-literal=dbname=wordpress \
                            --from-literal=dbpassword=password1234 \
                            --from-literal=dbrootpassword=root1234 \
                            -o yaml --dry-run=client > mariadb-secret.yaml

    kubectl create -f mariadb-secret.yaml 

Creamos el servicio, que será de tipo ClusterIP:

    kubectl create -f mariadb-srv.yaml 

Y desplegamos la aplicación:
    
    kubectl create -f mariadb-deployment.yaml

wordpress

Lo primero creamos el servicio:

    kubectl create -f wordpress-srv.yaml 

Y realizamos el despliegue:

    kubectl create -f wordpress-deployment.yaml 

Por último creamos el recurso ingress que nos va a permitir el acceso a la aplicación utilizando un nombre:

    kubectl create -f wordpress-ingress.yaml



## Ejemplo 11: StatefulSet

Vamos a crear los distintos objetos de la API:

    kubectl create -f service.yaml

### Creación ordenada de pods

En un terminal observamos la creación de pods y en otro terminal creamos los pods

    watch kubectl get pod
    kubectl create -f statefulset.yaml

### Comprobamos la identidad de red estable

Vemos los `hostname` y los nombres DNS asociados:

    for i in 0 1; do kubectl exec web-$i -- sh -c 'hostname'; done
    web-0
    web-1

    kubectl run -i --tty --image busybox:1.28 dns-test --restart=Never --rm
    / # nslookup web-0.nginx
    ...
    Address 1: 172.17.0.4 web-0.nginx.default.svc.cluster.local
    / # nslookup web-1.nginx
    ...
    Address 1: 172.17.0.5 web-1.nginx.default.svc.cluster.local

### Eliminación de pods

 En un terminal observamos la creación de pods y en otro terminal eliminamos los pods

    watch kubectl get pod
    kubectl delete pod -l app=nginx

Comprobamos la identidad de red estable: Vemos los hostnames y los nombres DNS asociados (Las IP pueden cambiar):

    for i in 0 1; do kubectl exec web-$i -- sh -c 'hostname'; done

    kubectl run -i --tty --image busybox:1.28 dns-test --restart=Never --rm
    / # nslookup web-0.nginx
    / # nslookup web-1.nginx

### Escribiendo en los volúmenes persistentes

Comprobamos que se han creado volúmenes para los pods:

    kubectl get pv,pvc

Escribimos en los documentroot y accedemos al servidor:

    for i in 0 1; do kubectl exec "web-$i" -- sh -c 'echo "$(hostname)" > /usr/share/nginx/html/index.html'; done
    for i in 0 1; do kubectl exec -i -t "web-$i" -- sh -c 'curl http://localhost/'; done
    web-0
    web-1

Volvemos a eliminar los pods, y comprobamos que la información es persistente al estar guardadas en los volúmenes:

    kubectl delete pod -l app=nginx
    for i in 0 1; do kubectl exec -i -t "web-$i" -- sh -c 'curl http://localhost/'; done

### Escalar statefulset

Escalamos a más o menos pods:

    kubectl scale sts web --replicas=5

Comprobamos los pods y los volúmenes:

    kubectl get pod,pv,pvc

Si reducimos el número de pods los volúmenes no se eliminan.

Para terminar eliminamos el statefulset y el service:

    kubectl delete -f .

Para borrar los volúmenes:

    kubectl delete --all pv,pvc

## Ejemplo 12: DaemontSet

    kubectl get nodes                 
    NAME    STATUS   ROLES    AGE   VERSION
    k3s-1   Ready    <none>   17d   v1.14.1-k3s.4
    k3s-2   Ready    <none>   17d   v1.14.1-k3s.4
    k3s-3   Ready    <none>   17d   v1.14.1-k3s.4

    kubectl create -f ds.yaml

    kubectl get pods -o wide      
    NAME            READY   STATUS    RESTARTS   AGE   IP       NODE
    logging-5v4dh   1/1     Running   0          9s    10.42.2.26   k3s-2
    logging-gqfbb   1/1     Running   0          9s    10.42.1.55   k3s-3
    logging-wbdjj   1/1     Running   0          9s    10.42.0.25   k3s-1


Podemos seleccionar los nodos en los que queremos que se ejecuten los pod por medio de un selector.

    kubectl create -f ds2.yaml

    kubectl get pods -o wide      
    No resources found.

    kubectl get ds          
    NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR      AGE
    logging   0         0         0       0            0           app=logging-node   17s

    kubectl label node k3s-3 app=logging-node --overwrite

    kubectl get ds                                       
    NAME      DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE   NODE SELECTOR      AGE
    logging   1         1         0       1            0           app=logging-node   41s

    kubectl get pods -o wide                             
    NAME            READY   STATUS    RESTARTS   AGE   IP           NODE    NOMINATED NODE   READINESS GATES
    logging-556r9   1/1     Running   0          7s    10.42.1.56   k3s-3   <none>           <none>


## Ejemplo 13: Horizontal Pod AutoScaler

Veamos un ejemplo: creamos un despliegue de una aplicación php y modificamos lo que va a reservar del CPU el pod (0,2 cores de CPU):

    kubectl create deploy php-apache --image=k8s.gcr.io/hpa-example
    kubectl expose deploy php-apache --port=80 --type=NodePort
    kubectl set resources deploy php-apache --requests=cpu=200m

Creamos el recurso hpa, indicando el mínimo y máximos de pods que va a terner el despliegue, y el límite de uso de CPU que va a tener en cuenta para crear nuevos pods:

    kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=5

Vamos a hacer una prueba de estrés a nuestra aplicación y observamos cómo se comporta:

    service=$(minikube service php-apache --url)
    while true; do wget -q -O- "$service"; done

    watch kubectl get pod
    kubectl get hpa -w

## Ejemplo 14: HELM

### Instalación de helm v3

    curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/master/scripts/get-helm-3
    chmod 700 get_helm.sh
    ./get_helm.sh

### Inicializar repositorio de Chart

    helm repo add bitnami https://charts.bitnami.com/bitnami

    helm repo list
    NAME   	URL                                              
    bitnami	https://charts.bitnami.com/bitnami         

    helm repo update      

### Instalación de chart

    helm search repo 

    helm search repo nginx
    
    helm install server_web bitnami/nginx --set service.type=NodePort


Listamos las aplicaciones instaladas

    helm ls

Borramos la aplicación

    helm delete server_web

Vemos los recursos creados

    helm status server_web

Desinstalar aplicación:

    helm uninstall <name>

### Crear nuestros propios charts

    helm create mychart

* En el directorio `templates` vamos a tener nuestros manifiestos yaml.
* En estos manifiestos podemos parametrizar los valores de los distintas configuraciones. De este modo a la hora de instalar el chart podremos definir la configuracion.
* Los valores por defecto de los parámetros indicados se indican en el fichero `values.yaml`.
* En las platillas además de variables, podemos añadir control (if, loop, ...)

Podemos comprobar los maniefiestos que se han creado a partir de las plantillas:

    helm install --debug --dry-run mynginx ./mychart

Para instalar nuestro chart, con los valores por defecto:

    helm install mynginx ./mychart

Tambien podemos indicar algunos de los parámetros:

    helm install mynginx ./mychart --set replicaCount=3 --set service.type=NodePort --set ingress.hosts[0].host=mynginx.192.168.99.100.nip.io
