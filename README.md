# Pipeline d'Acquisition — Transport & Logistique France

> Automatisation no-code n8n pour identifier, enrichir et scorer des PME françaises de transport et de logistique, dans le cadre d'une recherche de cibles d'acquisition. Pipeline orchestré en 33 nœuds, croisant Google Maps, le registre du commerce et Dropcontact pour produire une liste qualifiée d'entreprises avec leur score d'opportunité.

---

## En une lecture diagonale

- **Contexte** — mission ponctuelle pour une personne en recherche de cibles d'acquisition
- **Périmètre** — TPE/PME françaises de transport et logistique, codes NAF 49.xx, 52.xx et 53.xx
- **Approche** — orchestration no-code n8n, ingestion multi-sources, scoring déterministe
- **Sortie** — Google Sheet ventilé par région + export Excel + rapport email de synthèse
- **Couverture par run** — 480 combinaisons (48 villes × 10 mots-clés)

---

## L'origine du projet

Une personne en recherche de cibles d'acquisition dans le secteur transport-logistique m'a contacté avec un besoin précis : produire une liste qualifiée d'entreprises à approcher, accompagnée de leurs comptes sociaux, du nom de leur dirigeant et d'un email de contact direct. La phase manuelle qu'elle pratiquait — Google Maps, puis Pappers, puis recherche LinkedIn, puis enrichissement email — lui prenait plusieurs jours par run et restait peu reproductible d'une zone géographique à l'autre.

J'ai abordé la mission comme un audit de son workflow plutôt que comme un exercice technique. La question n'était pas *« quelle plateforme de scraping utiliser »* mais *« quelle est la suite minimale d'opérations qui transforme une zone géographique en liste qualifiée »*. La réponse — quatre sources de données, un filtre par code NAF, un scoring sur des critères financiers et opérationnels — s'est traduite en un pipeline n8n que la cliente pouvait, à terme, opérer elle-même sans dépendance technique.

---

## Le pipeline en quatre temps

```
[1] Découverte géographique
    Google Maps Text Search × 48 villes × 10 mots-clés
        ↓ déduplication par place_id
[2] Enrichissement opérationnel
    Google Maps Place Details — téléphone, site web
        ↓
[3] Enrichissement financier et juridique
    Pappers API — SIREN, code NAF, CA, résultat net, dirigeant
        ↓ filtre codes NAF transport/logistique uniquement
[4] Enrichissement contact
    Dropcontact — email du dirigeant si site web disponible
    sinon génération d'une URL de recherche LinkedIn ciblée
        ↓
[5] Scoring 0–100 et émission
    Google Sheet ventilé par région + Excel + rapport email
```

Chaque étape comporte un nœud `splitInBatches` et un `wait` paramétré, pour respecter les rate limits des APIs sans avoir à câbler une logique de retry sur mesure. Tous les appels HTTP sont configurés en `continueOnFail` ; les erreurs sont logguées dans un onglet dédié plutôt que d'arrêter la chaîne complète. C'est un choix structurant — sur 2 500 entreprises traitées, quelques échecs ponctuels d'API ne doivent pas faire perdre les 2 495 résultats valides.

---

## Critères de scoring

| Critère | Points |
|---|---|
| CA entre 1 M€ et 50 M€ | +30 |
| Effectif entre 10 et 250 salariés | +20 |
| Entreprise créée avant 2005 | +20 |
| Résultat net positif | +15 |
| Site web présent | +5 |
| Email dirigeant trouvé | +10 |

Trois catégories en sortie : **Cible Prioritaire** (≥ 70), **À Qualifier** (40–69), **Faible Potentiel** (< 40). Les seuils ont été calibrés avec la cliente sur un échantillon manuel de 50 entreprises avant de lancer le run complet.

---

## Stack

| Couche | Outil |
|---|---|
| Orchestration | n8n (self-hosted via Docker Compose) |
| Données géographiques | Google Maps Places API |
| Données financières | Pappers API |
| Enrichissement email | Dropcontact |
| Persistance | Google Sheets + export Excel |
| Notifications | Gmail (rapport de fin de run) |

---

## Économie d'un run complet

Pour 480 combinaisons et environ 2 500 entreprises traitées :

| Service | Volume | Coût |
|---|---|---|
| Google Maps (Text Search + Place Details) | ~4 000 requêtes | ~90 $ |
| Pappers (plan Pro) | ~2 500 requêtes | inclus, 49 €/mois |
| Dropcontact | 500–800 enrichissements | inclus, 24 €/mois |
| **Total marginal par run** | | **~90–130 $** |

Comparé au temps humain qu'un consultant aurait facturé pour un travail équivalent, le ROI est immédiat dès le premier run.

---

## Pourquoi le code n'est pas hébergé ici

Le pipeline est calibré sur un cas d'usage client, et les credentials API y sont étroitement liés. Le présent dépôt documente l'approche, l'architecture et les arbitrages ; le workflow paramétré reste en local.

---

## Auteur

**Mancef Ferrah** — étudiant en master, candidat à un stage d'AI Engineer.
Mission réalisée en mars 2026, en autonomie.
