<h1 align="center"> Programmateur horaire ESP32  Version 2.0  </h1> 

<h2 align="center">N relais, interface web, écran OLED, OTA et Programmation à la minute prêt</h2>

![Platform](https://img.shields.io/badge/Platform-ESP32-green)
![Framework](https://img.shields.io/badge/Framework-Arduino-blue)
![Status](https://img.shields.io/badge/Status-Active-green)
![Release](https://img.shields.io/badge/Release-v2.0.OTA-orange)


Ce projet transforme un **ESP32** en programmateur horaire connecté capable de piloter plusieurs relais indépendants.

Le programme utilisé est :

`Programmateur_horaire_ESP32_Multi_grilles_horaire_au_choix_V2_sans_auth_plages.ino`

Fonctions principales :

- pilotage d'un nombre configurable de relais ;
- mode **AUTO** avec programmation horaire ;
- mode **MANUEL** avec forçage ON/OFF ;
- jusqu'à `MAX_PLAGES` plages horaires par relais ;
- horaires réglables **à la minute près** ;
- plages pouvant traverser minuit ;
- effacement complet des plages possible ;
- sauvegarde des réglages dans la mémoire flash **NVS** ;
- interface Web embarquée directement dans l'ESP32 ;
- interface adaptée aux smartphones ;
- accès par adresse IP et par **mDNS** (`richardv.local`) ;
- affichage sur OLED SSD1306 128×64 ;
- boutons physiques optionnels avec anti-rebond ;
- connexion automatique au meilleur réseau Wi-Fi connu ;
- reconnexion Wi-Fi non bloquante ;
- recherche périodique d'un réseau connu offrant un meilleur signal ;
- mise à jour du firmware par **OTA** ;
- affichage de la puissance Wi-Fi en dBm et en pourcentage ;
- validation des horaires avant sauvegarde ;
- conservation des réglages lors d'une simple recompilation ou mise à jour OTA.

---

# 2. Architecture générale

Le fonctionnement peut être résumé ainsi :

```text
                         ┌─────────────────────┐
                         │       ESP32         │
                         │                     │
                         │  Programme horaire  │
                         │        + NVS        │
                         └──────────┬──────────┘
                                    │
             ┌──────────────────────┼──────────────────────┐
             │                      │                      │
             ▼                      ▼                      ▼
       Interface Web             OLED SSD1306          Boutons physiques
       Smartphone/PC              128 x 64             optionnels
             │                      │                      │
             └──────────────────────┼──────────────────────┘
                                    │
                                    ▼
                              GPIO des relais
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
                  Relais 1        Relais 2       Relais N
```

Le navigateur ne contient pas la programmation en dur : il demande la configuration à l'ESP32 avec `/get-config`, puis construit dynamiquement l'interface.

---

# 3. Configuration actuelle du programme

La configuration livrée avec le fichier est :

| Programmateur | Fonction | GPIO relais | GPIO bouton | Mode par défaut | Plages par défaut |
|---|---|---:|---:|---|---|
| 1 | Cuisine | GPIO32 | GPIO14 | AUTO | 06:32-08:10, 09:00-13:15 |
| 2 | Portail | GPIO33 | GPIO16 | AUTO | 07:05-09:10, 17:08-19:33 |
| 3 | Relais 3 | GPIO25 | GPIO17 | AUTO | aucune |
| 4 | Relais 4 | GPIO26 | GPIO18 | AUTO | 23:17-01:52 |

Les quatre relais sont actuellement configurés :

```cpp
relayActiveHigh = true
```

Cela signifie :

```text
GPIO HIGH → relais activé
GPIO LOW  → relais désactivé
```

Si votre carte relais fonctionne en logique inverse, mettre :

```cpp
false
```

pour le relais concerné.

---

# 4. Matériel nécessaire

## 4.1 Éléments principaux

- ESP32 compatible Arduino ;
- module(s) relais compatibles avec les niveaux logiques de l'ESP32 ;
- alimentation adaptée à l'ESP32 et aux relais ;
- écran OLED SSD1306 I2C 128×64, adresse `0x3C` ;
- boutons poussoirs optionnels ;
- câblage adapté.

## 4.2 Attention au niveau logique

Les GPIO de l'ESP32 fonctionnent en logique **3,3 V**.

Ne pas appliquer directement du 5 V sur une GPIO.

Selon le module relais utilisé, vérifier :

- tension de commande ;
- courant demandé par l'entrée ;
- compatibilité avec un signal 3,3 V ;
- logique active HIGH ou active LOW.

---

# 5. Câblage OLED

L'écran OLED SSD1306 utilisé par le programme est :

```text
Résolution : 128 × 64
Interface  : I2C
Adresse    : 0x3C
```

Câblage prévu :

| OLED | ESP32 |
|---|---|
| VCC | alimentation adaptée au module |
| GND | GND |
| SDA | GPIO21 |
| SCL | GPIO22 |

Dans le programme :

```cpp
#define SCREEN_WIDTH   128
#define SCREEN_HEIGHT  64
#define OLED_RESET     -1
#define SCREEN_ADDRESS 0x3C
```

Le bus I2C est initialisé par :

```cpp
Wire.begin();
```

---

# 6. Câblage des relais

Configuration actuelle :

| Relais | GPIO ESP32 | Logique |
|---|---:|---|
| Relais 1 | GPIO32 | actif HIGH |
| Relais 2 | GPIO33 | actif HIGH |
| Relais 3 | GPIO25 | actif HIGH |
| Relais 4 | GPIO26 | actif HIGH |

La fonction centrale utilisée pour commander un relais est :

```cpp
ecrireRelais(const Programmateur &p, bool etat)
```

Elle tient compte individuellement de `relayActiveHigh`.

Cette centralisation évite d'avoir des `digitalWrite()` incompatibles dispersés dans le programme.

---

# 7. Boutons physiques

Les boutons sont optionnels.

Configuration actuelle :

| Bouton | Programmateur | GPIO |
|---|---|---:|
| BP1 | Relais 1 | GPIO14 |
| BP2 | Relais 2 | GPIO16 |
| BP3 | Relais 3 | GPIO17 |
| BP4 | Relais 4 | GPIO18 |

Câblage :

```text
GPIO bouton ───── bouton poussoir ───── GND
```

Le programme utilise :

```cpp
pinMode(pinBP, INPUT_PULLUP);
```

Il n'est donc normalement pas nécessaire d'ajouter une résistance externe de pull-up.

### Fonctionnement

Un appui bref :

1. inverse l'état du relais ;
2. passe automatiquement le relais en mode MANUEL ;
3. sauvegarde le nouvel état dans la NVS.

Un relais forcé en manuel n'est plus piloté par ses horaires tant qu'il reste en mode MANUEL.

---

# 8. Ajouter ou supprimer un relais

La configuration principale se trouve dans :

```cpp
Programmateur programmateurs[] = {
```

Exemple :

```cpp
{ "5", "Programmation 5", "Relais 5", "#a78bfa",
  27, true, "12:00-22:00", true, false, -1 },
```

Les champs sont :

```text
id
nom
sousNom
couleur
GPIO relais
relayActiveHigh
plagesDefaut
modeAuto
relayState
GPIO bouton
```

### Exemple sans bouton physique

```cpp
{ "5", "Programmation 5", "Relais 5", "#a78bfa",
  27, true, "12:00-22:00", true, false, -1 },
```

`-1` signifie :

```text
aucun bouton physique
```

La page Web, les routes HTTP, la NVS et l'affichage OLED utilisent automatiquement le nombre de lignes présentes dans `programmateurs[]`.

---

# 9. GPIO à utiliser avec prudence

Le programme fournit notamment les GPIO suivants comme sorties possibles sur un ESP32 DevKit classique :

```text
4, 5, 13, 14, 16, 17, 18, 19,
21, 22, 23, 25, 26, 27, 32, 33
```

Mais :

- GPIO21 = SDA I2C ;
- GPIO22 = SCL I2C ;
- GPIO34 à GPIO39 = entrées uniquement ;
- les GPIO de boot doivent être utilisés avec prudence ;
- les GPIO déjà utilisés par d'autres périphériques ne doivent pas être réutilisés.

Toujours vérifier le brochage exact de votre modèle d'ESP32.

---

# 10. Nombre maximal de plages horaires

Le nombre maximal est défini ici :

```cpp
const int MAX_PLAGES = 6;
```

Actuellement :

```text
6 plages maximum par relais
```

Pour passer à 8 :

```cpp
const int MAX_PLAGES = 8;
```

La structure, la NVS et l'interface Web sont conçues pour suivre automatiquement cette valeur.

---

# 11. Format des plages horaires

Les horaires sont exprimés à la minute près.

Exemples valides :

```text
06:32-08:10
09:00-13:15
18:45-22:30
```

Plusieurs plages sont séparées par une virgule :

```text
06:32-08:10,09:00-13:15,18:45-22:30
```

---

# 12. Plage traversant minuit

Le programme accepte directement :

```text
23:17-01:52
```

Cela signifie :

```text
23:17 → 24:00
+
00:00 → 01:52
```

Il n'est pas nécessaire de créer deux plages.

La fonction `plageEstActive()` traite automatiquement ce cas.

---

# 13. Utilisation de 24:00

Le moteur interne autorise :

```text
24:00
```

comme heure de fin.

Exemple :

```text
06:00-24:00
```

signifie :

```text
06:00 → minuit
```

`24:00` est stocké comme :

```text
1440 minutes
```

Le champ HTML `input type="time"` ne gérant pas correctement `24:00` dans tous les navigateurs, l'interface Web utilise une gestion spécifique pour cette valeur.

---

# 14. Effacer toutes les plages

Une programmation vide est autorisée :

```text
plages=""
```

Cela signifie :

```text
aucune plage horaire
```

En mode AUTO, le relais reste donc éteint, sauf autre logique spécifique du programme.

Important : une liste vide sauvegardée dans la NVS est distinguée d'une absence totale de configuration.

---

# 15. Validation des horaires

Avant d'être enregistrée, une nouvelle programmation est vérifiée par :

```cpp
textePlagesValide()
```

Le programme vérifie notamment :

- présence du séparateur `-` ;
- validité des heures ;
- validité des minutes ;
- respect du nombre maximum de plages ;
- impossibilité d'avoir une plage de durée nulle.

Exemple refusé :

```text
08:00-08:00
```

Exemple accepté :

```text
08:00-10:00
```

Si une programmation est invalide, l'ancienne programmation reste conservée.

---

# 16. Interface Web

L'interface HTML/CSS/JavaScript est intégrée directement dans le fichier `.ino`.

Il n'est pas nécessaire d'utiliser :

- carte SD ;
- LittleFS ;
- SPIFFS ;
- fichier HTML externe.

La page est placée dans :

```cpp
const char index_html[] PROGMEM = R"rawliteral(
...
)rawliteral";
```

L'interface s'adapte automatiquement au nombre de relais.

---

# 17. Accès à l'interface

Après connexion Wi-Fi, l'ESP32 est normalement accessible par :

```text
http://richardv.local
```

et également par son adresse IP.

Exemple :

```text
http://192.168.1.50
```

Le nom mDNS est défini par :

```cpp
const char* hostname = "richardv";
```

Si vous changez cette ligne :

```cpp
const char* hostname = "monesp32";
```

l'adresse devient :

```text
http://monesp32.local
```

---

# 18. Compatibilité smartphone

La page Web utilise notamment :

```html
<meta name="viewport"
      content="width=device-width, initial-scale=1, viewport-fit=cover">
```

L'interface est conçue pour une largeur d'environ 430 px et s'adapte aux écrans de smartphones.

Elle peut être utilisée depuis :

- Android ;
- iPhone/iPad ;
- Windows ;
- Linux ;
- macOS ;

à condition que l'appareil soit sur le réseau local approprié.

---

# 19. Authentification Web

### Modification importante de cette version

La modification des **plages horaires est volontairement SANS authentification**.

La route :

```text
/save
```

est donc librement accessible sur le réseau local.

Cela permet de modifier les horaires directement depuis le smartphone sans demander de mot de passe.

### Les autres commandes sensibles restent protégées

Les routes suivantes utilisent une authentification HTTP Basic :

```text
/toggle-mode
/force-state
/reset-auto
```

L'identifiant est :

```text
admin
```

Le mot de passe utilisé est :

```cpp
SECRET_OTA_PASSWORD
```

qui doit être défini dans `arduino_secrets.h`.

---

# 20. Attention à la sécurité

Le serveur Web fonctionne en :

```text
HTTP
```

et non en HTTPS.

Le mot de passe HTTP Basic ne doit donc pas être considéré comme une protection cryptographique forte.

Le projet est destiné en priorité à un **réseau local de confiance**.

Éviter d'exposer directement le port 80 de l'ESP32 sur Internet.

---

# 21. Fichier arduino_secrets.h

Le fichier :

```text
arduino_secrets.h
```

n'est normalement pas inclus dans le dépôt ou dans le partage public du programme.

Il doit contenir les informations Wi-Fi et le mot de passe OTA.

Exemple de structure :

```cpp
#define SECRET_SSID  "Nom_du_WiFi_1"
#define SECRET_PASS  "Mot_de_passe_WiFi_1"

#define SECRET_SSID2 "Nom_du_WiFi_2"
#define SECRET_PASS2 "Mot_de_passe_WiFi_2"

#define SECRET_OTA_PASSWORD "Mot_de_passe_OTA"
```

Si un troisième réseau est utilisé :

```cpp
#define SECRET_SSID3 "Nom_du_WiFi_3"
#define SECRET_PASS3 "Mot_de_passe_WiFi_3"
```

Le troisième réseau est pris en compte par :

```cpp
#ifdef SECRET_SSID3
```

---

# 22. Connexion Wi-Fi

Le programme peut connaître plusieurs réseaux Wi-Fi.

Il commence par scanner les réseaux visibles.

Il ne conserve que ceux qui sont présents dans :

```cpp
knownNetworks[]
```

Il sélectionne ensuite le réseau connu offrant le meilleur RSSI.

Le programme utilise une gestion non bloquante pour les recherches ultérieures.

---

# 23. Reconnexion Wi-Fi

Si le Wi-Fi est perdu :

1. la boucle principale continue ;
2. l'ESP32 ne reste pas bloqué dans une longue attente ;
3. une nouvelle recherche est lancée ;
4. les réseaux connus sont évalués ;
5. une reconnexion est tentée.

La tentative est espacée dans le temps afin d'éviter une boucle de reconnexion permanente.

---

# 24. Recherche d'un meilleur réseau

Même lorsque l'ESP32 est déjà connecté, le programme vérifie périodiquement si un autre réseau connu offre un signal sensiblement meilleur.

Une marge RSSI est utilisée pour éviter des changements incessants entre deux réseaux proches.

La constante utilisée est :

```cpp
const int RSSI_SWITCH_MARGIN = 8;
```

Cela correspond à une marge d'environ 8 dBm.

---

# 25. Affichage du signal Wi-Fi

La page Web affiche :

- RSSI en dBm ;
- estimation en pourcentage.

Exemple :

```text
-52 dBm
```

Le pourcentage est une estimation destinée à rendre la lecture plus intuitive.

Le dBm reste la valeur technique la plus utile pour comparer deux signaux.

---

# 26. NTP et heure française

Le programme utilise NTP pour synchroniser l'horloge interne :

```cpp
configTzTime(
  TZ_INFO,
  "pool.ntp.org",
  "time.google.com"
);
```

Le fuseau utilisé est :

```cpp
const char *TZ_INFO =
  "CET-1CEST,M3.5.0,M10.5.0/3";
```

Il correspond à la France avec passage automatique :

- heure d'hiver ;
- heure d'été.

---

# 27. Démarrage sans synchronisation NTP

Le programme tente de synchroniser l'heure pendant un temps limité.

Il effectue au maximum :

```text
20 tentatives
```

avec :

```text
500 ms
```

entre les tentatives.

Soit environ :

```text
10 secondes
```

Si la synchronisation échoue :

```text
NTP Timeout
```

le démarrage continue.

---

# 28. Application immédiate de la programmation

Après la synchronisation NTP, le programme appelle :

```cpp
appliquerProgrammation();
```

Cela évite d'attendre le premier cycle complet de `loop()` pour corriger l'état physique des relais.

C'est particulièrement utile après :

- redémarrage ;
- coupure secteur ;
- mise à jour OTA.

---

# 29. Mode AUTO

En mode AUTO :

```text
les horaires déterminent l'état du relais
```

Toutes les secondes, le programme recalcule l'état voulu.

Si une plage est active :

```text
Relais ON
```

Sinon :

```text
Relais OFF
```

---

# 30. Mode MANUEL

En mode MANUEL :

```text
les horaires sont ignorés
```

L'état du relais est commandé par :

- bouton Web `FORCER ON/OFF` ;
- bouton physique correspondant.

Un relais qui reste en MANUEL après un redémarrage ou une OTA reste volontairement hors du contrôle horaire.

---

# 31. Retour en mode AUTO

La route :

```text
/reset-auto
```

permet de remettre tous les relais en automatique.

Cette commande :

- ne supprime pas les horaires ;
- ne réinitialise pas la NVS ;
- remet tous les relais en AUTO ;
- applique immédiatement la programmation actuelle.

Cette route est protégée par authentification.

---

# 32. Sauvegarde NVS

Les réglages sont sauvegardés dans la mémoire NVS de l'ESP32 via :

```cpp
#include <Preferences.h>
```

Sont notamment conservés :

- plages horaires ;
- mode AUTO/MANUEL ;
- état mémorisé du relais.

Les réglages survivent à :

- redémarrage ;
- coupure d'alimentation ;
- mise à jour OTA.

---

# 33. Correction importante de la NVS

Dans une ancienne logique, la signature :

```cpp
__DATE__ " " __TIME__
```

pouvait être utilisée pour déterminer si la configuration devait être effacée.

Cela était problématique car cette signature change à chaque compilation.

Dans cette version :

```cpp
const char* FIRMWARE_BUILD = __DATE__ " " __TIME__;
```

sert uniquement au **diagnostic du firmware**.

La décision de réinitialiser la NVS dépend désormais de :

```cpp
const uint16_t CONFIG_VERSION = 2;
```

---

# 34. CONFIG_VERSION

Cette valeur ne doit être modifiée que si la structure des données stockées en NVS devient réellement incompatible.

Exemple :

```cpp
const uint16_t CONFIG_VERSION = 2;
```

Si une future version nécessite une nouvelle structure :

```cpp
const uint16_t CONFIG_VERSION = 3;
```

Lors du prochain démarrage, l'ancienne configuration NVS sera réinitialisée.

### Important

Ne pas augmenter cette valeur simplement parce que vous avez :

- changé le HTML ;
- changé le CSS ;
- corrigé une fonction ;
- changé un commentaire ;
- effectué une mise à jour OTA.

Sinon les réglages utilisateur seront perdus.

---

# 35. Diagnostic des mises à jour OTA

La signature :

```cpp
FIRMWARE_BUILD
```

est visible :

- dans le moniteur série ;
- sur l'OLED ;
- dans `/get-info` ;
- dans la fenêtre « Infos système ».

Cela permet de vérifier qu'un nouveau firmware est réellement exécuté.

La NVS contient également la signature du firmware précédent.

---

# 36. Mise à jour OTA

Le programme utilise :

```cpp
#include <ArduinoOTA.h>
```

Après un premier téléversement USB avec OTA actif, l'ESP32 peut être programmé par Wi-Fi.

Dans Arduino IDE :

```text
Outils
  → Port
    → Port réseau
```

puis sélectionner l'ESP32.

Le mot de passe OTA est :

```cpp
SECRET_OTA_PASSWORD
```

---

# 37. Différence entre OTA et programmation Web

Il est important de distinguer :

### OTA

Modifie :

```text
le firmware de l'ESP32
```

### Interface Web

Modifie :

```text
la configuration utilisateur
```

Une mise à jour OTA ne doit pas supprimer les horaires.

Les horaires restent dans la NVS.

---

# 38. Routes HTTP disponibles

## Page principale

```text
GET /
```

Retourne la page Web.

---

## Configuration des relais

```text
GET /get-config
```

Retourne notamment :

- nombre de relais ;
- identifiant ;
- nom ;
- sous-nom ;
- couleur ;
- GPIO ;
- nombre maximal de plages.

---

## État courant

```text
GET /get-data
```

Retourne notamment :

- plages ;
- résumé ;
- mode AUTO/MANUEL ;
- état du relais ;
- temps avant prochain changement ;
- heure courante ;
- qualité Wi-Fi.

---

## Informations système

```text
GET /get-info
```

Retourne notamment :

- état Wi-Fi ;
- SSID ;
- hostname ;
- adresse IP ;
- adresse MAC ;
- RSSI ;
- RSSI en pourcentage ;
- signature du firmware ;
- état de réinitialisation NVS ;
- ancienne signature firmware.

---

## Changement AUTO/MANUEL

```text
GET /toggle-mode?id=1
```

Cette route est protégée par authentification.

---

## Forçage ON/OFF

```text
GET /force-state?id=1
```

Cette route :

- passe le relais en MANUEL ;
- inverse son état ;
- applique immédiatement l'état physique ;
- sauvegarde dans la NVS.

Elle est protégée par authentification.

---

## Modification des plages

```text
POST /save?id=1
```

avec :

```text
plages=06:30-08:00,18:45-22:30
```

Cette route est **volontairement sans authentification** dans cette version.

Exemple avec `curl` :

```bash
curl -X POST "http://richardv.local/save?id=1" \
     -d "plages=06:30-08:00,18:45-22:30"
```

Pour supprimer toutes les plages :

```bash
curl -X POST "http://richardv.local/save?id=1" \
     -d "plages="
```

---

## Retour de tous les relais en AUTO

```text
GET /reset-auto
```

Cette route est protégée par authentification.

---

# 39. Anti-rebond des boutons

Le temps anti-rebond est :

```cpp
const unsigned long BP_DEBOUNCE_MS = 40;
```

Le programme distingue :

- dernière lecture brute ;
- état stabilisé ;
- instant de dernière transition.

Cela évite plusieurs déclenchements pour un seul appui mécanique.

---

# 40. Boucle principale

La fonction :

```cpp
void loop()
```

effectue notamment :

```text
1. lecture des boutons physiques
2. traitement OTA
3. surveillance Wi-Fi
4. reconnexion si nécessaire
5. recherche périodique d'un meilleur réseau
6. application de la programmation horaire
7. changement éventuel de page OLED
8. mise à jour OLED
```

La programmation horaire et la surveillance Wi-Fi sont traitées environ toutes les secondes.

Les boutons et l'OTA sont traités à chaque passage de `loop()` pour conserver une bonne réactivité.

---

# 41. OLED

L'écran est organisé pour afficher notamment :

- heure ;
- réseau Wi-Fi ;
- adresse IP ;
- état des relais ;
- mode AUTO/MANUEL ;
- informations de programmation.

La constante :

```cpp
const int OLED_LIGNES_PAR_PAGE = 5;
```

définit le nombre de relais affichés par page.

Si le nombre de relais dépasse cette valeur, les pages tournent automatiquement.

Le changement de page intervient toutes les :

```text
8 secondes
```

---

# 42. Bibliothèques nécessaires

Le programme utilise les bibliothèques suivantes :

```cpp
WiFi.h
ESPAsyncWebServer.h
Preferences.h
ArduinoJson.h
time.h
ESPmDNS.h
ArduinoOTA.h
Wire.h
Adafruit_GFX.h
Adafruit_SSD1306.h
```

Selon la version du core ESP32 utilisée, `WiFi.h`, `Preferences.h`, `time.h`, `ESPmDNS.h`, `ArduinoOTA.h` et `Wire.h` sont fournies avec le support ESP32.

Les bibliothèques à installer si elles ne sont pas déjà présentes comprennent notamment :

- **ESPAsyncWebServer**
- **ArduinoJson**
- **Adafruit GFX Library**
- **Adafruit SSD1306**

---

# 43. Installation dans Arduino IDE

## Étape 1 – Installer le support ESP32

Dans Arduino IDE :

```text
Fichier
→ Préférences
→ URL de gestionnaire de cartes supplémentaires
```

Installer ensuite le package ESP32 approprié via :

```text
Outils
→ Type de carte
→ Gestionnaire de cartes
```

---

## Étape 2 – Choisir la carte

Sélectionner le modèle ESP32 correspondant à votre matériel.

Par exemple, pour un ESP32 DevKit classique, sélectionner la carte correspondant exactement au module utilisé.

---

## Étape 3 – Placer les fichiers

Dans le même dossier :

```text
Programmateur_horaire_ESP32_Multi_grilles_horaire_au_choix_V2_sans_auth_plages.ino
arduino_secrets.h
```

Le nom du dossier Arduino doit normalement correspondre au nom principal du sketch.

---

# 44. Première mise en service

Ordre recommandé :

1. vérifier le câblage ;
2. vérifier l'alimentation ;
3. vérifier les GPIO ;
4. créer `arduino_secrets.h` ;
5. installer les bibliothèques ;
6. sélectionner la bonne carte ESP32 ;
7. téléverser par USB ;
8. ouvrir le moniteur série à `115200 bauds` ;
9. vérifier la connexion Wi-Fi ;
10. vérifier l'adresse IP ;
11. vérifier le fonctionnement de `richardv.local` ;
12. vérifier l'heure NTP ;
13. vérifier chaque relais ;
14. vérifier les boutons physiques ;
15. vérifier les horaires ;
16. tester ensuite l'OTA.

---

# 45. Test recommandé des relais

Avant de connecter une charge réelle, tester les relais avec une charge de test adaptée.

Pour chaque relais :

1. ouvrir la page Web ;
2. vérifier le mode AUTO ;
3. passer en MANUEL ;
4. forcer ON ;
5. vérifier le relais ;
6. forcer OFF ;
7. vérifier le relais ;
8. revenir en AUTO ;
9. programmer une plage très courte ;
10. vérifier le changement automatique.

---

# 46. Test des plages traversant minuit

Exemple :

```text
23:55-00:05
```

Tester :

```text
23:54 → OFF
23:55 → ON
00:00 → ON
00:04 → ON
00:05 → OFF
```

Ce test permet de vérifier le fonctionnement du passage à minuit.

---

# 47. Test de conservation après redémarrage

Après avoir configuré une plage :

```text
06:32-08:10
```

faire :

1. sauvegarder ;
2. attendre la confirmation ;
3. redémarrer l'ESP32 ;
4. ouvrir la page Web ;
5. vérifier que la plage est toujours présente.

---

# 48. Test de conservation après OTA

Configurer une plage personnalisée.

Exemple :

```text
06:17-08:23
```

Puis :

1. compiler une nouvelle version ;
2. effectuer une mise à jour OTA ;
3. attendre le redémarrage ;
4. ouvrir « Infos système » ;
5. vérifier que `FIRMWARE_BUILD` a changé ;
6. vérifier que la plage `06:17-08:23` est toujours présente.

Si la plage est conservée, la gestion NVS fonctionne comme prévu.

---

# 49. Dépannage Wi-Fi

## L'ESP32 ne se connecte pas

Vérifier :

- SSID ;
- mot de passe ;
- présence du réseau ;
- bande Wi-Fi compatible ;
- contenu de `arduino_secrets.h`.

Le moniteur série doit indiquer les tentatives de connexion.

---

# 50. `richardv.local` ne fonctionne pas

Tester d'abord l'adresse IP affichée dans le moniteur série.

Exemple :

```text
http://192.168.1.50
```

Si l'IP fonctionne mais pas :

```text
http://richardv.local
```

le problème est probablement lié à la résolution mDNS du client.

Vérifier :

- smartphone et ESP32 sur le même réseau local ;
- réseau non isolé ;
- support mDNS du navigateur/système ;
- absence de VPN bloquant le réseau local ;
- configuration du routeur.

Le fonctionnement par IP ne dépend pas de mDNS.

---

# 51. Le relais ne réagit pas

Vérifier dans l'ordre :

1. GPIO réellement utilisé ;
2. alimentation du module relais ;
3. masse commune ;
4. logique active HIGH/LOW ;
5. valeur de `relayActiveHigh` ;
6. mode AUTO/MANUEL ;
7. état de la programmation horaire.

Si le relais fonctionne à l'inverse :

```cpp
true
```

peut être remplacé par :

```cpp
false
```

pour le relais concerné.

---

# 52. Le relais reste bloqué en MANUEL

C'est normalement le comportement prévu.

Un relais en MANUEL ignore les horaires.

Utiliser :

```text
/reset-auto
```

pour repasser tous les relais en AUTO.

Cette commande nécessite l'authentification Web.

---

# 53. Une plage ne se sauvegarde pas

Vérifier :

- format `HH:MM-HH:MM` ;
- nombre de plages inférieur ou égal à `MAX_PLAGES` ;
- heure valide ;
- minutes comprises entre `00` et `59` ;
- plage non nulle.

Exemples :

```text
06:30-08:00
```

valide.

```text
06:30-06:30
```

refusé.

---

# 54. Attention aux modifications du tableau `programmateurs[]`

Les valeurs présentes dans :

```cpp
plagesDefaut
modeAuto
relayState
```

sont des **valeurs par défaut**.

Si une valeur existe déjà dans la NVS, elle est prioritaire.

Par exemple, changer :

```cpp
"06:32-08:10"
```

en :

```cpp
"07:00-09:00"
```

dans le code ne remplacera pas automatiquement une programmation déjà sauvegardée dans la NVS.

Pour modifier une programmation utilisateur, utiliser l'interface Web.

---

# 55. Ajouter un nouveau relais après utilisation

Si vous ajoutez :

```cpp
{ "5", ... }
```

le nouveau relais n'aura normalement aucune ancienne configuration NVS associée à son nouvel identifiant.

Ses valeurs `plagesDefaut`, `modeAuto` et `relayState` pourront donc servir de valeurs initiales.

---

# 56. Modifier CONFIG_VERSION

Ne faire ceci que lors d'une modification incompatible de la structure NVS :

```cpp
const uint16_t CONFIG_VERSION = 3;
```

Au prochain démarrage, la configuration du namespace `config` sera réinitialisée.

### Conséquence

Les horaires et états sauvegardés seront perdus.

Avant une telle modification, noter ou sauvegarder les horaires.

---

# 57. Structure logique du programme

Le fichier `.ino` est organisé autour des blocs suivants :

```text
Bibliothèques
    ↓
Configuration Wi-Fi
    ↓
Structures Plage / Programmateur
    ↓
Tableau programmateurs[]
    ↓
Anti-rebond boutons
    ↓
Gestion des plages
    ↓
Calcul des plages actives
    ↓
OLED
    ↓
HTML / CSS / JavaScript
    ↓
NVS / Preferences
    ↓
Wi-Fi
    ↓
mDNS
    ↓
OTA
    ↓
Setup
    ↓
Routes HTTP
    ↓
Application programmation
    ↓
Boutons physiques
    ↓
Loop
```

---

# 58. Principes importants du programme

## La NVS contient les réglages utilisateur

Elle ne doit pas être effacée à chaque OTA.

## Le tableau `programmateurs[]` contient les valeurs par défaut

Il sert à initialiser un relais qui n'a pas encore de configuration sauvegardée.

## `FIRMWARE_BUILD` sert au diagnostic

Il indique la compilation réellement installée.

## `CONFIG_VERSION` sert à la compatibilité NVS

Elle ne doit changer que si la structure NVS change.

## AUTO et MANUEL sont indépendants

MANUEL prend le contrôle du relais jusqu'au retour en AUTO.

---

# 59. Précautions électriques

Le programme peut commander des relais connectés à des tensions dangereuses.

Si les relais commandent du :

- 230 V AC ;
- moteur ;
- chauffage ;
- éclairage secteur ;
- équipement industriel ;

le câblage doit respecter les règles de sécurité électrique applicables.

L'ESP32 et sa partie basse tension doivent être correctement isolés du secteur.

Utiliser :

- boîtier adapté ;
- fusibles/protections appropriés ;
- borniers adaptés ;
- distances d'isolement suffisantes ;
- relais correctement dimensionnés.

Ne jamais manipuler un câblage secteur sous tension.

---

# 60. Résumé des paramètres principaux

| Paramètre | Valeur actuelle | Fonction |
|---|---:|---|
| `MAX_PLAGES` | 6 | Nombre max de plages/relais |
| `CONFIG_VERSION` | 2 | Version de structure NVS |
| `OLED_LIGNES_PAR_PAGE` | 5 | Relais affichés par page OLED |
| `BP_DEBOUNCE_MS` | 40 ms | Anti-rebond boutons |
| `RSSI_SWITCH_MARGIN` | 8 dBm | Marge de changement réseau |
| Nom mDNS | `richardv` | Adresse `richardv.local` |
| Port Web | 80 | HTTP |
| OLED | 128×64 | SSD1306 |
| Adresse OLED | `0x3C` | I2C |
| SDA | GPIO21 | I2C |
| SCL | GPIO22 | I2C |
| NTP | `pool.ntp.org` | Synchronisation heure |
| NTP secondaire | `time.google.com` | Synchronisation heure |

---

# 61. Checklist avant installation définitive

- [ ] `arduino_secrets.h` correctement renseigné
- [ ] bon modèle ESP32 sélectionné
- [ ] bibliothèques installées
- [ ] OLED détecté en `0x3C`
- [ ] GPIO relais vérifiés
- [ ] logique HIGH/LOW des relais vérifiée
- [ ] alimentation correcte
- [ ] boutons correctement câblés
- [ ] Wi-Fi connecté
- [ ] adresse IP connue
- [ ] `richardv.local` testé
- [ ] heure NTP synchronisée
- [ ] toutes les plages testées
- [ ] plages traversant minuit testées
- [ ] conservation NVS testée
- [ ] OTA testée
- [ ] mode MANUEL testé
- [ ] retour AUTO testé
- [ ] charges réelles testées avec précautions

---

# 62. Fichiers du projet

Structure recommandée :

```text
Programmateur_horaire_ESP32_Multi_grilles_horaire_au_choix_V2_sans_auth_plages/
│
├── Programmateur_horaire_ESP32_Multi_grilles_horaire_au_choix_V2_sans_auth_plages.ino
├── arduino_secrets.h
└── README.md
```

Le fichier `arduino_secrets.h` doit rester privé car il contient les informations d'accès au Wi-Fi et le mot de passe OTA.

---

# 63. Version du programme documentée

Ce README correspond au programme :

```text
Programmateur_horaire_ESP32_Multi_grilles_horaire_au_choix_V2_sans_auth_plages.ino
```

Caractéristiques importantes de cette version :

- NVS conservée lors des compilations/OTA ordinaires ;
- version de configuration NVS ;
- diagnostic `FIRMWARE_BUILD` ;
- plages libres à la minute ;
- gestion du passage par minuit ;
- gestion de `24:00` ;
- validation avant sauvegarde ;
- authentification supprimée uniquement pour `/save` ;
- authentification conservée pour les commandes de relais et de mode ;
- gestion du niveau actif HIGH/LOW par relais ;
- application immédiate des changements ;
- Wi-Fi non bloquant ;
- mDNS ;
- OTA ;
- OLED ;
- boutons physiques.

---

## Licence / utilisation

Ce programme est destiné à un usage personnel et expérimental avec ESP32.

Avant toute utilisation avec une installation électrique ou une charge dangereuse, vérifier le dimensionnement matériel, l'isolation et les règles de sécurité applicables.
