# Mon Sommeil : application Android native

Cahier des charges pour Claude Code. Projet personnel d'Eric : une appli d'analyse du sommeil par le micro, inspirée de Sleep Cycle, gratuite, sans publicité, sans compte et sans réseau. Elle remplace la version web (`../index.html`, publiée sur https://eric-newik.github.io/mon-sommeil/) et en reprend les règles de détection, l'analyse et le design.

## Pourquoi une version native
La version web a deux limites que rien ne corrige dans un navigateur :
- elle doit garder l'écran allumé toute la nuit. Si le téléphone se verrouille (par exemple pour regarder l'heure), le micro s'arrête : le 26/11, l'écoute s'est coupée de 5 h 01 à 7 h 45 ;
- Android met la page en veille : le réveil a sonné 5 min en retard.

L'appli native doit donc écouter **écran éteint**, dans un service de premier plan, et réveiller avec une **vraie alarme système**.

## Contraintes
- Téléphone cible : **OPPO Find X3 Lite 5G (CPH2145), Android 11 / ColorOS 13.1**. Prévoir minSdk 26, targetSdk à jour.
- Kotlin, Jetpack Compose, Room pour les données. Aucune dépendance réseau, aucune permission INTERNET. Aucune analyse ni publicité.
- Interface **en français**, thème sombre, mêmes couleurs que la version web (voir « Design »).
- Installation par APK (débogage USB depuis Android Studio), sans Play Store.
- Toutes les données restent sur le téléphone. Export et import JSON pour sauvegarder et pour l'analyse avec Claude.
- Honnêteté : les phases sont **estimées à partir des sons et des mouvements**. Ce n'est pas un appareil médical ni un diagnostic ; le dire dans l'appli comme dans la version web.
- Utiliser **Context7** pour la documentation à jour d'Android (foreground service de type `microphone`, `AlarmManager.setAlarmClock`, permissions Android 13/14, Compose, Room).

## Architecture
1. **Service d'écoute** (`ForegroundService`, `foregroundServiceType="microphone"`, notification permanente discrète « Mon Sommeil écoute ») :
   - `AudioRecord` mono 16 kHz, `VOICE_RECOGNITION` ou `UNPROCESSED` si disponible, sans AGC, sans suppression de bruit ;
   - analyse en continu par trames de 2048 échantillons à 48 kHz dans la version web, soit environ 43 ms : garder la même durée de trame (environ 680 échantillons à 16 kHz) pour que les seuils restent valables ;
   - une FFT par trame pour les bandes graves (60-300 Hz), médiums (300-3000 Hz) et aigus (3-8 kHz) ;
   - mémoire tampon circulaire de 75 s pour découper les extraits audio, rien n'est enregistré en continu ;
   - `PARTIAL_WAKE_LOCK` pendant la nuit, enregistrement minute par minute dans Room (pour ne rien perdre si l'appli est tuée) ;
   - délai avant l'écoute réglable (0, 5, 10, 15 ou 20 min ; 10 par défaut).
2. **Alarme** : `AlarmManager.setAlarmClock()` pour l'heure limite (sonne même si le service meurt), plus le réveil intelligent décidé par le service dans la fenêtre. L'écran de sonnerie s'affiche par-dessus l'écran de verrouillage (full-screen intent), avec montée progressive du volume, vibration, snooze (5, 9 ou 10 min) et choix de sonnerie (deux sons intégrés et un fichier personnel).
3. **ColorOS** : au premier lancement, guider Eric pas à pas pour désactiver l'optimisation de batterie pour l'appli, autoriser le démarrage automatique et l'activité en arrière-plan (ColorOS tue les services agressivement). Vérifier ces réglages avant chaque nuit et prévenir s'ils manquent.
4. **Analyse** après la nuit, puis réanalyse automatique des nuits enregistrées quand les règles changent (`rules` = numéro de version des règles, comme dans la version web).

## Règles de détection (à reprendre telles quelles)
La référence exacte est le code de `../index.html` (classe `Tracker`, fonctions `evRule`, `analyze`, `finalizeNight`, `smartOK`). **Lire ce fichier et porter les seuils à l'identique**, puis vérifier avec les tests ci-dessous. Résumé :
- Fond sonore : calibré sur les 50 premières trames (30e centile), puis adaptatif (descend vite, monte lentement). Un événement est un son au-dessus de `fond + seuil` (seuil : 12, 8 ou 5 dB selon la sensibilité faible, moyenne ou élevée ; moyenne par défaut). Il se termine après 0,35 s de calme, ou au bout de 12 s.
- Caractéristiques gardées pour chaque son : durée, pic au-dessus du fond, part des graves, médiums et aigus, nombre de syllabes, attaque (temps jusqu'au pic − 6 dB).
- Classement dans l'ordre : bruit (≥ 11 s), ronflement fort, parole, toux, candidat ronflement, mouvement, avec la règle « ronflement en contexte » (version 2.7). Un ronflement est confirmé après 3 sons réguliers (intervalles de 1,8 à 10 s).
- Comptage (`evRule`, version 2.8) : toux seulement au-delà de 40 dB au-dessus du fond ; parole plus comptée ; un « mouvement » fort, bref et sans aigus n'est pas compté.
- Pauses respiratoires : silence de 10 à 90 s juste après un ronflement installé (5 candidats dans la minute en direct, 3 ronflements confirmés à la réanalyse), avec un ronflement médian d'au moins 30 dB et une reprise forte. C'est un **indice**, pas un diagnostic d'apnée.
- Phases minute par minute (éveil, léger, paradoxal, profond) à partir de l'activité lissée, des pics d'agitation (fins de cycle) et du ronflement ; score de 0 à 100. Reprendre `analyze()` à l'identique.
- Réveil intelligent (`smartOK`, version 2.3), vérifié toutes les 30 s dans la fenêtre : phase estimée éveil ou léger, pas de ronflement dans les 2 dernières minutes, agitation des 2 dernières minutes ≥ max(2, 2 × 75e centile de la nuit). Sinon, sonnerie à l'heure. Il reste à valider sur de vrais matins.
- Si l'écoute s'interrompt, enregistrer la coupure et l'exclure de l'analyse (version 2.10).

Points encore incertains, à garder réglables :
- les « mouvements » ordinaires n'ont presque jamais été confirmés (1 sur 19 étiquetés ; la plupart étaient des ronflements). La version 2.9 garde des extraits « Son grave régulier » et la 2.10 des « Mouvements ordinaires » tirés au hasard pour trancher ;
- la seule vraie toux étiquetée (23/11) n'avait presque pas d'aigus, contrairement à la règle actuelle.

## Fonctions et écrans (reprendre la version web)
- **Nuit** : réglage du réveil (heure, réveil intelligent, fenêtre de 10 à 45 min), carte des exercices anti-ronflement, bouton « Commencer la nuit ». Pendant la nuit, l'écran peut s'éteindre ; l'écran de suivi (heure, compteurs) n'apparaît que si Eric rallume le téléphone.
- **Exercices anti-ronflement** : les 5 exercices guidés (protocole Ieto et al., Chest 2015), 20 répétitions au rythme d'une toutes les 2 s, avec bip et vibration, jours d'affilée ; étiquette `exos` ajoutée automatiquement à la nuit.
- **Rapport** : score, analyse écrite de la nuit (règles, sans IA), « Ta soirée » (alcool, sport, café, repas tardif, stress, écrans, côté, exercices), graphique « Ta nuit » (hypnogramme en escalier, cycles, pistes Mouvements, Ronflement, Pauses et toux), phases, événements, carte Respiration, niveau sonore relatif.
- **Extraits audio** (5, 10 ou 20 par nuit, WAV, au moins 2 de chaque type) avec **requalification** : « Vraiment un … ? », Oui / Non et ce que c'était vraiment. Les étiquettes sont gardées dans la nuit (`labels`) et dans l'export : c'est ce qui sert à recalibrer.
- **Historique** : liste des nuits, analyse sur le long terme, tendances (7 nuits, 30 nuits, tout).
- **Réglages** : sensibilité, extraits, sonnerie, délai, export et import JSON, effacement, nuit de démonstration.

## Données et compatibilité
- Garder **le même format de nuit** que la version web (`id`, `start`, `end`, `bed`, `status`, `m.act/db/dbn/mov/cough/talk/snore/noise`, `events` avec `x`, `snoreEps`, `snores`, `apneas`, `apneasRaw`, `gaps`, `alarm`, `tags`, `labels`, `analysis`, `counts`, `rules`, `sens`), pour que :
  - l'**import** du fichier exporté par la version web (`mon-sommeil-AAAA-MM-JJ.json`) récupère toutes les nuits d'Eric depuis le 13/11/2026 ;
  - l'**export** de l'appli native reste lisible par les scripts d'analyse de Claude.
- Les extraits audio ne sont pas dans l'export JSON (ils restent sur le téléphone), comme dans la version web.

## Design
- Couleurs : fond `#0b1020`, cartes `#151b30` / `#1e2645`, texte `#e8ecf8`, secondaire `#8c95b3`, accent `#8aa4ff`. Phases : éveil `#f2b84b`, léger `#5ec8f2`, paradoxal `#c77dff`, profond `#3d5afe`. Alerte `#ff6b6b`, bon `#4cd6a0`.
- Coins arrondis (cartes 18 dp), grandes cibles tactiles, police système.
- **Icône** : dossier `icone/`. Icône adaptative = `ic_launcher_foreground.svg` (lune, étoiles, petit hypnogramme, dans la zone sûre centrale) + `ic_launcher_background.svg` (dégradé bleu nuit). Importer avec Android Studio › New › Image Asset. `icone-1024.png` et `icone-512.png` servent d'aperçu.

## Étapes conseillées
1. Créer le projet (Compose, Kotlin, minSdk 26) avec l'icône, puis installer une coquille vide sur le téléphone pour valider la chaîne USB.
2. Service d'écoute écran éteint, avec seulement le niveau sonore par minute : faire une vraie nuit de test et vérifier qu'il n'y a **aucune coupure**.
3. Porter les règles de détection et l'analyse ; tests unitaires qui rejouent un export JSON réel et doivent retrouver les mêmes `counts` et `analysis` que la version web (à ±1 minute près).
4. Alarme système et réveil intelligent.
5. Écrans (rapport, historique, réglages), extraits et requalification, import et export.
6. Exercices anti-ronflement.

Avant de livrer une étape : compiler, lancer les tests, installer sur le téléphone, puis noter la version et les changements dans `JOURNAL.md`.
