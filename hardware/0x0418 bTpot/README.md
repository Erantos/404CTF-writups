# Write-Up 404-CTF : 0x0418 bTpot

__Catégorie :__ Sécurité matérielle - Difficile

**Enoncé :**

![Enoncé du challenge](images/enonce.png)

**Fichiers :** System.py

**Résolution :**

Dans ce challenge, nous avons accès à un programme combinant Python et un HDL (Hardware Description Language) comme dans le challenge [L'Être ou le Néant](../l'etre%20ou%20le%20neant/README.md). Cependant ce programme a été obfusqué par les créateurs du challenge en remplaçant tous les noms de variables et de fonctions par des noms d'écrivains français, rendant la tâche de compréhension du programme beaucoup (mais raiment BEAUCOUP) plus difficile.

Petite parenthèse, j'ai passé pratiquement une journée entière à comprendre précisément le programme.

Le programme est décrit donc le fonctionnement d'un processeur avec son Unité Arithmétique et Logique, quelques registres et des flags d'erreurs. Il permet d'exécuter un programme compilé en hexadécimal. Voici les utilités des fonctions principales du programme :
```
Sand()   =>   Gère principalement le compteur ordinal (program counter)
Coulon()   =>   Gestion des opérations arithmétiques et logiques
Stael()   =>   Lecture dans une mémoire
Senghor()   =>   Fonction principale du processeur, redirige vers la bonne fonction du programme selon l'instruction et lève les bons flag
Alcott()   =>   Equivalent d'un fetch, affecte une instuction à un registre.    
Boetie()   =>   Initialisation du programme, premières vérification   
Supervielle()   =>   Equivalent à main(), initialise tous les modules et exécute les relient entre eux.
```

Syntaxe de chaque instruction sur 8 bits :
```
Instr (16 bits) = opcode (7 bits) | dest (3 bits) | op1 (3 bits) | op2 (3 bits)
ou
Instr (16 bits) = opcode (7 bits) | dest (3 bits) | address (6 bits)


INC - 0x23 | coco | gaboriau | beda -> reg[gaboriau] + 1 => reg[coco]
INC - 0x4f | coco | gaboriau | beda -> reg[gaboriau] + 1 => reg[coco]$

XOR - 0x3e | coco | gaboriau | beda -> reg[gaboriau] XOR reg[beda] => reg[coco]
XOR - 0x41 | coco | gaboriau | beda -> reg[gaboriau] XOR reg[beda] => reg[coco]

ADD - 0x42 | coco | gaboriau | beda -> reg[gaboriau] + reg[beda] => reg[coco]
ADD - 0x7e | coco | gaboriau | beda -> reg[gaboriau] + reg[beda] => reg[coco]

NOT - 0x12 | coco | gaboriau | beda -> NOT reg[gaboriau] => reg[coco]
NOT - 0x18 | coco | gaboriau | beda -> NOT reg[gaboriau] => reg[coco]

JZ (Jump if zero) - 0x3d | coco | gaboriau | beda -> Si reg[gaboriau] = 0 alors reg[beda] => reg[coco] ; sinon rien
JZ (Jump if zero) - 0x4a | coco | gaboriau | beda -> Si reg[gaboriau] = 0 alors reg[beda] => reg[coco] ; sinon rien

MUL - 0x2d | coco | gaboriau | beda -> reg[gaboriau] * reg[beda] => reg[coco]
MUL - 0x6f | coco | gaboriau | beda -> reg[gaboriau] * reg[beda] => reg[coco]

SHR (shift right) - 0x0e | coco | gaboriau | beda -> reg[gaboriau] >> reg[beda] => reg[coco]
SHR (shift right) -  0x11 | coco | gaboriau | beda -> reg[gaboriau] >> reg[beda] => reg[coco]

SHL (shift left) - 0x01 | coco | gaboriau | beda -> reg[gaboriau] << reg[beda] => reg[coco]
SHL (shift left) - 0x33 | coco | gaboriau | beda -> reg[gaboriau] << reg[beda] => reg[coco]

? - 0x29 | coco | gaboriau | beda -> ROT à droite de (reg[beda] % 8) bits de reg[gaboriau] => reg[coco]
? - 0x6c | coco | gaboriau | beda -> ROT à droite de (reg[beda] % 8) bits de reg[gaboriau] => reg[coco]

? - 0x27 | coco | gaboriau | beda -> ROT à gauche de (reg[beda] % 8) bits de reg[gaboriau] => reg[coco]
? - 0x45 | coco | gaboriau | beda -> ROT à gauche de (reg[beda] % 8) bits de reg[gaboriau] => reg[coco]

JMP - 0x13 | coco | bordes -> bordes => reg[coco]
JMP - 0x5a | coco | bordes -> bordes => reg[coco]

LOAD - 0x03 | coco | gaboriau | beda -> memory[reg[gaboriau]] => reg[coco]
LOAD - 0x6a | coco | gaboriau | beda -> memory[reg[gaboriau]] => reg[coco]

STORE - 0x46 | coco | gaboriau | beda -> prgm_memory_2[reg[gaboriau]] = reg[beda]
STORE - 0x66 | coco | gaboriau | beda -> prgm_memory_2[reg[gaboriau]] = reg[beda]

STOP - 0x0418
```

Grâce à cette phase de reverse ainsi qu'à l'énoncé, l'objectif du challenge devient plus clair : nous devons écrire un programme assembleur, le compiler en hexa et l'exécuter sur cette machine virtuelle. Le programme devra répondre au contrainte suivante :
- Calculer le modulo de nombres stockées initialement dans la mémoire et écrire le résultat dans un autre tableau en mémoire.
- Se limiter aux instructions ci-dessus (pas de modulo dispo, bien sûr)
- S'arrêter lorsque nous rencontrons un zéro en seconde opérande
- Ne pas provoquer d'erreur système
- Terminer proprement avec l'instruction STOP (0x0418)
- Le XOR de toutes les instructions doit valoir `0xcafe` en hexa (cf. fonction `Boetie`)
- Le programme doit être assez optimisé pour se terminer avant la fin de la simulation, soit 700 000 cycles d'horloges, et pour s'exécuter avec seulement 7 registres

Après plusieurs itérations, j'ai enfin réussi à obtenir un programme satisfaisant toutes les contraintes.  
La contrainte de temps est respecté grâce à un algo optimisé (contrairement à ma première version bruteforce, bien plus simple mais se terminant en 3 000 000 de cycles) et pour obtenir `0xcafe`, une instruction factice sans effet de bord est ajoutée en fin de programme.

Version logique en Python :
```python
memory_in # Mémoire interne accessible par LOAD
memory_out # Mémoire interne accessible par STORE

def mod(a, b):
    if (a == b)
        return 0
    elif (a < b)
        return a
    else
        return mod(a-b, b)

def main():
    idx_out = 0
    while True:
        idx_in = 2 * idx_out
        op1 = memory_in[idx_in]
        op2 = memory_in[idx_in + 1]
        if op2 == 0:
            return
        memory_out[idx_out] = mod(op1, op2)
        idx_out += 1
```

Version pseudo-assembleur optimisé :
```
      | --- Init ---                                                            
 0, 1 | r6 = 0 # idx_mem_out                                
      | --- Boucle 1 ---                                
 2, 3 | r5 = r6 + r6                                
 4, 5 | load mem[r5] r0 # op1                               
 6, 7 | inc r5                              
 8, 9 | load mem[r5] r1 # op2                               
10,11 | r4 = 0x3f                             
12,13 | r5 = 0xf                                
14,15 | r5 = r4 + r5     # @"End"                               
16,17 | Si r1 = 0 alors r7 = r5                             
      | --- Boucle 2 ----                               
18,19 | r2 = 8 # i                              
20,21 | r3 = r0 XOR r1   
22,23 | r5 = 0x7 
24,25 | r5 = r4 + r5  #@"Mod nul"
26,27 | Si r3 = 0 alors r7 = r5                  
      | --- Boucle 3 ---                                
28,29 | r5 = 0x1                                
30,31 | r5 = NOT r5                             
32,33 | Inc r5                              
34,35 | r2 = r2 + r5                                                         
36,37 | r4 = r3 >> r2                               
38,39 | r5 = 0x1c  # @"Boucle 3"                                
40,41 | Si r4 = 0 alors r7 = r5                             
      | --- Calcul ---                              
42,43 | r2 = NOT r2                             
44,45 | Inc r2 # r2 = -i                                
46,47 | r4 = 7                              
48,49 | r2 = r4 + r2                                
50,51 | r3 = r0 << r2                               
52,53 | r3 = r3 >> r4                               
54,55 | r4 = 0x3f                               
56,57 | r5 = 0x9                                
58,59 | r5 = r4 + r5  @"Store res"                              
60,61 | Si r3 = 0 alors r7 = r5                             
62,63 | r2 = NOT r1                             
64,65 | r2 = r2 + 1 # -r1                               
66,67 | r0 = r0 + r2                                
68,69 | r7 = 0x12   @"Boucle 2"
      | --- Mod nul ---
70,71 | r0 = 0                        
      | --- Store res ---                               
72,73 | store r0 mem[r6]                                
74,75 | inc r6                              
76,77 | r7 = 2 # @"Boucle 1"                                
      | --- End ---                             
78,79 | r3 = r3 + r6                               
80,81 | Stop    
```

Version compilé hexadécimal :
```
      | --- Init ---
 0, 1 | 0x13 | 0x6 | 0x0
      | --- Boucle 1 ---
 2, 3 | 0x42 | 0x5 | 0x6 | 0x6
 4, 5 | 0x03 | 0x0 | 0x5 | 0x0
 6, 7 | 0x23 | 0x5 | 0x5 | 0x0
 8, 9 | 0x03 | 0x1 | 0x5 | 0x0
10,11 | 0x13 | 0x4 | 0x3f
12,13 | 0x13 | 0x5 | 0xf
14,15 | 0x42 | 0x5 | 0x4 | 0x5
16,17 | 0x3d | 0x7 | 0x1 | 0x5
      | --- Boucle 3 ----
18,19 | 0x13 | 0x2 | 0x8
20,21 | 0x3e | 0x3 | 0x0 | 0x1
22,23 | 0x13 | 0x5 | 0x7
24,25 | 0x42 | 0x5 | 0x4 | 0x5
26,27 | 0x3d | 0x7 | 0x3 | 0x5
      | --- Boucle 2 ---
28,29 | 0x13 | 0x5 | 0x1
30,31 | 0x12 | 0x5 | 0x5 | 0x0
32,33 | 0x23 | 0x5 | 0x5 | 0x0
34,35 | 0x42 | 0x2 | 0x2 | 0x5
36,37 | 0x0e | 0x4 | 0x3 | 0x2
38,39 | 0x13 | 0x5 | 0x1c
40,41 | 0x3d | 0x7 | 0x4 | 0x5
      | --- Calcul ---
42,43 | 0x12 | 0x2 | 0x2 | 0x0
44,45 | 0x23 | 0x2 | 0x2 | 0x0
46,47 | 0x13 | 0x4 | 0x7
48,49 | 0x42 | 0x2 | 0x4 | 0x2
50,51 | 0x01 | 0x3 | 0x0 | 0x2
52,53 | 0x0e | 0x3 | 0x3 | 0x4
54,55 | 0x13 | 0x4 | 0x3f
56,57 | 0x13 | 0x5 | 0x9
58,59 | 0x42 | 0x5 | 0x4 | 0x5
60,61 | 0x3d | 0x7 | 0x3 | 0x5
62,63 | 0x12 | 0x2 | 0x1 | 0x0
64,65 | 0x23 | 0x2 | 0x2 | 0x0
66,67 | 0x42 | 0x0 | 0x0 | 0x2
68,69 | 0x13 | 0x7 | 0x12
      | --- Mod nul ---
70,71 | 0x13 | 0x0 | 0x0
      | --- Store res ---
72,73 | 0x46 | 0x0 | 0x6 | 0x0
74,75 | 0x23 | 0x6 | 0x6 | 0x0
76,77 | 0x13 | 0x7 | 0x2
      | --- End ---
78,79 | 0x7e | 0x3 | 0x3 | 0x6
80,81 | 0x02 | 0x0 | 0x3 | 0x0
```

Nous avons donc la payload finale suivante nous permettant d'obtenir le flag : `27808576062847680668273f274f85657bcd26887cc1274785657bdd27412568476884951d1a275c7be524904690270784a202c21cdc273f274985657bdd24884690840227d226008c3047b027c2fcde0418`

![Alt text](images/conversion.png)

**Flag :** `404CTF{0x1-0U-0b10_oP_coDe$_daNs_Vo7r3_0xcafe?}`