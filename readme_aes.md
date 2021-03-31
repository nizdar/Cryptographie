On dispose d’un message m clair et on va le crypter, avec l’algorithme d’AES, afin d’obtenir un message crypté C.
On chiffre par block de texte de 128 bit, c’est-à-dire 16 octets. Pour commencer la procédure du chiffrement AES on place le bloc de texte clair dans une matrice 4x4 où chaque case de la matrice contient un octet.
En utilisant le code ASCII (chaque caractère est codé sur 8 bits) le bloc à chiffrer comportera 16 caractères, ainsi chaque caractère correspond à un octet.
Je décris ici les étapes du programme AES que j’ai créé sous Python. Je suppose que je dispose en entré d’un bloc de 16 caractères sous forme d’une chaine de caractères et d’une clé K de 16 octet sous forme d’un tableau.
# Etape 0 : placer le texte dans une matrice 4x4
Soit m=‘‘JESERAISAVECVOUS’’ le bloc à crypter
1. Je crée une fonction toAscii() qui donne le code Ascii correspondant à une chaine de caractères

<pre><code>  
  def toAscii(chaine):
    chaineAscii=[]
    for c in m :
      chaineAscii.append(ord(c))
    return chaineAscii
 </code></pre>
    
La fonction prend comme paramètre une chaine de caractère et retourne une liste « chaineAscii » des codes Ascii correspondant à chaque caractère de « chaine ».

![Le texte m en code Ascii](https://user-images.githubusercontent.com/77968416/113199635-a6a43c00-9267-11eb-8fec-2d23f2dd166d.png)

2. Je crée une fonction toBoxes() qui va mettre les 16 éléments de « toAscii(m) » dans une matrice de taille 4x4
<pre><code>
  def toBoxes(liste):
    boxes=np.zeros((4,4), type(int))
    for j in range(4):
      for i in range(4):
        boxes[i][j]=liste[j*4+i]
    return boxes
</code></pre>

La fonction prend comme paramètre une liste de 16 éléments et retourne une matrice 4x4 « boxes » remplie, colonne par colonne, à partir de « liste ».

![La matrice correspondante au texte m](https://user-images.githubusercontent.com/77968416/113200959-457d6800-9269-11eb-861a-ff99e89019f5.png)

Je note w = toBoxes(toAscii(m)).

# Etape 1 : La fonction subBytes

Les éléments de w sont des chiffres dans F= {0,1,…..255}, la fonction subBytes est une bijection sur F, elle associe donc à chaque élément de w 
un unique élément dans F via une fonction inversible f de {0,1}8 dans {0,1}8. Sans rentrer dans le détail du calcul de f et au lieu de le faire 
à chaque fois, on stocke dans une table, appelée S-box, les images de tout élément de F en hexadécimal et on l’utilise de la façon suivante :

![image](https://user-images.githubusercontent.com/77968416/113201331-aad15900-9269-11eb-95d9-4d66f0ddfcd3.png)

Par exemple pour w(i, j) = 74 = 4a en hexadécimal on a subBytes(4a) = d6 = 214 en base décimale.

1. je crée une fonction find_box(), qui trouve l’image correspondante à un octet donné (en décimal, hexadécimal ou binaire) dans la table S-box comme décrit dans 
l’exemple ci-dessus.

<pre><code>
def find_box(s):
  for i in range(16) :
  j=0
  while j<16 and (i*16+j)!=s:
    j+=1
  if j!=16 :
    break
  return s_box[i][j]
</pre></code>

La fonction prend comme paramètre un entier s, et cherche son image dans la table s_box. La méthode de recherche est la suivante :

Pour 0 ≤i ≤15 et 0 ≤ j ≤15 on cherche le couple (i,j) tel que i x 16 + j = s ainsi ij est l’écriture de s en base hexadécimale. 
L’image de s via la table s_box correspond donc à l’élément qui se trouve à la ième ligne et jème colonne, c-à-d : s_box[i][j].

![L'image de 74 par la table s_box](https://user-images.githubusercontent.com/77968416/113202113-a8bbca00-926a-11eb-804f-a5d8f248e3a5.png)

2. Je crée la fonction subBytes() qui transforme chaque élément d’une matrice carrée de taille 4 via la table s_box.

<pre><code>
  def subBytes(boxes):
    boxes_trans=np.array([[0]*4]*4, type(int))
     for i in range(4):
      for j in range(4):
        boxes_trans[i][j]=find_box(boxes[i][j])
  return boxes_trans
</pre></code>

La fonction prend comme paramètre une matrice « boxes » de taille 4 et applique la fonction find_box() sur chacun de ses éléments. En fin on 
récupère la matrice transformée « boxes_trans ».

Pour w = toBoxes(toAscii(m)) on crée w = subBytes(w) 

![Image de la matrice w par subBytes](https://user-images.githubusercontent.com/77968416/113202788-7fe80480-926b-11eb-9b95-3808cc83af57.png)

# Etape 2 : la fonction shiftRows()

On effectue une permutation cyclique pour chaque ligne de la matrice w, tel que, les octets de la ligne i sont décalés de i indices.

<pre><code>
def shiftRows(boxes):
  boxes_shift=np.zeros((4,4), type(int))
  for i in range(4):
    for j in range(4):
      boxes_shift[i][j]=boxes[i][(j+i)%4]
  return boxes_shift
</pre></code>

La fonction prend en entrée une matrice de taille 4 « boxes », effectue la permutation décrite ci-dessus et renvoie la matrice permutée « boxes_shift ».

![shiftRows appliquée à la matrice w](https://user-images.githubusercontent.com/77968416/113203230-0e5c8600-926c-11eb-8896-f2950b8e575b.png)

# Etape 3 : La transformation mixColumns

Il s’agit de multiplier chaque colonne de la matrice w par la matrice :

![matrice](https://user-images.githubusercontent.com/77968416/113203390-482d8c80-926c-11eb-8302-942a7482e350.png)

Le calcul est effectué modulo 256.

Pour programmer cette transformation je crée les fonctions suivantes :

1. La fonction ext_colonne() qui me permet d’extraire une colonne j d’une matrice donnée.

<pre><code>
def ext_colonne(matrice,j):
  l=len(matrice)
  colonne=np.zeros(l, int)
  for i in range(l):
    colonne[i]=matrice[i][j]
  return colonne
</pre></code>

La fonction prend en entrée une matrice et un entier j, elle extrait ensuite la jème colonne de « matrice » et la place dans un tableau nommé « colonne ». 
Finalement elle renvois « colonne ».

2. La fonction affect_colonne() qui me permet d’affecter un tableau d’entiers donné à une matrice donnée.

<pre><code>
 def affect_colonne (matrice,j,colonne) :
  l=len(matrice)
  for i in range(l):
    matrice[i][j]=colonne[i]
</pre></code>

La fonction prend en entrée une matrice, un entier « j » et un tableau « colonne » qu’elle affecte à la jème colonne de la matrice « matrice ».

3. La fonction produit() qui me permet de faire le produit matricielle modulo 256 de la matrice A par un vecteur.

<pre><code>
def produit(colonne):
  A=np.array([[2,3,1,1], [1,2,3,1], [1,1,2,3], [3,1,1,2]], type(int))
  res=np.zeros(4, type(int))
  for i in range(4):
    res[i]=0
    for j in range(4):
    res[i]=(res[i]+A[i][j]*colonne[j])%256
  return res
</pre></code>

La fonction prend comme paramètre un tableau « colonne » qu’elle multiplie par la matrice A et renvoie le vecteur résultant "res".

4. La transformation mixColumn()

<pre><code>
def mixColumns(boxes):
  boxes_mix=np.zeros((4,4), type(int))
  for k in range(4):
    colonne=ext_colonne(boxes,k)
    colonne=produit(colonne)
    affect_colonne(boxes_mix,k,colonne)
  return boxes_mix
</pre></code>

La fonction prend comme paramètre une matrice « boxes » et applique la fonction produit sur chaque colonne. La fonction renvoie la matrice « boxes_mix » 
qui contient la transformation qu’a subie la matrice boxes.

 ![La transformation mixColumns](https://user-images.githubusercontent.com/77968416/113205076-28976380-926e-11eb-98ff-95cbb666212e.png)
 
# Etape 4 La transformation addRoundKey

Cette opération consiste à faire un XOR entre la matrice w et la clé d’étage k, représentée également par une matrice de taille 4.
Le XOR se fait sous Python avec l’operateur ^.

<pre><code>
def addRoundKey(boxes,key):
  boxes_add=np.zeros((4,4), type(int))
  for i in range(4):
    for j in range(4):
      boxes_add[i][j]=boxes[i][j]^key[i][j]
  return boxes_add
</pre></code>

![La transformation addRoundKey](https://user-images.githubusercontent.com/77968416/113205512-a65b6f00-926e-11eb-8082-61b15eb0849a.png)

# Etape 5 : Diversification de la clé

On travaille avec une clé de 128 bits, donc 16 octets, qu’on va placer dans une matrice de taille 4. A partir de cette clé, on crée un code qui va générer 10 autres clés, ce qui donne au total 11 clés.
Dans cette étape je vais réutiliser les fonctions ext_colonne(), affect_colonne() et find_box. Et je crée de nouvelles fonctions décrites ci-dessous :

1. La fonction rotWord() qui permet de faire une rotation d’éléments d’une liste.

<pre><code>
def rotWord(liste):
  l=len(liste)
  rot=np.zeros(l, int)
  for i in range(l):
    rot[i]=liste[(i+1)%l]
  return rot
</pre></code>

La fonction prend comme paramètre une liste d’éléments « liste », fait le décalage d’un pas modulo le nombre d’éléments de la liste et revoie la nouvelle liste « rot ».

![la fonction rotWord](https://user-images.githubusercontent.com/77968416/113205821-fe927100-926e-11eb-815c-cdd9f44658db.png)

2. La fonction subBytes_column() qui étant donné une liste d’éléments, retourne leurs correspondances dans la table s_box.

<pre><code>
def subBytes_column(liste):
  l=len(liste)
  trans=np.zeros(l, int)
  for i in range(l):
    trans[i]=find_box(liste[i])
  return trans
</pre></code>

La fonction prend comme paramètre une liste d’éléments « liste », cherche la correspondance de chaque élément de la liste dans la table s_box via la fonction 
find_box() et retourne la liste transformée « trans ».

3. La fonction xorVecteur() qui permet de faire le XOR entre deux tableaux.

<pre><code>
def xorVecteur(v1,v2):
  l=len(v1)
  res=np.zeros(l, int)
  for i in range(l):
    res[i]=v1[i]^v2[i]
  return res
</pre></code>

La fonction prend en paramètre deux tableaux d’éléments «v1» et «v2», fait le XOR habituelle entre chaque deux éléments du même indice et retourne le tableau 
résultant « res ». Sous Python le XOR est effectué avec l’operateur ^.

![Exemple d'exécution de xorVecteur](https://user-images.githubusercontent.com/77968416/113206087-5204bf00-926f-11eb-8c3a-aeb1b8d8d4eb.png)

4. La fonction addFirst() qui permet de XORer le premier élément de la liste avec un élément donné.

<pre><code>
def addFirst(liste, k):
liste[0]=liste[0]^k
return liste
</pre></code>

La fonction prend en paramètres une liste d’éléments « liste » et un nombre « k », XOR le premier élément avec « k » et retourne « liste » après modification.

![Exemple d'exécution de addFirst](https://user-images.githubusercontent.com/77968416/113206741-1caca100-9270-11eb-83a4-222e9d255071.png)

5. La fonction column_multiple() qui permet de créer une nouvelle colonne d’une matrice donnée à partir des colonnes déjà existantes. Elle est réservée 
à la colonne dont l’indice est un multiple de 4.

<pre><code>
def column_multiple(matrice, j):
  rcon=np.array([0x1,0x2,0x4,0x8,0x10,
  0x20,0x40,0x80,0x1b,0x36])
  liste=subBytes_column(rotWord(ext_colonne(matrice,j-1)))
  v=xorVecteur(ext_colonne(matrice,j-4),liste)
  liste=addFirst(v,rcon[int(j/4)-1])
  affect_colonne(matrice,j,liste)
</pre></code>

La fonction column_multiple () prend en paramètre une matrice « matrice » et un indice « j ». elle applique à la (j-1)ème colonne de la matrice les fonctions : 
rotWord() et subBytes_column(), le résultat est contenu dans « liste ». Ensuite elle XOR « liste » avec la (j-4) ème colonne de « matrice », le résultat est 
contenue dans « v ». Puis elle applique la fonction addFirst() où elle XOR le premier élément de « v » avec le ((j/4)-1)ème élément de « rcon ». Finalement, 
cette nouvelle liste qui vient d’être créé est affectée à la jème colonne de « matrice ».

6. La fonction column_notMultiple() qui permet aussi de créer une nouvelle colonne d’une matrice donnée à partir des colonnes existantes, sauf qu’elle est 
réservée aux indices non divisibles par 4.

<pre><code>
def column_notMultiple(matrice, j):
  l=len(matrice)
  for i in range(l):
    matrice[i,j]=matrice[i,j-4]^matrice[i,j-1]
</pre></code>

La fonction prend en paramètre une matrice et un indice « j ». Une jème colonne est construite en faisant le XOR entre la (j-4)ème et la (j-1)ème 
colonne de « matrice ».

7. L’opération keyExpansion() qui permet de créer 10 clés à partir de la clé secrète.

<pre><code>
def keyExpansion(key):
  c=np.concatenate((toBoxes(key),
  np.zeros((4,4*10), int)), axis=1)
  for j in range(4,44):
    if j%4==0:
      column_multiple(c, j)
    else:
      column_notMultiple(c, j)
  return c
 </pre></code>
 
 La fonction prend en paramètre une liste d’éléments « key ». Elle crée une matrice « c » de taille 4x44 dont les quatre premières colonnes sont les éléments de 
 « key » et le reste c’est des zéros. Ensuite pour chaque colonne d’indice j>4 de la matrice « c » elle applique column_multiple() si 4 divise j et column_notMultiple() 
 sinon. Elle retourne la matrice « c » qui correspond aux 11 clés qui seront utiliser dans le processus du cryptage.

 
 ## Exemple de génération de clés avec keyExpansion()
 
Je considère la clé secrète suivante écrite en base hexadécimale :
 
k=[0x2b,0x7e,0x15,0x16,0x28,0xae,0xd2,0xa6,0xab,0xf7,0x15,0x88,0x09,0xcf,0x4f,0x3c]

L’ajout de 0x devant chaque octet fait comprendre à Python qu’on a des nombres en hexa.

J’exécute la commande M=keyExpansion(k) et j’obtiens une matrice M de 4 lignes et 4x11 colonnes où chaque quatre nouvelles colonnes créées correspondent à une 
clé d’étage. Les éléments de M sont des entiers entre 0 et 256, mais je crée une fonction toHexa() pour afficher les clés en hexa.
 
![a clé secrète k](https://user-images.githubusercontent.com/77968416/113207937-87121100-9271-11eb-889f-e89dd1bac3c9.png)
 
 ![les 10 clés d'étages](https://user-images.githubusercontent.com/77968416/113208026-9f822b80-9271-11eb-98f4-7cac8c42dde9.png)

 # Etape 6 : Algorithme AES et affichage du texte crypté
 
 Maintenant nous allons enchainer les opérations créées dans les étapes précédantes pour crypter un message clair.
Pour cela je crée d’abord la fonction crypt_aes() qui renvoie le message crypté dans une matrice de taille 4x4 et ensuite une fonction message_cryp() qui 
renvoie le message crypté sous forme d’une chaine de caractères en hexadécimal.

1. La fonction crypt_aes()

<pre><code>
def crypt_aes(message, key):
  w=toBoxes(toAscii(message))
  keys=keyExpansion(key)
  w=addRoundKey(w,keys[:,0:4])
  for i in range(1,10):
    w=mixColumns(shiftRows(subBytes(w)))
    w=addRoundKey(w,keys[:,i*4:(i+1)*4])
  w=shiftRows(subBytes(w))
  w=addRoundKey(w,keys[:,10*4:11*4])
  return w
</pre></code>

La fonction prend en paramètres le message clair « message » et la clé « key ». Elle commence par transformer le message de 16 octets en matrice carrée de taille 4 « w ».
L’algorithme de chiffrement comporte 11 étages (je travaille avec une clé de 128 bits) où chacun utilise une clé d’étage générée par keyExpansion(k) et placée dans
la matrice « keys » de taille 4x(4x11).

* Etage 0 : utilise la clé initiale « key » et appelle l’opération addRoundKey().
* Etages 1 jusqu’à 9 : Applique successivement les opérations subBytes(), shiftRows(), mixColumns() sur w ensuite addRoundKey() entre w et la clé d’étage récupérée 
à partir de la matrice keys.
* Etage 10 : Applique successivement subBytes() et shiftRows() sur w résultant des étages précédant, ensuite addRoundKey() entre w et la dernière clé de la matrice keys.

2. La fonction message_cryp()

<pre><code>
def message_cryp(message, key):
  liste=toliste(toHexa(crypt_aes(message, key)))
  decry="".join(liste)
  return decry
</pre></code>

La fonction prend en paramètres le message à crypter et la clé secrète « key ». elle appelle crypt_aes(), transforme les éléments de la matrice obtenue en hexa et la
récupère sous forme d’une liste de 16 éléments. Ensuite transforme cette liste en chaine de caractères et retourne cette dernière.

Je prends le massage clair du début m="JESERAISAVECVOUS" et je le crypte avec le chiffrement AES en utilisant la clé : 

k=[0x2b,0x7e,0x15,0x16,0x28,0xae,0xd2,0xa6,0xab,0xf7,0x15,0x88,0x09,0xcf,0x4f,0x3c]

![Message crypté avec AES](https://user-images.githubusercontent.com/77968416/113208929-b37a5d00-9272-11eb-863a-8eb4b59cc5b1.png)



