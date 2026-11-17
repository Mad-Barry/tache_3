<!-- 

Fait:

1. modifier le workflow github actions pour que le build échoue si un commit introduit une nouvelle vulnérabilité que semgrep trouve

2. décrire et expliquer comment on a implémenté la modification pour le workflow en plus de la manière dont la modification a été validée (validée par qui, jsp)


À Faire:

3. modifier le code de Tika afin d’introduire 7 vulnérabilités de types différents détectables par Semgrep

4. décrire et justifier chaque vulnérabilité qu'on rajoute, donc le choix du fichier modifié, le type de vulnérabilité, la raison pour laquelle le code introduit est vulnérable et le résultat obtenu avec semgrep (en générant un rapport)

5. comparer le rapport Semgrep obtenu avant les modifications avec le rapport obtenu après l’introduction des vulnérabilités, afin de montrer quelles vulnérabilités sont nouvellement introduites (je pense pas on a besoin de décrire énormément, pt juste les énumérer)

6. introduire un élément d'humour dans la suite de test du cas d'étude: rickroll quand un cas de test du cas d'étude échoue. Pour cela, vous invoquerez une action réutilisable qui rickroll, par exemple, en modifiant la github action existante random-rickroll, ou bien faire votre propre implémentation d'une action qui rickroll, par exemple en rust. (flemme de programmer en rust, je pense que tout ce qu'on pourrait faire c'est d'ajouter un message du genre "womp womp, le build a échoué" avec un emoji de clown)
-->


<!-- =========================================================== rapport ==================================================================== -->



<!-- Version du repo github avant les ajouts des vulnérabilités: ddb1ebe714bec624404f821d9186d53373b48e2d  (à mettre qqpart dans le rapport)-->

## Github Actions (GA)
La tâche nous demande de modifier le Github Action (GA) pour que l'action échoue si un commit introduit une ou plusieurs nouvelles vulnérabilités qui sont détectées par semgrep.

1. Comment décider quel fichier modifier?

    Afin de modifier les GA et le workflow, nous avons pris le document "main-jdk17-build.yml" pour lui ajouter les actions voulues. Nous avons pris ce document en particulier plutôt que les autrs puisque les documents "main-jdk21-build.yml" et "main-jdk25-build.yml" n'ont pas l'option de "on: pull request", ils ont juste l'action de push. Puisqu'on veut que l'action échoue si un commit introduit une ou plusieurs nouvelle vulnérabilité, alors on a forcément besoin d'avoir cette option. Si on prenait "main-jdk21-build.yml" ou "main-jdk25-build.yml", le workflow ne serait compilé qu'après que le repo github ait déjà reçu le nouveau commit et il serait trop tard pour l'arrêter.

2. Explication des modifications

    Nous avons ajouté la section semgrep dans la section jobs, afin que l'action soit faite tout comme `build` et `e2e-tests`.

    Puisque pour `build` et `e2e-tests` roulaient sur `ubuntu-latest`, alors nous avons décidé de prendre la même chose pour semgrep. De la même façon, un timer maximum de 30 minutes à été pris. Pour éviter d'avoir à installer semgrep avec un `pip -install semgrep`, on utilise une image Docker pour le conteneur, car semgrep y est déjà installée.

    Pour les `steps` à suivre, on récupère d'abord tout l'historique git avec `fetch-depth: 0` pour pouvoir que `--baseline-commit` puisse trouver le commit de comparaison plus tard. On exécute ensuite le bloc de code se trouvant dans `name: Trouver le commit de base`. On a besoin de celui-ci puisque sans, on ne peut pas savoir quels sont les problèmes qui existaient déjà dans Tika et ceux que le commit qu'on veut faire ajoute. On exécute ensuite le bloc de code de `Lancer Semgrep` pour pouvoir lancer semgrep et trouver les erreurs que le nouveau commit apporterait. Un rapport est ensuite généré et enregistré dans `semgrep_rapport`. Pour les cas où semgrep échoue, on a quand même `Conserver le rapport` qui enregistre une version du rapport tout le temps.

## Vulnérabilités

7 vulnérabilités, types différents.

CWE-78 (command injection), TesseractOCRParser.java ligne 550. Le check du binaire tesseract fait un `exec()` avec le chemin concaténé direct, sans validation. Regle semgrep: `command-injection-formatted-runtime-call`.

CWE-330 (aléa faible), même fichier ligne 441. Le nom du fichier de debug OCR est genere avec `new Random()`, pas crypto, donc devinable. Regle: `weak-random`.

CWE-611 (XXE), XFAExtractor.java ligne 176. Si le XFA est corrompu, on retombe sur une factory XML sans protection, ce qui reouvre la porte aux entités externes. Regle: `xmlinputfactory-possible-xxe`.

CWE-328 (hash faible), OPCPackageDetector.java ligne 165. Dédup des logs de debug via MD5, facilement cassable. Regle: `use-of-md5`.

CWE-798 (credentials en dur), NetworkParser.java ligne 58. Identifiant et mot de passe écrits en clair dans le code, utilisés en fallback. Regle: `detected-username-and-password-in-uri`.

CWE-295 (cert désactivé), NetworkParser.java lignes 114/117. Le `X509TrustManager` accepte n'importe quel certificat, protection TLS annulée. Regle: `insecure-trust-manager`.

CWE-502 (désérialisation), ComponentInstantiator.java ligne 129. Le cache est relu avec `readObject()` sans vérification, exploitable si quelqu'un y dépose un fichier malicieux. Regle: `object-deserialization`.

## Avant/après

Avant les 7 ajouts, semgrep trouvait rien de neuf sur ces fichiers (sauf dans core, 3 resultats existaient deja avant, du code Tika d'origine, pas nous). Après, chaque vuln ressort dans le rapport avec sa regle. Rapports complets (avant et après, par module) dans `semgrep_rapport/`.

## Humour

Rickroll si un test échoue dans la build (voir le step `Rickroll` dans `main-jdk17-build.yml`). Et [pour faire bonne mesure](https://youtu.be/dQw4w9WgXcQ).

