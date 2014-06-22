Modèle de données physique dans **Cassandra**
============================================

 Voici le premier article d'une longue série sur la modélisation de données dans **Apache Cassandra** avec **CQL3**
 

# Qu'est ce que **Cassandra** ?
 
Avant d'entrer dans le vif du sujet, une petite présentation de **Apache Cassandra** s'impose pour ceux qui ne la connaissent pas.

## Historique
 
**Apache Cassandra** est une base de données de la famille **NoSQL** très en vogue. Elle se classe parmi les bases _orientées colonnes_ tout comme
*HBase*, **Apache Accumulo**, **Big Table**. Cette base a été développée à l'origine par des ingénieurs de **Facebook** pour leurs besoins en
interne avant d'être mis à la disposition du grand public en _open-souce_. 

>Pour la petite histoire, dans le code source de **Cassandra** on 
retrouve encore des classes préfixées avec **FB** (comme **Facebook**) qui rappelle cette origine
 
## Origine du nom      
 
Une petite anecdote veut que le nom de **Cassandra** ait été choisie par rapport à **Oracle**. D'après la mythologie grecque, **Cassandra** est un oracle
maudit qui prédisait du malheur mais dont personne ne voulait croire les prédictions, jusqu'au jour où ... C'était un clin d'oeil très explicite à la base
de données **Oracle** faite à l'époque par les ingénieurs de **Facebook**.
 
## Architecture
 
Sans rentrer dans les détails, sachez que l'architecture **Cassandra** (qu'on va appeler désormais en C* pour plus de concision) s'inspire énormément
du papier de recherche **Big Table** de **Google** ainsi que de l'architecture **Dynamo** d' **Amazon**. Le moteur de stockage de C* dérive
directement de Big Table alors que sa couche de distribution de données s'inspire de l'architecture de **Dynamo*

## Cluster

Quand on parle de C*, on parle souvent de **cluster**. Un cluster est un regroupement de plusieurs **noeuds** (serveur physique C*) qui communiquent
entre eux pour la gestion de données.
  
  ![Cluster à 5 noeuds](Cluster à 5 noeuds.png)
     

## Modèle physique de données
### Keyspace et tables
 
Dans un cluster Cassandra, on trouve des **tables** et des **keyspaces**. Un **keyspace** peut-être vu comme une base. En effet dans un modèle 
multi-tenant on sépare les données de chaque partie dans des **keyspaces**.
            
A l'intérieur de chaque **keyspace**, on trouve des **tables**. Une **table** dans C* a la même signification qu'une table dans le monde SQL, avec 
des lignes et des colonnes.

> la terminologie de **table** est apparue avec **CQL3**. L'ancien terme pour désigner une **table**  est **column family**. Dans la suite de l'article
nous utiliserons indistinctement chacun de ces termes

### Partitions, colonnes et cellules             
       
Dans une **table**, les données sont stockées sous forme de ligne et de colonnes comme suit:
        
NB: # = clé        

| #partition | Colonne1| Colonne2| Colonne3| Colonne4| Colonne5|etc|
| ---------- | --------| ------- | ------- | ------- | ------- |---|         
| Partition1| #col1/Cellule1 | #col2/Cellule2 | #col3/Cellule3 | | | |         
| Partition2| #col1/Cellule1 | #col2/Cellule2 | | | | | 
| Partition3| #col1/Cellule1 | #col2/Cellule2 | #col3/Cellule3 | #col4/Cellule4 | #col5/Cellule5 |...|
| Partition4| #col1/Cellule1 | | | | | |

   
           
Les données sont disposées sur des lignes (qu'on appelle **partitions**) et au sein de chaque partition on trouve une série de **colonnes** avec __#colonne/Cellule__
qu'on peut assimiler à une série de **clé/valeur**.   

On peut remarquer qu'un **partition** peut contenir de **1 à N colonnes**, la limite physique de **N** étant de 2 milliards (2.10<sup>9</sup>) ce qui nous laisse
beaucoup de marge. Une différence avec une base de données SQL classique est qu'ici C* ne réserve pas d'espace en avance pour chaque colonne. La création de
colonne se fera lorsqu'on insérera de nouvelles données, dynamiquement.

Pour résumer, une **table** peut être visualisée conceptuellement comme ensemble de 

 <pre>
    &lt;Clé de partition, &lt;Clé de colonne,Cellule&gt;&gt;
 </pre> 
 
**A noter que les colonnes sont triées par leur clé de colonne. Nous verrons par la suite que cette fonctionnalité est cruciale pour la modélisation de données dans Cassandra**
 
## Distribution de données
### Concept

Dans un cluster, les partitions dans une table sont réparties entre plusieurs noeuds. Il y a 2 façons de répartir les données:
 
 1)  de manière ordonnée, chaque noeud prend en charge une plage de **clé de partition** triée par ordre croissant.
 
 Exemple: si la clé de partition est le nom de famille, les noms commençant entre A et D se répartissent sur le noeud 1, entre E et L sur le noeud2 etc...
 
 ![Distribution ordonnée](Distribution ordonnée des données.png)
  
 2)  de manière aléatoire, chaque noeud prend en charge une plage de **clé de partition** distribuée de manière aléatoire.  
 

Une répartition ordonnée a l'avantage de conserver l'ordre par contre on se retrouvera avec des noeuds non équilibrés. En effet il est fort à parier qu'il y 
aura beaucoup plus de noms de famille commençant par __F-J__ que __V-Z__ du coup certains noeuds contiendront plus de données que d'autres.

 
Le choix du type de partitionnement se fait au niveau du **keyspace**. Pour un partitionnement ordonné, il faut utiliser le `BytesOrderedPartitioner` et `RandomPartitioner` 
pour un partitionnement aléatoire. Depuis C* version 2, on conseille l'utilisation du `Murmur3Partitioner`, plus performant que le `RandomPartitioner` basé sur 
un simple hash **MD5**.
 
Dans tous les cas, il est vivement conseillé d'utiliser un partitionnement aléatoire pour éviter les hot-spots dans le cluster et une meilleure répartition de la charge.
Les cas où l'utilisation du partitionnement ordonné est approprié sont très rares.


### Les tokens

L'un des moyens simples pour bien répartir les données sur tout le cluster c'est d'utiliser une fonction de hachage très dispersif. En pratique, 
à chaque opération de modification ou de lecture de données, le client fournit une clé de partition (#partition) à C*. Cette clé passe d'abord par une fonction
de hachage et le résultat est ce qu'on appelle un **token**. En choisissant bien la fonction de hachage, on peut faire en sorte qu'à 2 valeurs de #partition assez proche, 
les tokens produits sont très différents:


<pre>
 <strong>hash</strong>(#partition1) = token1
 
 <strong>hash</strong>(#partition2) = token2
</pre>

Si l'on prend le hash **MD5**, la valeur du token se situe dans l'intervalle [0 .. 2<sup>127</sup>-1]. On répartit uniformément cette plage de 
valeur de token entre les différents noeuds du cluster. Ainsi, pour un cluster de 5 noeuds, on aura par exemple:
 
* le noeud 1 est responsable de la plage ]0 ..  (2<sup>127</sup>-1)/5]  
* le noeud 2 est responsable de la plage ](2<sup>127</sup>-1)/5 ..  (2<sup>127</sup>-1)*2/5]  
* le noeud 3 est responsable de la plage ](2<sup>127</sup>-1)*2/5 ..  (2<sup>127</sup>-1)*3/5]  
* le noeud 4 est responsable de la plage ](2<sup>127</sup>-1)*3/5 ..  (2<sup>127</sup>-1)*4/5]  
* le noeud 5 est responsable de la plage ](2<sup>127</sup>-1)*4/5 ..  (2<sup>127</sup>-1)]  
 
![Distribution aléatoire](Distribution aléatoire des données.png)
   
 

## Structure d'une table
 
Maintenant, nous allons voir comment une table est représentée dans le moteur de stockage (storage engine) de C*.
 
Il faut savoir d'abord que les tables sont très fortement typés. Par type, on entend le type de:
 
 1. la clé de partition (#partition)
 2. la clé de colonne (#col)
 3. la celulle
 
Lors de la création d'une table (column family), on doit définir ces 3 types qui ne peuvent plus être modifiés par la suite.
 
### Les types

De base, C* supportent les types suivants:
 
 * bytes
 * boolean
 * composite
 * **counter**
 * date (bientôt déprécié)
 * decimal
 * double
 * float
 * inetAddress
 * int32 (entier de taille fixe)
 * integer (entier de taille variable)
 * long
 * uuid
 * timestamp
 * timeuuid (uuid de type 1)

Ci-dessous le script de création d'une table **user** avec l'API Thrift:
 
 <pre>
    create column family user 
      with key_validation_class = LongType 
      and comparator = UTF8Type 
      and default_validation_class = UTF8Type        
 </pre>
 
 * `key_validation_class` corresponds au type de la #partition
 * `comparator` corresponds au type de la #col
 * `default_validation_class` corresponds au type de la cellule.
   
Ici, la #partition est l'userId représenté par un Long ( _LongType_ ). Les clés de colonnes sont codés en String ( _UTF8Type_ ) et correspondent au nom de chaque propriété d'une personne.
Le type de la cellule est du String ( _UTF8Type_ ) pour stocker les données sous forme textuel. 
 
Voici un contenu possible de la table `user`:
 
<pre>
 -------------------
 RowKey: 10
 => (name=age, value=32, timestamp=1403441029826000)
 => (name=nom, value=MARTIN, timestamp=1403441024052000)
 => (name=prenom, value=Jean, timestamp=1403441027825000)
 -------------------
 RowKey: 11
 => (name=age, value=26, timestamp=1403441047501000)
 => (name=nom, value=DUCROS, timestamp=1403441034463000)
 => (name=prenom, value=Elise, timestamp=1403441039578000)
</pre>
  
On remarquera que les propriétés (nom, prénom, age) sont triés alphabétiquement. Ceci est dû au fait que C* trie naturellement les #col dans l'ordre naturel du type,
qui est ici du String.
  
### Abstraction

D'une manière générale, une table dans C* peut être assimilée à une structure de données **`Map<#partition,SortedMap<#col,cellule>>`**

On retrouve le tri des #col avec la **SortedMap** (Map triée). La Map externe n'est pas triée si on utilise un partitionnement aléatoire qui disperse les #partition
sur tous les noeuds.

Si on avait choisie un partitionnement ordonné, une table serait assimilable à  **`SortedMap<#partition,SortedMap<#col,cellule>>`**

## Les requêtes
### L'API Thrift

### Abstraction

### Exemples
 
 
## Le type **composite**
### Définition 
 
### Application


  
  
 
 
 
 
