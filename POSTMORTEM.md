# Postmortem : Attaque force brute SSH sur serveur

**Date de l'incident :** 22 octobre 2026
**Auteur :** Kael Avande
**Statut :** Terminé

## Résumé exécutif
Le serveur de test a subi une compromission temporaire suite à une **attaque par force brute SSH menée avec succès**. Un compte de service doté d'un mot de passe faible a été compromis, permettant à l'attaquant de s'introduire sur la machine mais il n'a pas changé le mot de passe.
* **Impact :** Pas de fuite de données constatée.

## Cartographie MITRE ATT&CK
L'analyse des journaux permet d'aligner les actions de l'attaquant sur le référentiel MITRE ATT&CK :
* **Tactique : Accès Initial (TA0001)** → **Technique : Brute Force (T1110.001)** via des tentatives massives de connexion SSH par dictionnaire sur le port 22.
* **Tactique : Exécution (TA0002)** → **Technique : Interpréteur de commandes et de scripts (T1059.004)** via l'ouverture d'une session Bash interactive après authentification.

## Chronologie des événements

| Heure | Événement |
|-------|-----------|
| 15h32 | Début d'un pic massif de tentatives d'authentification SSH échouées sur le serveur de test. |
| 15h37 | **Compromission réussie :** Authentification valide sur le compte test depuis l'IP externe `172.20.20.27`. |
| 15h45 | Déconnexion du serveur

## Analyse des causes

### Cause immédiate
Le compte de test utilisait un mot de passe hérité, trop simple (faible entropie), ce qui a permis à l'attaque par dictionnaire (**MITRE T1110.001**) de réussir en moins de 5 minutes.

### Causes contributives
1. **Exposition directe :** Le port SSH (22) du serveur était ouvert à l'ensemble d'Internet sans restriction IP.
2. **Pas de bannissement automatique :** L'outil Fail2ban n'était pas activée.
3. **Supervision tardive :** Le SIEM a alerté sur l'activité post-compromission, mais pas sur la phase de force brute massive.

### Cause profonde
Absence de politique stricte de durcissement (*hardening*) des accès SSH externes et de gestion rigoureuse des exceptions.

## Ce qui a bien fonctionné
- Traces et logs système (`auth.log`) intacts, ce qui a facilité l'analyse *forensic*.

## Ce qui a mal fonctionné
- Fail2ban était inactif sur cette instance, laissant le champ libre à l'attaquant.
- Le bruit généré par le force brute initial (milliers de connexions échouées) n'a pas déclenché d'alerte critique immédiate.

## Actions correctives

- Changer de port SSH
- Audit complet de la robustesse des mots de passe

## Leçons apprises
1. Il faut toujours utiliser un mot de passe fort (avec des caractères spéciaux, ...)
2. Le maintien en condition de sécurité (ex: Fail2ban) doit être vérifié de manière automatisée.