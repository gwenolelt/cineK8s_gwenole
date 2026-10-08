# Examen CinéK8s — LE THOER Gwenole

## Partie 1 — Comprendre le code

**Q1.1** — `MovieClient` lit la propriété Spring **`movie.url`** (injectée par `@Value("${movie.url}")` dans `MovieClient.java:57`, valeur par défaut `http://localhost:8080` dans `ticket-service/src/main/resources/application.yaml:6`).
Variable d'environnement pour la surcharger sans toucher au code : **`MOVIE_URL`**. Mécanisme : *relaxed binding* Spring Boot — majuscules + `.` → `_` (`movie.url` → `MOVIE_URL`, `movie.environment` → `MOVIE_ENVIRONMENT`). C'est ce qui permet de l'injecter via ConfigMap + `envFrom` en K8s et via `environment:` en Compose.

**Q1.2** — D'après `ticket-service/src/main/java/fr/k8s101/ticket/TicketController.java` :
- (a) Film inexistant → **422 UNPROCESSABLE_ENTITY** (`TicketController.java:48` : `orElseThrow(... UNPROCESSABLE_ENTITY, "Film … inconnu")`).
- (b) Pas assez de places (`movie.seats() < request.seats()`) → **409 CONFLICT** (`TicketController.java:57`).
- (c) `movie-service` injoignable (`ResourceAccessException` : DNS, connexion refusée, timeout) → **503 SERVICE_UNAVAILABLE** (`TicketController.java:52`).
- Bonus : `seats < 1` → 400, succès → 201 CREATED.

**Q1.3** — Ligne complétée dans `ticket-service/src/main/resources/application.yaml` :
```yaml
management:
  endpoint:
    health:
      group:
        readiness:
          include: readinessState,movie
```
Le nom `movie` vient du bean `@Component("movie")` de `MovieHealthIndicator.java:11` (un `HealthIndicator` est référencé dans un groupe par son nom de bean).
Pourquoi readiness et jamais liveness : la readiness qui échoue retire juste le Pod des endpoints du Service (plus de trafic, **sans restart**, `RESTARTS` reste à 0) ; quand `movie` revient, `ticket` redevient `1/1` tout seul. Une liveness qui échoue fait **redémarrer le conteneur par le kubelet** : si `movie` tombe, tous les `ticket` entreraient en `CrashLoopBackOff` alors que leur JVM va bien — panne en cascade et redémarrages inutiles.

**Q1.4**

| Endpoint | Probe(s) Kubernetes qui l'utilisent | Conséquence d'un **échec** de la probe |
|----------|-------------------------------------|----------------------------------------|
| `/actuator/health/liveness` | `startupProbe` (gating au démarrage) + `livenessProbe` | kubelet **redémarre le conteneur** (`RESTARTS +1`) |
| `/actuator/health/readiness` | `readinessProbe` | Pod retiré des **endpoints** du Service (plus de trafic), **sans restart** ; revient tout seul quand ça repasse UP |

`server.shutdown: graceful` : à la réception de SIGTERM (rolling update, `rollout restart`, scale down), Spring finit les requêtes en cours avant de fermer le contexte au lieu de couper brutalement → évite les 500/502 pendant le roulement, en complément de la readiness qui a déjà coupé le trafic.

## Partie 2 — Tester en local, sans Kubernetes

> Exécuté le 08/10/2026. Adaptation Windows : `movie` tourne sur **8085** (cf. `movie-service/application.yaml`), `ticket` lancé avec `$env:SERVER_PORT="8082"; $env:MOVIE_URL="http://localhost:8085"` (le `movie.url` par défaut `http://localhost:8080` ne pointe pas vers 8085). `JAVA_HOME=C:\Program Files\Java\jdk-23` requis pour `./mvnw.cmd`. Builds OK (`EXIT:0`, tests `MovieControllerTest` / `TicketControllerTest` passés).

```text
=== 2.1 titres ===
(Invoke-RestMethod http://localhost:8085/api/movies).title
Pod Fiction
Le Seigneur des Pods
Docker Wars
Rollback to the Future

=== 2.1 whoami ===
{"environment":"local","hostname":"DESKTOP-DHFUL1L"}

=== 2.1 POST ticket (movieId 2, seats 3) ===
{"id":4,"movieId":2,"movieTitle":"Le Seigneur des Pods","seats":3,"total":36.00,"createdAt":"2026-10-08T10:31:28Z"}
# total 36.00 = 3 x 12.00, status HTTP 201

=== 2.1 readiness ticket ===
{"status":"UP","components":{"movie":{"status":"UP"},"readinessState":{"status":"UP"}}}

=== 2.2 après arrêt de movie (PID 22832 killé) ===
curl.exe -s http://localhost:8082/actuator/health/readiness
{"status":"DOWN","components":{"movie":{"status":"DOWN","details":{"error":"I/O error on GET request for \"http://localhost:8085/actuator/health/liveness\": null"}},"readinessState":{"status":"UP"}}}
curl.exe -s http://localhost:8082/actuator/health/liveness
{"status":"UP"}
POST /api/tickets -> 503
```

**Q2.1** — `SERVER_PORT=8082` surcharge `server.port` sans modifier `application.yaml` (qui dit `port: 8086` pour ticket, `8085` pour movie). Mécanisme : configuration externalisée Spring Boot + relaxed binding (`SERVER_PORT` → `server.port`, les env vars ont priorité sur le YAML). On garde le même jar / la même image pour tous les environnements, on ne fait que changer l'env au lancement. Ici on a aussi dû passer `MOVIE_URL=http://localhost:8085` car le défaut `http://localhost:8080` ne correspond à aucun service local.

**Q2.2** — C'est exactement voulu : la JVM `ticket` est vivante (liveness UP → pas de restart), mais elle ne peut plus remplir son contrat car sa dépendance `movie` est DOWN (readiness DOWN → hors trafic). On veut signaler « ne m'envoyez plus de requêtes » sans tuer un processus sain. En K8s Partie 6, c'est ce qui retirera les Pods `ticket` des endpoints.

## Partie 3 — Conteneuriser

> Exécuté le 08/10/2026. Note : `server.port` remis à **8080** dans les deux `application.yaml` (le dépôt disait 8085/8086, mais tout l'énoncé suppose 8080 dans le conteneur : `ports: ["8080:8080"]`, healthcheck sur `:8080`, `containerPort: 8080`, commandes `curl localhost:8080`). Sans ça, le mapping et les probes ne pouvaient pas fonctionner.

```text
=== docker images ===
movie-service    1.0.0   331MB
ticket-service   1.0.0   331MB
=== non-root ===
docker run --rm --entrypoint id movie-service:1.0.0  -> uid=10001(spring) gid=101(spring)
docker run --rm --entrypoint id ticket-service:1.0.0 -> uid=10001(spring) gid=101(spring)

=== docker compose ps ===
cinek8s_gwenole-movie-1   movie-service:1.0.0    Up 15 seconds (healthy)   0.0.0.0:8080->8080/tcp
cinek8s_gwenole-ticket-1  ticket-service:1.0.0   Up 4 seconds              0.0.0.0:8082->8080/tcp
# ticket n'a démarré qu'après movie (healthy) -> depends_on OK

=== whoami ===
{"environment":"compose","hostname":"45d8974b8984"}
# "compose" prouve que MOVIE_ENVIRONMENT surcharge application.yaml

=== POST movieId 1 seats 2 ===
{"id":1,"movieId":1,"movieTitle":"Pod Fiction","seats":2,"total":21.00}
=== readiness ticket ===
{"status":"UP","components":{"movie":{"status":"UP"},"readinessState":{"status":"UP"}}}
```
Compose coupé ensuite (`docker compose down` OK).

**Q3.1** — On copie `pom.xml` seul puis `RUN mvn dependency:go-offline` pour créer une couche Docker cachée avec toutes les dépendances. Comme Docker invalide le cache à partir de la première couche modifiée, si on ne touche qu'à une ligne de Java, seule la couche `COPY src` + compile est rejouée, les dépendances sont réutilisées → build en secondes au lieu de minutes. Si on copiait tout d'un coup, chaque modif retéléchargerait tout Maven.

**Q3.2** — `-Xmx512m` est une valeur fixe : trop grande → OOMKilled si `limits.memory` est plus bas ; trop petite → mémoire gaspillée si la limite est plus haute. `-XX:MaxRAMPercentage=75` dimensionne le heap à 75 % de la mémoire vue par le conteneur (cgroup), donc portable quel que soit `limits.memory: 512Mi` et laissant ~25 % pour metaspace, threads, native. Indispensable avec des limites K8s différentes entre dev/prod.

**Q3.3** — K8s n'a pas de `depends_on`. Si `ticket` démarre avant `movie`, sa readiness (`movie` DOWN) échoue : Pods `ticket` en `Running 0/1`, exclus des endpoints, sans trafic, `RESTARTS 0`. Dès que `movie` devient prêt, la probe repasse UP et `ticket` rejoint le Service tout seul. Pas de crash, juste une attente active — c'est pour ça qu'on met `timeoutSeconds: 3`.

## Partie 4 — Déployer sur Minikube

> Exécuté le 08/10/2026 (minikube K8s v1.37.0, images chargées via `minikube image load`, `imagePullPolicy: IfNotPresent`).

```text
=== kubectl get pods (stable) ===
NAME                      READY   STATUS    RESTARTS      AGE
movie-59684459f4-2sn74    1/1     Running   2 (2m21s ago) 5m14s
movie-59684459f4-8pvwb    1/1     Running   2 (2m21s ago) 5m14s
ticket-66d95c98b6-bzpbb   1/1     Running   0             80s
ticket-66d95c98b6-c6knj   1/1     Running   0             80s

=== kubectl get endpoints movie ticket ===
movie    10.244.0.13:8080,10.244.0.14:8080
ticket   10.244.0.17:8080,10.244.0.18:8080

=== appel inter-services depuis un Pod ticket ===
kubectl exec deploy/ticket -- wget -qO- http://movie:8080/api/movies/whoami
{"environment":"kubernetes","hostname":"movie-59684459f4-2sn74"}
kubectl exec deploy/ticket -- wget -qO- http://localhost:8080/actuator/health/readiness
{"status":"UP","components":{"movie":{"status":"UP"},"readinessState":{"status":"UP"}}}

=== réservation via port-forward (movieId 2, seats 2) ===
{"id":1,"movieId":2,"movieTitle":"Le Seigneur des Pods","seats":2,"total":24.00}
```
Note : `movie` a `RESTARTS 2` car sur cette machine (2 vCPU virtualisés) le premier démarrage Spring dépassait le budget de la startupProbe (30 × 2 s = 60 s) avec 4 JVM en parallèle. Séquençage sans toucher aux manifests : `scale deploy/ticket --replicas=0` → `movie` 1/1 → `scale deploy/ticket --replicas=2` → `ticket` 1/1 avec `RESTARTS 0`. Manifests conservés tels que l'énoncé les demande (`failureThreshold: 30`). Sur une machine normale, le premier `apply` suffit.

**Q4.1** — `kubectl apply -f k8s/` ne trie pas par kind, il traite les fichiers par ordre alphabétique : les préfixes `00-`, `10-`, `20-`… sont le seul mécanisme d'ordre. `00-namespace.yaml` d'abord (tout objet namespacé est rejeté si le namespace n'existe pas), puis `10-config.yaml` (un Deployment qui `envFrom` une ConfigMap absente reste en `CreateContainerConfigError`), puis Deployments/Services, puis Ingress. `apply` étant déclaratif et idempotent, un mauvais ordre se résorbe au 2ᵉ passage, mais les préfixes évitent ces erreurs transitoires dès le premier.

**Q4.2** — C'est la **`startupProbe`** (`GET /actuator/health/liveness`, `periodSeconds: 2`, `failureThreshold: 30`). Non, pas une anomalie : Spring Boot met 10-40 s à démarrer, le conteneur est `Running` mais pas `Ready` (`0/1`) tant que la startup n'a pas réussi ; elle suspend liveness/readiness pour éviter des restarts prématurés. Observé ici : `Startup probe failed … connection refused (x25)` puis `RESTARTS +1` quand le budget de 60 s est dépassé — au-delà, c'est le signal d'une JVM trop lente (ou d'un `failureThreshold` à relever, cf. dépannage).

**Q4.3** — Avec `imagePullPolicy: Always`, le kubelet tente toujours de tirer l'image depuis un registre distant, alors que `movie-service:1.0.0` n'existe que dans le daemon Docker de Minikube (`minikube image load/build`) et n'a jamais été pushée → `ErrImagePull` / `ImagePullBackOff`. Il faut `IfNotPresent` (utiliser l'image du nœud si présente).

## Partie 5 — Exposer avec un Ingress

> Exécuté le 08/10/2026. `k8s/40-ingress.yaml` : `networking.k8s.io/v1`, `ingressClassName: nginx`, `host: cinema.local`, `/api/movies` → `movie:http` + `/api/tickets` → `ticket:http`, `pathType: Prefix`. `describe` : les 2 règles avec backends (`movie:http 10.244.0.13:8080,10.244.0.14:8080`, `ticket:http …`). Note Windows/driver docker : `192.168.49.2:80` accepte TCP mais ne répond pas en HTTP, donc `minikube tunnel` lancé en fond et tests via `127.0.0.1` (équivalent réseau de `cinema.local` une fois le hosts renseigné).

```text
=== GET /api/movies via Ingress (titres) ===
Pod Fiction
Le Seigneur des Pods
Docker Wars
Rollback to the Future

=== POST /api/tickets via Ingress (movieId 3, seats 10) ===
{"id":1,"movieId":3,"movieTitle":"Docker Wars","seats":10,"total":90.00}

=== 6x whoami (load-balancing visible) ===
movie-59684459f4-8pvwb
movie-59684459f4-2sn74
movie-59684459f4-8pvwb
movie-59684459f4-2sn74
movie-59684459f4-8pvwb
movie-59684459f4-2sn74

=== GET /actuator/health via Ingress ===
404
```

**Q5.1** — **2** Pods `movie` distincts ont répondu (`…-8pvwb` et `…-2sn74`, alternance stricte sur 6 appels). C'est l'objet **`Service`** `movie:8080` (via ses endpoints + kube-proxy, round-robin) qui répartit la charge ; l'Ingress ne fait que router vers le Service.

**Q5.2** — Avec `pathType: Exact` sur `/api/movies`, seul ce chemin exact matcherait : `GET /api/movies/1` et `/api/movies/whoami` tomberaient en **404** (aucune règle). Il faut `Prefix` pour router tout le sous-arbre `/api/movies/*` (les services servent déjà leurs routes sous `/api`, pas de rewrite nécessaire).

**Q5.3** — **404**. Oui, souhaitable : l'Ingress n'expose que `/api/movies` et `/api/tickets` ; `/actuator/health` n'a aucune règle donc n'est pas exposé publiquement (surface d'attaque réduite, endpoints internes réservés aux probes kubelet et au debug via `kubectl exec/port-forward`).

## Partie 6 — Casser pour comprendre

### 6.1 — movie à 0 réplica

**Prédictions (à écrire avant de casser) :**
- (a) Pods `ticket` : `READY 0/1`, `STATUS Running`, `RESTARTS 0` (pas de restart).
- (b) `kubectl get endpoints ticket` : vide (`<none>`, 0 adresse).
- (c) `GET http://cinema.local/api/tickets` : **503** (pas de backend prêt ; `POST` → 503 aussi, plus jamais 422/409 car on n'atteint même pas le code).
- (d) Liveness `ticket` : reste **UP**.

**Observations (08/10/2026, conformes aux prédictions) :**
```text
kubectl scale deploy/movie --replicas=0
NAME                      READY   STATUS    RESTARTS   AGE
ticket-66d95c98b6-bzpbb   0/1     Running   0          19m
ticket-66d95c98b6-c6knj   0/1     Running   0          19m
kubectl get endpoints ticket  ->  (vide, 0 adresse)
curl -si http://cinema.local/api/tickets | head -1  ->  HTTP/1.1 503 Service Temporarily Unavailable
kubectl exec deploy/ticket -- wget -qO- localhost:8080/actuator/health/liveness  ->  {"status":"UP"}
kubectl exec deploy/ticket -- wget -qO- localhost:8080/actuator/health/readiness  ->  503 (DOWN)
kubectl describe pod -l app=ticket  ->  Warning Unhealthy 20s (x6 over 40s) Readiness probe failed: HTTP probe failed with statuscode: 503
```
Rétablissement : `kubectl scale deploy/movie --replicas=2` → après ~75 s, `movie` 1/1 (nouveaux Pods, `RESTARTS 0`), `ticket` repassé 1/1 tout seul (`RESTARTS` toujours 0), endpoints repeuplés, `GET /api/tickets` de nouveau 200 — **sans rien toucher à `ticket`**.

**Q6.1 — Explication en 4 étapes :**
1. `scale deploy/movie --replicas=0` supprime les Pods movie → `GET http://movie:8080` échoue (DNS sans endpoints / connexion refusée).
2. `MovieHealthIndicator.ping()` lève une exception → composant `movie` DOWN → groupe `readiness` (`readinessState,movie`) DOWN.
3. La `readinessProbe` (`GET /actuator/health/readiness`) échoue 3 fois → kubelet retire les Pods `ticket` des endpoints (plus de trafic).
4. Service `ticket` sans endpoint → l'Ingress n'a plus de backend → **503**. `RESTARTS` reste à 0 car seule la readiness a échoué (le kubelet ne redémarre que sur liveness) ; au `scale --replicas=2`, tout repasse UP sans toucher à `ticket`.

### 6.2 — ticket-debug (3 bugs)

| # | Statut observé | Commande de diagnostic | Cause exacte | Correction apportée |
|---|----------------|------------------------|--------------|---------------------|
| 1 | `ImagePullBackOff` (après `ErrImagePull`), Pod `ticket-debug-6bc9655bd5-dm55d` 0/1 | `kubectl describe pod -l app=ticket-debug` → Events `Pulling image "ticket-service:1.0.0"`, `Back-off pulling image`, `Error: ImagePullBackOff` | `imagePullPolicy: Always` force un pull depuis un registre distant alors que l'image n'existe que dans Minikube (`minikube image load`, jamais pushée) | `imagePullPolicy: IfNotPresent` |
| 2 | `CreateContainerConfigError`, Pod `ticket-debug-748f79d8cf-dcbpk` 0/1 | `kubectl describe pod …` → Events `Error: configmap "ticket-configmap" not found` ; `kubectl get cm` ne liste que `movie-config` et `ticket-config` | `envFrom.configMapRef.name: ticket-configmap` n'existe pas, la vraie ConfigMap s'appelle `ticket-config` (`k8s/10-config.yaml`) | `name: ticket-config` |
| 3 | `Running` mais `0/1` persistant, Pod `ticket-debug-c9999c4dc-f7q8k` | `kubectl describe pod …` → Events `Readiness probe failed: Get "http://10.244.0.23:8081/actuator/health/readiness": dial tcp …:8081: connect: connection refused (x13 over 60s)` | `readinessProbe.httpGet.port: 8081` alors que le conteneur écoute sur `containerPort: 8080` (`server.port: 8080`) | `port: 8080` |
| ✅ final | `ticket-debug-56f4f5848-k4jtk 1/1 Running 0`, puis `kubectl delete -f broken/ticket-debug.yaml` (deleted, pod Terminating) | `kubectl get pods -l app=ticket-debug` | — | — |

### 6.3

**Q6.3** — Une ConfigMap consommée via `envFrom` (variables d'environnement) n'est injectée qu'à la **création du conteneur** : le process Java a déjà ses env en mémoire, `kubectl apply` ne fait que mettre à jour l'objet etcd. Observé : après `apply` (`MOVIE_ENVIRONMENT: production`), `whoami` → encore `{"hostname":"movie-59684459f4-nzzwx","environment":"kubernetes"}`. Il faut recréer les Pods (`kubectl rollout restart deploy/movie` → `successfully rolled out`, nouveaux Pods `movie-7477664884-…`) : `whoami` → `{"environment":"production"}`.

## Partie 7 — Questions de synthèse

**Q7.1** — Le Pod `ticket` appelle `http://movie:8080` : c'est **CoreDNS** (DNS interne du cluster) qui résout `movie` en la **ClusterIP** du Service `movie` (dans le même namespace `cinema-exam` ; FQDN `movie.cinema-exam.svc.cluster.local`). La requête va à cette IP virtuelle, puis **kube-proxy** (iptables/IPVS sur chaque nœud) la DNAT vers **l'IP d'un des Pods prêts** choisis dans les endpoints (load-balancing). Le Pod `movie` élu répond directement.

**Q7.2** — Observé le 08/10/2026 : 5 POST créent les ids `2,2,3,3,4` (deux compteurs indépendants → deux listes séparées), puis `GET /api/tickets | length` donne `3,4,3,4,3,4,3,4`. Les tickets sont stockés dans une `CopyOnWriteArrayList` **en mémoire par JVM** (`TicketController.java:26`), donc chaque réplica a sa propre liste et son propre compteur ; le Service alterne entre les 2 Pods et chaque `GET` ne renvoie que la liste du Pod touché. (Détail : les 4 premiers POST avaient d'abord donné deux fois `length = 3` — les deux listes avaient alors exactement 3 éléments chacune par coïncidence, le 5ᵉ POST a fait apparaître le `4`.) Supprimer les Pods `ticket` **perd toutes les données** (état éphémère). Vraie solution architecturale : persistance externe partagée (**PostgreSQL/MySQL/Redis**), pas de sticky sessions ni de copie locale.

**Q7.3** — Observé : `pod "movie-7477664884-526hr" deleted` → 40 s après, `movie-7477664884-7vtgr 1/1 Running 0 (42s)` aux côtés de `…-jkv28` : le **Deployment** (via son ReplicaSet) a recréé un Pod pour maintenir `replicas: 2` (nouveau nom/IP, self-healing, service intact). Avec un `kind: Pod` « nu », aucun contrôleur ne le recréerait : capacité définitivement amputée (1 Pod au lieu de 2), pas de rescheduling ni de rolling update. (Note PowerShell : `kubectl delete pod $(kubectl get pod …)` échoue car `get -o name` renvoie déjà `pod/…` → utiliser `kubectl delete $(kubectl get pod -l app=movie -o name | Select-Object -First 1)`.)

## Bonus

### B1 — Durcir le Deployment `movie` (exécuté le 08/10/2026)

Ajouté dans `k8s/20-movie.yaml` : `securityContext` au conteneur + volume `emptyDir` sur `/tmp` (+ `strategy` de B2, voir ci-dessous). Rollout : `successfully rolled out`.
```text
kubectl get pods -l app=movie
movie-5b6cd4b95f-rq8ck   1/1     Running   0          26s
movie-5b6cd4b95f-t9whk   1/1     Running   0          45s
kubectl exec deploy/movie -- id  ->  uid=10001(spring) gid=101(spring) groups=101(spring)
kubectl exec deploy/movie -- touch /test  ->  touch: cannot touch '/test': Read-only file system (exit 1)
kubectl exec deploy/movie -- touch /tmp/test-ok  ->  (OK, l'emptyDir est inscriptible : Tomcat démarre, Pods 1/1)
```

### B2 — Rolling update sans coupure (exécuté le 08/10/2026)

`strategy: RollingUpdate { maxUnavailable: 0, maxSurge: 1 }` ajoutée au Deployment `movie`. Pendant `kubectl rollout restart deploy/movie` (`successfully rolled out`), boucle 300 requêtes :
```text
200    300
```
**QB2** — 100 % de `200`, 0 erreur ni 503. Trois éléments y contribuent : (1) la `strategy` (`maxUnavailable: 0` = toujours 2 Pods dispo, `maxSurge: 1` = le remplaçant monte avant de tuer l'ancien) ; (2) la `readinessProbe` (le nouveau Pod ne reçoit du trafic que prêt, et l'ancien est retiré des endpoints avant son SIGTERM) ; (3) `server.shutdown: graceful` (l'ancien Pod finit ses requêtes en cours au lieu de les couper).

---