# WPM Interface

Interface web et firmware pour le WPM basé sur un Arduino MKR1000. Le projet permet de lire, écrire, réinitialiser et télécharger les presets des modules 200e, et de faire suivre une clock MIDI physique reçue par RTP-MIDI.

## Vue d'ensemble

Le projet contient trois parties principales :

- `index.html` : interface web complète, ouverte directement dans un navigateur avec `file:///`.
- `styles.css` : styles de l'interface, thèmes, modales et panneau MIDI.
- `2Wireless/2Wireless.ino` : firmware Arduino du MKR1000.
- `2Wireless/2Wireless.ino.bin` : binaire compilé prêt à téléverser.
- `empty_presets/` et `empty_preset_library.js` : presets vides utilisés par Reset.
- `module_catalog.js` : catalogue des modules et adresses I2C.

Aucun fichier `.cpp` Arduino n'est modifié par ce projet.

## Matériel et réseau

- Carte : Arduino MKR1000, processeur SAMD21.
- Point d'accès Wi-Fi : `BUCHLA200E`.
- Adresse du WPM : `192.168.0.1`.
- Serveur HTTP : port `80`.
- RTP-MIDI / AppleMIDI : ports UDP `5004` et `5005`.
- Ordinateur : souvent connecté sur `192.168.0.100`.

Le serveur HTTP doit répondre avant que l'interface affiche le WPM comme appairé. Le test minimal est :

```powershell
curl.exe --connect-timeout 5 --max-time 10 --silent --show-error http://192.168.0.1/midiclockstatus
```

Si cette commande expire ou renvoie `STATUS=000`, le problème est réseau ou firmware, pas JavaScript.

## Dépendances Arduino

Versions utilisées pendant cette session :

- Arduino CLI installé dans `C:\Program Files\Arduino CLI\arduino-cli.exe`.
- FQBN : `arduino:samd:mkr1000`.
- WiFi101 `0.16.1`.
- AppleMIDI `3.5.0`.
- MIDI Library `5.0.2`.
- MIDIUSB `1.0.5`.
- Arduino SAMD core `1.8.14`.

## Compiler et flasher

Depuis la racine du projet :

```powershell
arduino-cli compile --fqbn arduino:samd:mkr1000 .\2Wireless
arduino-cli upload -p COM3 --fqbn arduino:samd:mkr1000 .\2Wireless
```

Sur cette machine, si `arduino-cli` n'est pas dans le `PATH` :

```powershell
& 'C:\Program Files\Arduino CLI\arduino-cli.exe' compile --fqbn arduino:samd:mkr1000 .\2Wireless
& 'C:\Program Files\Arduino CLI\arduino-cli.exe' upload -p COM3 --fqbn arduino:samd:mkr1000 .\2Wireless
```

Pour produire le binaire livré avec le projet :

```powershell
& 'C:\Program Files\Arduino CLI\arduino-cli.exe' compile --fqbn arduino:samd:mkr1000 --export-binaries .\2Wireless
$built = Get-ChildItem .\2Wireless\build -Recurse -Filter '2Wireless.ino.bin' | Select-Object -First 1
Copy-Item $built.FullName .\2Wireless\2Wireless.ino.bin -Force
Remove-Item .\2Wireless\build -Recurse -Force
```

Le port peut changer après un reset ou un passage en bootloader. Vérifier avec :

```powershell
& 'C:\Program Files\Arduino CLI\arduino-cli.exe' board list
```

## Démarrage réseau et RTP-MIDI

Le firmware démarre le serveur HTTP, puis initialise AppleMIDI seulement lorsque le Wi-Fi est réellement connecté. Cette séquence est importante : démarrer AppleMIDI trop tôt empêche parfois les sockets UDP de s'ouvrir.

Le statut contient notamment :

- `rtp_started` : sockets AppleMIDI initialisés;
- `received` : clocks RTP reçues;
- `generated` : clocks produites par le timer physique;
- `sent` : clocks envoyées sur le bus I2C;
- `dropped` : ticks en attente remplacés par un nouveau tick;
- `send_failures` : erreurs d'envoi I2C;
- `max_jitter_us` : jitter observé sur la sortie.

Routes utiles :

```text
GET /midiclockstatus
GET /midiclockstart
GET /midiclockstop
GET /midiclocktempo?bpm=120
GET /midiclockfollow?enabled=1
GET /midiclockphasefollow?enabled=1
GET /midiclocklatency?ms=20
GET /midiclockphase?ms=0
GET /presentmodules
```

Pendant que la clock physique tourne, les opérations lourdes de presets sont volontairement refusées avec `423 Locked`. Il faut arrêter la clock avant Get, Set, Reset ou scan.

## Fonctionnement de la clock MIDI

Le WPM reçoit les messages MIDI temps réel RTP :

- `FA` : Start;
- `F8` : Clock;
- `FB` : Continue;
- `FC` : Stop.

La sortie physique est générée par le timer TC5 et envoyée au bus I2C en maître. Le timer ne met qu'un tick en attente : il n'accumule pas une rafale de ticks retardés, ce qui évite de rattraper brutalement le temps.

### Tempo

Paramètres actuels dans `2Wireless.ino` :

```cpp
#define MIDI_CLOCK_PPQN 24UL
#define MIDI_CLOCK_INPUT_DIVIDER 24U
#define MIDI_CLOCK_INPUT_EVENT_MULTIPLIER 6UL
#define MIDI_CLOCK_SMOOTHING_BEATS 4UL
#define MIDI_CLOCK_CHANGE_THRESHOLD_PCT 1UL
```

Interprétation :

- la sortie physique reste à 24 PPQN;
- la mesure est regroupée sur 24 ticks MIDI par bloc;
- la fenêtre déclarée est de 4 temps / 96 ticks;
- les callbacks RTP observés peuvent représenter plusieurs événements pour un même tick musical; le facteur `6` compense cette échelle observée;
- les variations de tempo inférieures à 1 % sont ignorées;
- au-delà de 1 %, le timer suit progressivement la nouvelle période;
- `Follow incoming tempo` doit être activé pour appliquer le tempo entrant. Sinon, le BPM personnel reste la référence et le tempo entrant est seulement mesuré.

Le firmware expose deux valeurs différentes :

- `tempo_set_bpm` : référence personnelle configurée dans l'UI;
- `estimated_tempo_bpm` : tempo calculé à partir des ticks entrants.

Il ne faut pas confondre ces deux valeurs lors du diagnostic.

### Phase et latence

Paramètres actuels :

```cpp
#define MIDI_CLOCK_LATENCY_COMPENSATION_US 20000L
#define MIDI_CLOCK_REALTIME_PHASE_LEAD_US 20000L
#define MIDI_CLOCK_PHASE_TRACK_DIVISOR 256UL
#define MIDI_CLOCK_PHASE_SMOOTHING_DIVISOR 16UL
```

- latence globale : `+20 ms`;
- anticipation temps réel : `+20 ms`;
- `Follow incoming phase` permet de suivre la phase RTP sans changer la référence de tempo;
- le slider `Phase correction` permet une correction manuelle indépendante;
- la correction de phase est amortie pour éviter les sauts du compteur TC5.

Ne jamais réintroduire une écriture agressive de `TC5->COUNT` dans le callback RTP. Le callback doit rester court : mesurer, accumuler, puis laisser la boucle et le timer faire le travail.

## Interface MIDI

Le bouton `MIDI` se trouve à côté de `Theme` et ouvre une modale centrée contenant :

- activation / désactivation de la clock physique;
- champ `Tempo reference` de 20 à 300 BPM;
- option `Follow incoming tempo`;
- option `Follow incoming phase`;
- slider de latence de -100 à +100 ms;
- slider de correction de phase de -100 à +100 ms;
- tempo entrant estimé en temps réel;
- phase mesurée en temps réel.

Le polling du statut est limité à la modale ouverte et ne remplace pas le champ BPM pendant que l'utilisateur le modifie.

Quand MIDI est actif, l'interface principale est désactivée et affiche qu'il faut arrêter MIDI avant Get, Set, Reset ou scan.

## Pairing et pannes HTTP

Le pairing utilise `/midiclockstatus`, car `/presentmodules` est verrouillé pendant la clock. Cela évite un faux message `WPM not paired` lorsque le WPM fonctionne mais refuse l'inventaire pendant MIDI.

Diagnostic conseillé :

1. Vérifier que le PC est connecté à `BUCHLA200E`.
2. Tester `/midiclockstatus`.
3. Tester `/presentmodules` seulement si `running` vaut `false`.
4. Si les deux routes ferment la connexion, faire un reset du MKR1000.
5. Si le problème persiste après reset, recompiler et flasher avec Arduino CLI.
6. Ne pas tester Get ou Get all pendant que la clock tourne.

Les symptômes suivants indiquent généralement un problème firmware ou réseau :

- `STATUS=000`;
- timeout HTTP;
- `connection reset`;
- le point d'accès est visible mais le port 80 ne répond pas.

## Presets et modules

Les fonctions Get, Set, Reset et Get all ont été stabilisées pendant la session :

- Get reprend les commandes de backup officielles Studio H;
- Get all télécharge les résultats réussis et conserve les modules échoués pour une nouvelle tentative;
- les retries sont différés pour ne pas saturer l'I2C;
- l'inventaire peut utiliser une liste en cache si le firmware officiel ne fournit pas `/presentmodules`;
- Reset utilise l'adresse du module réellement sélectionné, y compris les variantes 259e B/C/D;
- le parseur 259e utilise les marqueurs irréguliers réels et ne fabrique pas de presets vides;
- les overlays sont déplacés sous `body` et centrés par rapport à la fenêtre.

## Historique des changements importants

Les commits précédents indiquent les grandes étapes :

- `fb03dfd midi clock added` : ajout initial de la clock MIDI;
- `5f2de0b boot process updated` : mise à jour du démarrage;
- `683e187 fix pairing after page refresh` : correction du pairing après refresh;
- `d851313 update MIDI tempo sync and controls` : interface MIDI, réglages tempo/phase, télémétrie, suivi RTP, protections HTTP et firmware actuel.

## Commit et push

Le dépôt Git est : `https://github.com/modulow/wpm.git`.

Commandes standard :

```powershell
$git = "$env:LOCALAPPDATA\GitHubDesktop\app-3.6.5\resources\app\git\cmd\git.exe"
& $git status
& $git add README.md 2Wireless/2Wireless.ino 2Wireless/2Wireless.ino.bin index.html styles.css
& $git commit -m "document WPM MIDI and preset workflow"
& $git push origin main
```

Avant de pousser :

```powershell
& $git diff --check
```

## Notes pour continuer le développement

- Garder les callbacks AppleMIDI très courts.
- Ne pas modifier les fichiers `.cpp` de l'écosystème Arduino.
- Tester d'abord le serveur HTTP, puis RTP-MIDI, puis la clock physique, puis les opérations I2C.
- Éviter les requêtes HTTP fréquentes pendant une lecture MIDI : elles ajoutent du jitter.
- Ne pas utiliser un gros tri ou une grosse allocation dans le callback RTP.
- Toujours vérifier `received`, `estimated_tempo_bpm`, `generated`, `sent`, `dropped` et `send_failures` ensemble.
- Le tempo estimé peut être faux si aucun tick entrant n'a été reçu; dans ce cas la valeur affichée reste la dernière valeur connue ou la valeur initiale.
