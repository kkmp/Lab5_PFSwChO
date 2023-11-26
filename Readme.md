# Laboratorium 5 - sprawozdanie

Zadanie rozpocząłem od wydania polecenia, które pozwoliło mi utworzyć przestrzeń nazw *zad5*:
```
kubectl create ns zad5
```

# Punkt 1.
W pierwszej kolejności utworzyłem plik yaml o nazwie *zad5-quota.yaml*, zawierający zestaw ograniczeń na zasoby dla przestrzeni nazw *zad5*, zgodnie z treścią polecenia. Plik dodałem do repozytorium. Dokonałem tego przy użyciu następujacej komendy:
```
kubectl create quota zad5-quota -n zad5 --hard=cpu=2,memory=1.5Gi,pods=10 --dry-run=client -o yaml > zad5-quota.yaml
```

# Punkt 2.
Utworzyłem "szablon" pliku yaml konfiguracji poda o nazwie *worker*, opartego na obrazie *nginx* w przestrzeni nazw *zad5* przy użyciu następujacego polecenia:
```
kubectl run worker --image=nginx -n zad5 --dry-run=client -o yaml > worker-pod.yaml
```

Plik nazwałem *worker-pod.yaml*. Wymagał on modyfkacji w postaci dodania ograniczeń na wykorzystywane zasoby, zgodnie z treścią polecenia. Gotowy plik dołączyłem do repozytorium.

# Punkt 3.
Do utworzenia pliku *deployment-service.yaml* wykorzystałem podany przykład *php-apache.yaml*, który zmodyfikowałem, tak aby deployment oraz serwis znajdowały się w przestrzeni nazw *zad5*. Zmodyfikowałem również parametry *limits* i *requests*, aby były zgodne z ograniczeniami podanymi w treści polecenia. Plik dołączyłem do repozytorium.

# Punkt 4.
Aby utworzyć plik yaml, definiujący obiekt HorizontalPodAutoscaler, wykorzystałem następujące polecenie:
```
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=5 --cpu-percent=50 -n zad5 --dry-run=client -o yaml > hpa.yaml
```

Jednak aby móc utworzyć obiekt HPA przy użyciu tego polecenia, wykonałem najpierw polecenia tworzące zdefiniowany wcześniej obiekt quota, a następnie deployment z serwisem:
```
kubectl apply -f zad5-quota.yaml
kubectl apply -f deployment-service.yaml
```

Należy zaznaczyć, że podanie parametru *-n zad5* nie przynosi żądanego efektu w pliku yaml, dlatego też zmodyfikowałem ręcznie plik, wskazując pożądaną przestrzeń nazw. Jako wartość parametru *maxReplicas* przyjąłem 5 replik. Plik dołączyłem do repozytorium pod nazwą *hpa.yaml*
W celu określenia maksymalnej liczby replik, jako kluczowy parametr, wziąłem pod uwagę limit ilości pamięci RAM, zdefiniowany w obiekcie quota. Dostępny limit to 1,5Gi, które jest równe 1536Mi. Należy zwrócić uwagę na to, że pod *worker* ma ustalony własny limit pamięci, który wynosi 200Mi. W przypadku pełnego wykorzystania zasobów przez poda *worker*, maksymalna ilość pamięci, jaka będzie dostępna dla deploymentu, to 1536Mi - 200Mi = 1336Mi. Oznacza to, że w skrajnym przypadku (gdy poszczególne repliki deploymentu będą wykorzystywały swoje górne limity pamięci, równe 250Mi dla każdego poda) należy przyjąć maksymalną liczbę replik równą 5, aby nie przekroczyć parametrów quoty. 5 * 250Mi = 1250Mi, które pozostawia "bezpieczne" 86Mi wolnych, możliwych do wykorzystania zasobów. Wartość ta oczywiście mogłaby być niewystarczająca dla szóstej repliki. Należy również brać pod uwagę fakt, że po uruchomieniu poszczególnych podów mogą wystąpić pewne fluktuacje w użyciu zasobów, a rzeczywiste wykorzystanie zasobów może przekroczyć oczekiwane wartości. W przypadku zasobów procesora, górny limit to 2000m CPU. Pod *worker* może wykorzystywać maksymalnie 200m, a co za tym idzie, deployment może wykorzystywać co najwyżej 1800m CPU. Oczywiście 5 * 200m = 1000m, zatem wykorzystanie zasobów nawet w skrajnym przypadku nie powinno przekroczyć ustalonego limitu.

# Punkt 5.
Pozostałe obiekty zdefiniowane w poszczególnych plikach yaml uruchomiłem przy użyciu następującego zestawu poleceń:
```
kubectl apply -f worker-pod.yaml
kubectl apply -f hpa.yaml
```

Poprawność uruchomienia sprawdziłem przy użyciu następujących poleceń:
```
kubectl get all -n zad5

NAME                              READY   STATUS    RESTARTS   AGE
pod/php-apache-55f6b55c97-nt7q9   1/1     Running   0          5m36s
pod/worker                        1/1     Running   0          5m30s

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
service/php-apache   ClusterIP   10.105.130.98   <none>        80/TCP    5m36s

NAME                         READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/php-apache   1/1     1            1           5m36s

NAME                                    DESIRED   CURRENT   READY   AGE
replicaset.apps/php-apache-55f6b55c97   1         1         1       5m36s

NAME                                             REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
horizontalpodautoscaler.autoscaling/php-apache   Deployment/php-apache   0%/50%    1         5         1          39s
```

```
kubectl describe ns zad5

Name:         zad5
Labels:       kubernetes.io/metadata.name=zad5
Annotations:  <none>
Status:       Active

Resource Quotas
  Name:     zad5-quota
  Resource  Used   Hard
  --------  ---    ---
  cpu       250m   2
  memory    250Mi  1536Mi
  pods      2      10

No LimitRange resource.
```

```
kubectl describe deploy -n zad5 php-apache

Name:                   php-apache
Namespace:              zad5
CreationTimestamp:      Sun, 26 Nov 2023 16:40:10 +0100
Labels:                 <none>
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               run=php-apache
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  run=php-apache
  Containers:
   php-apache:
    Image:      registry.k8s.io/hpa-example
    Port:       80/TCP
    Host Port:  0/TCP
    Limits:
      cpu:     250m
      memory:  250Mi
    Requests:
      cpu:        150m
      memory:     150Mi
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   php-apache-55f6b55c97 (1/1 replicas created)
Events:
  Type    Reason             Age   From                   Message
  ----    ------             ----  ----                   -------
  Normal  ScalingReplicaSet  91s   deployment-controller  Scaled up replica set php-apache-55f6b55c97 to 1
```

```
kubectl describe pod -n zad5
```
Sekcja Events dla poda *worker*:
```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  116s  default-scheduler  Successfully assigned zad5/worker to minikube
  Normal  Pulling    115s  kubelet            Pulling image "nginx"
  Normal  Pulled     96s   kubelet            Successfully pulled image "nginx" in 6.371s (18.92s including waiting)
  Normal  Created    96s   kubelet            Created container worker
  Normal  Started    96s   kubelet            Started container worker
```

Sekcja Events dla poda deploymentu *php-apache*:
```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  2m2s  default-scheduler  Successfully assigned zad5/php-apache-55f6b55c97-nt7q9 to minikube
  Normal  Pulling    2m2s  kubelet            Pulling image "registry.k8s.io/hpa-example"
  Normal  Pulled     103s  kubelet            Successfully pulled image "registry.k8s.io/hpa-example" in 18.853s (18.853s including waiting)
  Normal  Created    103s  kubelet            Created container php-apache
  Normal  Started    102s  kubelet            Started container php-apache
```

```
kubectl describe hpa -n zad5

Name:                                                  php-apache
Namespace:                                             zad5
Labels:                                                <none>
Annotations:                                           <none>
CreationTimestamp:                                     Sun, 26 Nov 2023 16:45:07 +0100
Reference:                                             Deployment/php-apache
Metrics:                                               ( current / target )
  resource cpu on pods  (as a percentage of request):  0% (1m) / 50%
Min replicas:                                          1
Max replicas:                                          5
Deployment pods:                                       1 current / 1 desired
Conditions:
  Type            Status  Reason               Message
  ----            ------  ------               -------
  AbleToScale     True    ScaleDownStabilized  recent recommendations were higher than current one, applying the highest recent recommendation
  ScalingActive   True    ValidMetricFound     the HPA was able to successfully calculate a replica count from cpu resource utilization (percentage of request)
  ScalingLimited  False   DesiredWithinRange   the desired count is within the acceptable range
```

Wszystkie zdefiniowane obiekty zostały poprawnie utworzone i uruchomione.

# Punkt 6.
Aby móc bez przeszkód obserwować funkcjonowanie HPA, zgodnie z treścią instrukcji do laboratorium 5, aktywowałem dodatek *metrics-server* przy użyciu następującego polecenia:
```
minikube addons enable metrics-server
```

Następnie, korzystając z przykładów podanych w dokumentacji i treści instrukcji, wydałem polecenie:
```
kubectl -n zad5 run -n zad5 -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

Otrzymałem następujący błąd, wynikający z funkcjonowania obiektu *zad5-quota*, który został zgłoszony przy próbie uruchomienia poda bez zdefiniowanych limitów zasobów:
```
Error from server (Forbidden): pods "load-generator" is forbidden: failed quota: zad5-quota: must specify cpu for: load-generator; memory for: load-generator
```

Aby rozwiązać ten problem, utworzyłem obiekt typu LimitRange, który posiada zdefiniowane wartości domyślne limitów zasobów dla każdego uruchomianego kontenera. W związku z tym, że zarówno pod *worker*, jak i deployment, posiadają już zdefiniowane limity, ustawienie LimitRange nie powinno zakłócić wyników wykonywanego zadania. Warto zwrócić uwagę, że pod *load-generator* również nie może przekraczać ustawionego limitu w obiekcie quota. W skrajnym przypadku może wykorzystać do 86Mi pamięci i do 800m CPU. Dlatego też wykorzystałem przykładowy plik yaml z dokumentacji Kubernetes i ustawiłem domyślny limit w postaci 50Mi pamięci i 100m CPU. Plik pod nazwą *zad5-limitrange.yaml* dołączyłem do repozytorium sprawozdania. Poniższa komenda pozwoliła mi utworzyć zdefiniowany obiekt:
```
kubectl apply -f zad5-limitrange.yaml
```

Poprawność działania sprawdziłem przy użyciu komendy:
```
kubectl describe limitrange -n zad5

Name:       zad5-limitrange
Namespace:  zad5
Type        Resource  Min  Max  Default Request  Default Limit  Max Limit/Request Ratio
----        --------  ---  ---  ---------------  -------------  -----------------------
Container   cpu       -    -    100m             100m           -
Container   memory    -    -    50Mi             50Mi           -
```

Następnie ponownie wywołałem polecenie:
```
kubectl -n zad5 run -n zad5 -i --tty load-generator --rm --image=busybox:1.28 --restart=Never -- /bin/sh -c "while sleep 0.01; do wget -q -O- http://php-apache; done"
```

Aby sprawdzić przebieg procesu autoskalowania, w drugiej konsoli wywołałem komendę:
```
kubectl get hpa --watch -n zad5

NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
php-apache   Deployment/php-apache   0%/50%    1         5         1          10m
php-apache   Deployment/php-apache   10%/50%   1         5         1          10m
php-apache   Deployment/php-apache   166%/50%   1         5         1          11m
php-apache   Deployment/php-apache   166%/50%   1         5         4          12m
php-apache   Deployment/php-apache   61%/50%    1         5         4          12m
php-apache   Deployment/php-apache   44%/50%    1         5         4          13m
php-apache   Deployment/php-apache   45%/50%    1         5         4          14m
php-apache   Deployment/php-apache   44%/50%    1         5         4          15m
php-apache   Deployment/php-apache   45%/50%    1         5         4          16m
php-apache   Deployment/php-apache   9%/50%     1         5         4          17m
php-apache   Deployment/php-apache   0%/50%     1         5         4          18m
php-apache   Deployment/php-apache   0%/50%     1         5         4          22m
php-apache   Deployment/php-apache   0%/50%     1         5         1          22m
php-apache   Deployment/php-apache   0%/50%     1         5         1          23m
```

Zgodnie z oczekiwaniami, HPA utworzyło dodatkowe repliki, tak aby rozłożyć 166% zużycia CPU na poszczególne repliki i utrzymać zużycie na poziomie 50% przez każdą replikę. W wyniku tego utworzone zostały 4 repliki, których dokładne zużycie zasobów sprawdziłem przy użyciu polecenia:
```
kubectl top pods -n zad5
```

Przed obciążeniem:
```
NAME                          CPU(cores)   MEMORY(bytes)   
php-apache-55f6b55c97-nt7q9   1m           20Mi            
worker                        0m           3Mi   
```

W trakcie działania obciążenia:
```
NAME                          CPU(cores)   MEMORY(bytes)   
load-generator                4m           0Mi             
php-apache-55f6b55c97-bhdml   33m          15Mi            
php-apache-55f6b55c97-c9ptq   82m          11Mi            
php-apache-55f6b55c97-m29jl   67m          11Mi            
php-apache-55f6b55c97-nt7q9   84m          24Mi            
worker                        0m           3Mi    
```
Po obciążeniu:
```
NAME                          CPU(cores)   MEMORY(bytes)   
php-apache-55f6b55c97-nt7q9   1m           24Mi            
worker                        0m           3Mi   
```

Jak łatwo zaobserwować, zużycie CPU było nawet mniejsze niż 50% dla każdej repliki i nie przekraczało średnio 100m. Po zarzymaniu generowania obciążenia, HPA przeskalowało deployment ponownie do jednej repliki. Maksymalna liczba replik, którą ustawiłem była wystarczająca oraz zapewniała dodatkową "rezerwę" w przypadku wystąpienia jeszcze większego zużycia zasobów, przy jednoczesnej pewności, że limity zdefiniowane w obiekcie quota nie zostaną przekroczone. Cel zadania został osiągnięty.
