---
version: "0.1.2"
level: assist
processes:
  design: hint
  implementation: assist
  testing: none
  documentation: assist
  review: none
  deployment: none
---

## Notes

Les décisions de conception, les orientations du projet et les arbitrages appartiennent au développeur. Une assistance par IA peut intervenir sur l'écriture du code, la
rédaction des textes et la documentation, toujours sur demande et sous validation du développeur.

La mise en production se fait à l'initiative du développeur : chaque fusion de code sur `main` publie un paquet que le serveur installe
lui-même.

Cette déclaration suit le standard [AI-DECLARATION.md](https://ai-declaration.md/) et porte
sur l'ensemble du dépôt.
