## Reference
```http request
https://medium.com/@suyashmohan/setting-up-postgresql-database-on-kubernetes-24a2a192e962
```
```http request
https://github.com/suyashmohan/kubernetes_postgres_statefulset
```
```http request
https://learnbatta.com/blog/kubernetes-django-app-deployment-107/
```
```http request
https://github.com/AnjaneyuluBatta505/kubernetes_django
```
Test connect
```bash
kubectl run psql -it --rm=true --image=postgres:12 --command -- psql -h 10.1.147.183 -U mydbuser mydb
```