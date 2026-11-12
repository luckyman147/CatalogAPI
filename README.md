# TP DevOps - Product Catalog : compte rendu

## 1. Mesure et observation

`docker stats` montre une API très légère (~71 MiB, 0,01 % CPU, 20 PIDs) face à SQL Server (~876 MiB, 1,64 % CPU, 170 PIDs). La mesure est essentielle pour détecter les régressions et dimensionner ; on suivrait la latence p95, le taux d'erreur 5xx et l'usage CPU/mémoire. Un ping de port prouve seulement qu'un processus écoute, alors qu'un healthcheck applicatif garantit que l'application et la base sont opérationnelles (`depends_on: service_healthy`).

## 2. Automatisation et Lean

Le multi-stage sépare le build (SDK) de l'exécution (runtime) : aucun SDK local n'est requis, le build est reproductible et l'outillage ne part pas en production. Résultat : image plus légère, moins de disque, de réseau et de gaspillage.

## 3. Sécurité

`USER app` exécute l'application en non-root : en cas de faille, l'attaquant ne peut ni écrire dans le système, ni installer de paquets, ni ouvrir de ports privilégiés (moindre privilège).

## 4. Analyse d'images

L'image finale fait 236 Mo contre 800 à 900 Mo pour le SDK, car elle ne contient que le runtime ASP.NET (~218 Mo) et l'application publiée (~18 Mo) ; compilateur, MSBuild et NuGet restent dans l'étage de build.
