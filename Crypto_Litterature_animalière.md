# Write-Up 404-CTF : Littérature animalière

__Catégorie :__ Cryptanalyse

**Enoncé :**

![](https://s3.hedgedoc.org/demo/uploads/50eedaa3-95b4-4d95-b5fc-4e9b9698bfcf.png)

**Analyse :**

Dans ce challenge, nous avons à disposition un fichier _chiffre.wav_ dans lequel est caché un flag.

En essayant d'ouvrir le fichier tel un utilisateur de Windows lambda, on peut voir que le fichier ne s'ouvre pas correctement. En effet, il s'agissait d'un zip.

```
file chiffre.wav
> chiffre.wav: Zip archive data, at least v2.0 to extract, compression method=deflate
```
En le décompressant et en l'ouvrant, on obtient un dossier avec une image _chiffre.png_. Malheureusement, cette image a aussi des problèmes pour s'ouvrir (décidemment !). 

```
file chiffre.png
> chiffre.png: ASCII text, with very long lines (65512), with no line terminators
```

Le fichier est donc un texte, apparemment avec de très grandes lignes. Quand on l'affiche avec _cat_ (je l'ai fait mais en vrai ne le faites pas!), on voit une énorme suite de 0 et de 1 (avant que ma wsl crash à cause du nombre). En effet, on peut voir que le fichier ne fait qu'une ligne ... de 29 740 656 caractères...

Comme par hasard, ce nombre est un multiple de 8. Il est clair que nous devons regrouper les bits de ce fichier texte en octets pour en faire un nouveau fichier de 3,7Mo.

Cependant, en faisant ça, rien...
```
file decoded
> decoded: data
```

C'est là qu'il faut faire appel à notre sixième sens de hacker !

- Nous sommes dans un challenge crypto : le fichier a certainement dû être chiffré :eyes: 
- Quel chiffrement, assez simple pour que le chall soit classé facile, renvoie directement une série de bits ? Probablement un XOR

Le problème du XOR, c'est qu'il faut une partie du message en clair. Nous savons que le fichier final doit faire 3,7 Mo et nous devons en connaitre une partie.

**La révélation :** Et si chiffre.png ... était un PNG chiffré !

**Exploitation :**

En effet, le header des fichiers PNG est toujours le même (https://fr.wikipedia.org/wiki/Portable_Network_Graphics#Signature_PNG).

On récupère les 8 octets (64 bits) de la signature d'un PNG et on fait un XOR avec les 64 premiers bits de notre fichier.


```python
with open("chiffre.png", "r") as f:
    chiffre = f.read()

chiffre = chiffre[0:64]
png_sig = [137, 80, 78, 71, 13, 10, 26, 10]
res = ""

for i in range(0, len(chiffre), 8):
    byte = chiffre[i:i+8]
    xor = int(byte, 2) ^ png_sig[i//8]
    res += "{0:0>8b}".format(xor) # Convert int to bit string

print(res) 
```
On obtient alors la suite de bit si dessous, dans laquelle est cachée la (très probable) clé
```
0110010001101111110001001001000000101000110001101100100011011111
```
En effet, on voit que la série se répète à partir du 48ème caractères, visible si on écrit la suite sur deux lignes
```
01100100011011111100010010010000001010001100011
01100100011011111
```
On a donc trouvé la clé de chiffrement qui va nous permettre de déchiffrer le fichier.

```python
def decipher(cipher, key):
    b = []
    for i in range(0, len(cipher), 8):
        chunckFile = cipher[i:i+8]
        chunckKey = key[i:i+8]
        b.append(int(chunckFile, 2) ^ int(chunckKey, 2))
    return bytearray(b)

with open("chiffre.png", "r") as f:
    chiffre = f.read()

key = "01100100011011111100010010010000001010001100011"
a = len(chiffre) // len(key)
b = len(chiffre) % len(key)
extended_key = a * key + key[:b]

image = decipher(chiffre, extended_key)

with open("dechiffre.png", "wb") as f:
    f.write(image)
```
Et voilà ! On obtient une belle image après déchiffrement.

![](https://s3.hedgedoc.org/demo/uploads/a2b4b0a2-4511-44b0-84ec-211ea0d35e4e.png)

**Flag :** 404CTF{1g0rfu4v3rt1Env4ut2} 

**PS :** Le challenge était encore plus dur à flag du fait qu'il y avait une typo sur le CTFd. Merci au créateur pour avoir fait la correction, je n'aurais jamais réussi sans toi :heart: 





