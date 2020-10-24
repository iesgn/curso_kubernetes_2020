# Unidad 6: Administración de kubernetes

## Ejemplo 1: Espacios de nombres

Creamos un espacio de nombre con el usuario admin:

    kubectl create ns ejemplo
	
## Ejemplo2: Creación de usuario

Creamos el usuario “usuario1” del grupo “desarrollo” y el espacio de
nombres proyecto1:

Creamos el fichero CSR en formato JSON que vamos a pasar a
`cfssl`. Este es el paso que el usuario debería hacer en su propio
equipo y el fichero resultante enviarlo al responsable de firmar el
certificado:

```
cat > usuario1-csr.json <<EOF
{
  "CN": "usuario1",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "ES",
      "ST": "Sevilla",
      "L": "Sevilla",
      "O": "desarrollo",
      "OU": "k8s-cluster"
    }
  ]
}
EOF
```

Firmamos el certificado con `cfssl`:

```
cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  usuario1-csr.json | cfssljson -bare usuario1
```

Que genera el certificado x509 en el fichero `usuario1.pem` y la
correspondiente clave en `usuario1-key.pem`.

Creamos el espacio de nombres `proyecto1`:

```
kubectl create ns proyecto1
namespace/proyecto1 created
```

Creamos el usuario1 en el equipo de pruebas y ubicamos allí el
certificado x509, la clave correspondiente y el certificado de la
autoridad certificadora.

# Configuración del acceso del usuario1

Configuramos el cliente de kubernetes para que utilice las
credenciales anteriores en el cluster "k8s-cluster", que creará el
fichero `~/.kube/config`. Inicialmente el usuario no tiene configurada
el acceso al clúster:

```
usuario1@curso:~$ kubectl config view
apiVersion: v1
clusters: null
contexts: null
current-context: ""
kind: Config
preferences: {}
users: null
```

Agregamos las credenciales a la configuración:

```
kubectl config set-credentials usuario1 \
--client-certificate=usuario1.pem \
--client-key=usuario1-key.pem
```

Ya cambia la configuración:

```
usuario1@curso:~$ kubectl config view
apiVersion: v1
clusters: null
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: usuario1
  user:
    client-certificate: /home/usuario1/usuario1.pem
    client-key: /home/usuario1/usuario1-key.pem
```

Definimos la dirección IP del kube-apiserver en la variable de entorno
KUBE-API-ADDRESS y definimos los parámetros del clúster:

```
usuario1@curso:~$ export KUBE-API-ADDRES=...
usuario1@curso:~$ kubectl config set-cluster k8s-cluster \
--certificate-authority=ca.pem \
--embed-certs=true \
--server=https://$(KUBE-API-ADDRESS):6443
Cluster "k8s-cluster" set.
```

Y comprobamos el cambio en la configuración del cliente de kubernetes:

```
usuario1@curso:~$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.22.200.1:6443
  name: k8s-cluster
contexts: null
current-context: ""
kind: Config
preferences: {}
users:
- name: usuario1
  user:
    client-certificate: /home/usuario1/usuario1.pem
    client-key: /home/usuario1/usuario1-key.pem
```

Definimos un contexto para el usuario1 en este clúster y el espacio de
nombres proyecto1:

```
usuario1@curso:~$ kubectl config set-context usuario1-context \
--cluster=k8s-cluster \
--namespace=proyecto1 \
--user=usuario1
Context "usuario1-context" created.
```

Y una vez más comprobamos la configuración:

```
usuario1@curso:~$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: DATA+OMITTED
    server: https://172.22.200.1:6443
  name: k8s-cluster
contexts:
- context:
    cluster: k8s-cluster
    namespace: proyecto1
    user: usuario1
  name: usuario1-context
current-context: ""
kind: Config
preferences: {}
users:
- name: usuario1
  user:
    client-certificate: /home/usuario1/usuario1.pem
    client-key: /home/usuario1/usuario1-key.pem
```

Podemos verificar los clusters y contextos que se han creado:

```
usuario1@curso:~$ kubectl config get-clusters
NAME
k8s-cluster
usuario1@curso:~$ kubectl config get-contexts 
CURRENT   NAME               CLUSTER       AUTHINFO   NAMESPACE
          usuario1-context   k8s-cluster   usuario1   proyecto1
```

Aunque hay un solo contexto, no está definido como el contexto actual,
por lo que no utiliza la configuración correspondiente. Para que
utilice un determinado contexto:

```
usuario1@curso:~$ kubectl config use-context usuario1-context
Switched to context "usuario1-context".
usuario1@curso:~$ kubectl config get-contexts 
CURRENT   NAME               CLUSTER       AUTHINFO   NAMESPACE
*         usuario1-context   k8s-cluster   usuario1   proyecto1
```

Y ahora comprobamos que puede acceder al cluster y ver los pods:

```
usuario1@curso:~$ kubectl get pods
Error from server (Forbidden): pods is forbidden: User "usuario1" cannot list resource "pods" in API group "" in the namespace "proyecto1"
```

¡No puede porque no tiene permiso para hacerlo!
