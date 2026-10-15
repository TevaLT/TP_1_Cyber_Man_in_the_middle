# TP de cybersécurité : phishing et usurpation DNS

Ce dépôt accompagne le TP 3.2 consacré à la préservation de l’identité numérique de l’organisation, en BTS SIO, deuxième année, à IRIS Nice (Mediaschool). Il contient une page de connexion de démonstration, un serveur Python et une configuration Ettercap pour le laboratoire Docker CERTA Kali v3.

L’objectif est de distinguer trois mécanismes : l’imitation d’une interface de connexion, l’empoisonnement ARP et la falsification d’une réponse DNS. Les observations servent ensuite à expliquer les risques et à proposer des protections adaptées.

Le scénario NiceCosmetics est pédagogique. La page reprend l’apparence de K Beauty Cosmetics ; cette référence ne constitue pas une autorisation de tester le site public. Utiliser uniquement les machines du laboratoire prévu par le formateur et des identifiants fictifs.

## Travail attendu

L’[énoncé complet](docs/TP-3.2-identite-numerique-kali.md) fournit les activités, les questions et les éléments du compte rendu. Le présent README décrit les fichiers du dépôt et leur mise en place.

À l’issue du TP, l’étudiant doit pouvoir :

- expliquer la différence entre une adresse IP, une adresse MAC et un nom de domaine ;
- relier les échanges ARP, DNS et HTTP aux observations effectuées ;
- identifier les données qu’un formulaire transmet au serveur ;
- distinguer les protections du transport, de la résolution DNS et de l’authentification ;
- présenter les preuves recueillies, leurs limites et les mesures proposées.

## Organisation du dépôt

| Fichier | Fonction |
|---|---|
| `gestion_lab1.sh` | Lance le script de gestion du laboratoire CERTA, fourni séparément. |
| `setup.sh` | Copie les supports dans `kali-lab1` et ajoute les entrées DNS Ettercap. |
| `phishing-page/index.html` | Page HTML avec formulaire envoyé en HTTP POST. |
| `harvester/harvester.py` | Serveur HTTP Python qui enregistre les champs reçus. |
| `ettercap/etter.dns.snippet` | Entrées de démonstration pour `nicecosmetics.com`. |
| `docs/TP-3.2-identite-numerique-kali.md` | Énoncé pédagogique détaillé. |
| `docs/workflow-maintenance.md` | Workflow de révision et résultats des contrôles. |
| `scripts/check-project.sh` | Contrôles locaux sans lancement du laboratoire. |

Le répertoire `_byan/` contient les ressources d’assistance BYAN. Il n’est pas nécessaire à l’exécution du TP.

## Architecture prévue

Les adresses ci-dessous sont celles de l’énoncé. Vérifier leur correspondance avec le laboratoire installé avant la séance.

| Conteneur | Adresse IP | Rôle dans le TP |
|---|---|---|
| `serveur-lab1` | `192.168.56.10` | Serveur DNS BIND9. |
| `client-lab1` | `192.168.56.11` | Poste utilisé pour observer la navigation. |
| `kali-lab1` | `192.168.56.12` | Poste d’expérimentation et serveur HTTP. |
| `routeur-lab1` | `192.168.56.254` | Routage et accès extérieur du laboratoire. |

Le réseau attendu est `192.168.56.0/24`, associé au bridge `bridge_interne_lab`. Le dépôt ne contient ni les images Docker ni leur configuration de construction.

## Préparer le laboratoire

Exécuter les commandes suivantes sur l’hôte disposant de Git, Bash et d’un accès à Docker.

```bash
git clone https://github.com/Mediaschool-IRIS-BTS-SISR-2025/Phising_Nicebeauty.git
cd Phising_Nicebeauty
git clone https://forge.apps.education.fr/reseau-certa/bts-sio/labos-kali-docker/lab1.git lab1
bash gestion_lab1.sh -c
```

Le lanceur local utilise `lab1/gestion_lab1.sh` et se place dans son répertoire avant de l’exécuter. Il ne télécharge rien automatiquement. Si une copie du laboratoire existe déjà ailleurs :

```bash
CERTA_LAB_DIR=/chemin/vers/lab1 bash gestion_lab1.sh -l
```

Les options transmises au script CERTA sont `-c` pour créer, `-l` pour lancer, `-s` pour arrêter et `-d` pour supprimer. La suppression concerne le laboratoire ; elle ne constitue pas une étape nécessaire du TP. Consulter également le README livré avec la version CERTA utilisée.

## Installer les supports

Vérifier que les quatre conteneurs sont démarrés, puis lancer :

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}'
bash setup.sh
```

Le script prépare les répertoires, copie la page dans `/var/www/html/index.html` et le serveur dans `/opt/harvester.py`. Il ajoute les lignes absentes du fichier fourni à `/etc/ettercap/etter.dns`, sans répéter les lignes identiques. Ettercap doit déjà être installé dans le conteneur.

Le script vérifie aussi Apache et tente de l’installer s’il est absent, ce qui nécessite l’accès aux dépôts de paquets. Apache peut servir la page statique ; le traitement du formulaire est assuré par le serveur Python. Les deux services ne peuvent pas écouter simultanément sur le port 80.

## Observer le fonctionnement

Dans le conteneur Kali, démarrer le serveur de démonstration :

```bash
docker exec kali-lab1 service apache2 stop
docker exec -it kali-lab1 python3 /opt/harvester.py
```

Depuis un autre terminal, vérifier la page avec une requête HTTP GET :

```bash
docker exec client-lab1 curl -s http://192.168.56.12/ | head -n 5
```

La réponse attendue commence par le document HTML. Réaliser ensuite les activités ARP et DNS décrites dans l’énoncé, sur le périmètre du laboratoire. Comparer la résolution du nom avant et pendant l’expérience :

```bash
docker exec client-lab1 nslookup nicecosmetics.com 192.168.56.10
```

La configuration fournie vise une réponse contenant `192.168.56.12` pendant la simulation. Le résultat initial dépend du DNS effectivement configuré ; une réponse `NXDOMAIN` ne prouve pas à elle seule l’absence de risque.

Ouvrir la page depuis le navigateur du client et saisir exclusivement un compte fictif, par exemple `etudiant@example.test`. Après l’envoi du formulaire, consulter :

```bash
docker exec kali-lab1 cat /opt/creds.txt
```

Le serveur enregistre en clair l’heure, l’adresse IP du client et les champs du formulaire. Il renvoie ensuite une redirection HTTP 302 vers `https://kbeauty-cosmetics.com/`. Cette navigation sort du laboratoire et dépend de l’accès extérieur ; la collecte locale précède cette redirection. Elle ne prouve aucune authentification sur le site public.

## Interpréter les résultats

L’empoisonnement ARP modifie une association IP/MAC sur le réseau local. La falsification DNS cherche à faire accepter une adresse choisie pour un nom demandé. La page de phishing exploite ensuite la confiance accordée à l’interface. Ces mécanismes doivent être identifiés séparément dans le compte rendu.

HTTPS avec validation du certificat protège la connexion au nom demandé. Une falsification DNS ne fournit pas automatiquement un certificat valable pour ce nom. Un certificat valide ne garantit toutefois pas l’honnêteté du propriétaire d’un autre domaine ressemblant au domaine attendu.

Le DNS chiffré peut changer le chemin de résolution utilisé par le navigateur. Son activation dépend du navigateur, de sa configuration et des politiques appliquées. Une requête `nslookup` et une navigation ne testent donc pas nécessairement le même résolveur. Consigner ces paramètres au lieu de supposer que tous les navigateurs utilisent DoH.

## Compte rendu

Présenter la topologie observée, les paramètres utilisés, les résultats avant et pendant l’expérience, puis leur interprétation. Les captures doivent montrer les adresses et les échanges utiles, avec des données fictives uniquement. Pour chaque protection proposée, préciser le mécanisme visé et ce qu’elle ne couvre pas.

En fin de séance, arrêter les processus lancés, restaurer les paramètres modifiés selon l’énoncé et retirer les journaux de démonstration lorsqu’ils ne sont plus nécessaires. Pour arrêter le laboratoire :

```bash
bash gestion_lab1.sh -s
```

## Vérifications et dépannage

```bash
bash scripts/check-project.sh
```

Ce contrôle vérifie les scripts et l’absence d’emojis dans les fichiers texte locaux. Il ne valide pas une attaque réseau ni la disponibilité des images CERTA.

| Symptôme | Vérification |
|---|---|
| Script CERTA absent | Contrôler `lab1/gestion_lab1.sh` ou la valeur de `CERTA_LAB_DIR`. |
| Conteneur Kali introuvable | Vérifier le nom et l’état du conteneur avec `docker ps`. |
| Port 80 occupé | Vérifier quel service écoute et arrêter Apache avant le serveur Python. |
| Page absente | Vérifier la copie de `/var/www/html/index.html`. |
| Résolution différente dans le navigateur | Comparer le résolveur utilisé, les caches et la configuration DNS chiffrée. |
| Journal absent | Vérifier qu’un formulaire a été envoyé au serveur Python et que `/opt` est accessible en écriture. |

Ne pas contourner une erreur de certificat ou de signature de paquet pour terminer l’installation. Conserver le message exact et faire vérifier la configuration du laboratoire.

## Ressources

- [Laboratoire CERTA lab1](https://forge.apps.education.fr/reseau-certa/bts-sio/labos-kali-docker/lab1)
- [Documentation Ettercap](https://www.ettercap-project.org/)
- [Bibliothèque Python `http.server`](https://docs.python.org/3/library/http.server.html)
- [DNS over HTTPS dans Firefox](https://support.mozilla.org/fr/kb/dns-over-https-doh-firefox)

Aucun fichier de licence distinct n’est fourni dans ce dépôt. Les conditions de réutilisation des ressources tierces restent celles de leurs auteurs.
#   T P _ 1 _ C y b e r _ M a n _ i n _ t h e _ m i d d l e  
 