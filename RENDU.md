# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :**
**Groupe :**

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > 3 nœuds, tous `Ready` : `k3d-lab-server-0` (rôle `control-plane`, le « master » : API Server, scheduler, controller-manager, base etcd/sqlite) et `k3d-lab-agent-0` / `k3d-lab-agent-1` (workers, rôle `<none>`, qui exécutent les pods applicatifs via kubelet + containerd). Sur un cluster « vide », des pods système tournent déjà dans le namespace `kube-system` : coredns, metrics-server, local-path-provisioner, traefik (et ses jobs helm-install) et svclb-traefik.
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre 4 conteneurs : `k3d-lab-server-0`, `k3d-lab-agent-0`, `k3d-lab-agent-1` (image `rancher/k3s`) et `k3d-lab-serverlb` (image `k3d-proxy`, le load balancer qui publie le port 8081). Dans k3d, un nœud est donc simplement un **conteneur Docker** qui exécute k3s ; les pods tournent à l'intérieur de ces conteneurs (containerd imbriqué).

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > Un **Deployment** (`web`), qui crée un **ReplicaSet** (`web-68d995574f`), qui crée lui-même un **Pod** (`web-68d995574f-xxxxx`). Le **Service** `web` (ClusterIP, port 80) vient, lui, de `kubectl expose`.
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Le pod est passé en `Terminating` et un nouveau pod avec un **nom différent** (`web-68d995574f-vlmwt` → `web-68d995574f-tmp54`) a été créé immédiatement. Le ReplicaSet constate que l'état réel (0 pod) ne correspond plus à l'état désiré (1 replica) et recrée un pod : c'est la boucle de réconciliation (auto-réparation).

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non : après relance du tunnel, la page affichait de nouveau « Welcome to nginx! ». Le système de fichiers d'un conteneur est **éphémère** : le nouveau pod est recréé à partir de l'image `nginx` d'origine, la modification faite avec `exec` n'existait que dans l'ancien conteneur.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Un `port-forward` vers un Service est en réalité attaché à **un seul pod**, choisi au lancement. Quand ce pod est supprimé, le tunnel perd sa cible : `/tmp/web.log` affiche `network namespace for sandbox ... is closed` puis `error: lost connection to pod`. Il ne bascule pas automatiquement sur le nouveau pod.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `kubectl create` est **impératif** : il crée une ressource et échoue si elle existe déjà. `kubectl apply -f` est **déclaratif** : il décrit l'état désiré dans un fichier (versionnable dans Git) ; il crée la ressource si elle n'existe pas, sinon il calcule la différence et ne met à jour que ce qui a changé (idempotent, on peut le relancer autant de fois qu'on veut).
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` (ici `app: site`) indique au Service quels pods reçoivent le trafic, grâce à leurs labels ; Kubernetes en déduit la liste des **endpoints** (IP:port des pods prêts). S'il ne correspond à aucun pod, le Service existe mais n'a **aucun endpoint** (`kubectl get endpoints site` vide) : les requêtes échouent (connexion refusée / 503 / 502 via l'Ingress).

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
NAME                    READY   STATUS    RESTARTS   AGE     IP           NODE               NOMINATED NODE   READINESS GATES
site-68bbd5f975-8z7kf   1/1     Running   0          2m23s   10.42.2.13   k3d-lab-agent-0    <none>           <none>
site-68bbd5f975-s4rtr   1/1     Running   0          2m6s    10.42.1.11   k3d-lab-server-0   <none>           <none>
site-68bbd5f975-xxmk7   1/1     Running   0          2m16s   10.42.0.12   k3d-lab-agent-1    <none>           <none>
   ```
   > Le scheduler a réparti les 3 réplicas sur 3 nœuds différents (dans k3d, le nœud server n'a pas de taint, il accepte donc aussi des pods).
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > `ErrImagePull` puis `ImagePullBackOff`. La commande `kubectl describe pod <pod>` (section *Events*) donne la cause exacte : `Failed to pull image "nginx:9.99": ... docker.io/library/nginx:9.99: not found`. Ce tag n'existe pas sur le Docker Hub.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > Oui. Le rolling update crée d'abord le nouveau pod et ne supprime les anciens qu'une fois le nouveau prêt. Le nouveau pod n'est jamais devenu prêt, donc les 3 anciens pods (`nginx:1.28`) sont restés `Running` et le Service continuait à leur envoyer le trafic. Après `kubectl rollout undo`, `kubectl diff -f k8s/site/` ne renvoie plus aucune différence : le cluster et les fichiers sont de nouveau alignés.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://potential-space-funicular-x464rq69p54hv65v-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > Oui. `kubectl get deploy mario -o yaml` montre un Deployment ordinaire, comme ceux créés en ligne de commande (mêmes champs `spec`, `strategy: RollingUpdate`, etc.). Portainer, qui tourne dans le cluster avec le ServiceAccount `portainer-sa-clusteradmin`, et `kubectl` s'adressent tous les deux à l'**API Server** du control plane. Ce sont de simples clients de la même API REST.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > **A — Ingress** : Ingress `site` vers le Service `site:80` via Traefik, accessible sur le port 8081. Après suppression des 3 pods, on a obtenu un seul `502` puis des `200` : contrairement au port-forward, l'Ingress suit automatiquement les nouveaux pods.
    > **B — Probes et ressources** : `readinessProbe` HTTP sur `/`:80 et resources (requests 50m/32Mi, limits 100m/64Mi). Avec une probe sur `/nope`, le nouveau pod reste `0/1 Running` (`Readiness probe failed: HTTP probe failed with statuscode: 404`) : il n'est jamais ajouté aux endpoints et le rolling update reste bloqué, les anciens pods continuent donc de servir le site. Retour à `/` avec `kubectl apply`.
    > **C — Drain** : `kubectl drain k3d-lab-agent-0` cordonne le nœud et évince ses pods, qui sont recréés sur `agent-1` et `server-0`. Après `uncordon`, le nœud redevient planifiable mais les pods **ne reviennent pas seuls** : Kubernetes ne rééquilibre pas les pods qui tournent déjà, seuls les nouveaux pods peuvent y être placés (un `kubectl rollout restart deploy/site` les a de nouveau répartis sur les 3 nœuds).
    > **D — Namespaces** : namespace `prod` avec une seconde copie du site (`kubectl apply -n prod -f k8s/site/`). Les deux `site` coexistent (Deployment, Service, ConfigMap du même nom) car les noms sont isolés par namespace.

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
