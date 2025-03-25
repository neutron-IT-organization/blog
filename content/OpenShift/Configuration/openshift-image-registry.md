---
title: Comment exposer la registry interne OpenShift ?
tags:
  - Registry
---

# Exposition de la registry interne sur OpenShift

![Registry](./img/registry.png)

## Prérequis

Avant de commencer, assurez-vous de remplir les conditions suivantes :

- Être en mesure de se connecter à un cluster OpenShift.
- Avoir les privilèges `cluster-admin`.

## 1. Créer un PVC

Pour commencer, créez un PVC dédié à la registry interne sur OpenShift :

### Si vous avez OpenShift Data Foundation (conseillé) : 
Créez le fichier `image-registry-storage.yml` :
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  storageClassName: <your_odf_fs_storage_class>
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 100Gi
```
Créez la ressource OpenShift :
```sh
oc apply -f image-registry-storage.yml
```

### Si vous avez du stockage local (SNO) : 
Créez le fichier `image-registry-storage.yml` :
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: image-registry-storage
  namespace: openshift-image-registry
spec:
  storageClassName: <your_local_storage_class>
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
  volumeMode: Filesystem
```
Créez la ressource OpenShift :
```sh
oc apply -f image-registry-storage.yml
```

## 2. Modifier la configuration de l'Opérateur

Editez la configuration de l'Opérateur `image registry`:
```sh
oc edit configs.imageregistry.operator.openshift.io -n openshift-image-registry
```
Modifiez les attributs suivants : 
```yaml
spec:
  defaultRoute: true
  managementState: Managed
  storage:
    pvc:
      claim: image-registry-storage
  rolloutStrategy: Recreate # Uniquement si on utilise du stockage RWO pour éviter les problèmes de montage
```
Attendez que l'Opérateur soit à nouveau disponible :
```sh
watch oc get co image-registry
```

## 3. Pousser une image sur la registry

Récupérez la route de la registry :
```sh
HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
```

Ajoutez les certificats de la registry :
```sh
oc extract secret/$(oc get ingresscontroller -n openshift-ingress-operator default -o json | jq '.spec.defaultCertificate.name // "router-certs-default"' -r) -n openshift-ingress --confirm
```
```sh
sudo mv tls.crt /etc/pki/ca-trust/source/anchors/
```
```sh
sudo update-ca-trust enable
```

Connectez vous à la registry :
```sh
sudo podman login -u <user> -p $(oc whoami -t) $HOST
```

Taguez l'image que vous souahitez pousser : 
```sh
podman tag <your_image> $HOST/<your_project>/<your_image>
```

Créez le namespace associé à votre projet :
```sh
oc new-project <your_project>
```

Poussez votre image sur la registry :
```sh
podman push $HOST/<your_project>/<your_image>
```

## Conclusion

Vous avez maintenant exposé votre registry interne OpenShift. Vous pouvez pousser et récupérer des images via la route exposée.

---

**Auteur : [Romain GASQUET](https://www.linkedin.com/in/romain-gasquet/)**