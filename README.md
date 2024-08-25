# Ensimag - Printemps 2024 - Projet Logiciel en C
### Sujet : Encodeur JPEG

![Alt text](./shaun_masque.jpg "HEADER struct")


### Auteurs : Des enseignants actuels et antérieurs du projet C

## 1. Introduction
Le format JPEG est l'un des formats les plus répandus en matière d'image numérique. Il est en particulier utilisé comme format de compression par la plupart des appareils photo numériques, étant donné que le coût de calcul et la qualité sont acceptables, pour une taille d'image résultante petite.

Le codec¹ JPEG est tout d'abord présenté en détail au chapitre 2 et illustré sur un exemple en annexe A. Le chapitre 3 présente le mode progressif, à attaquer une fois votre encodeur baseline sequential terminé. Le chapitre 4 présente les spécifications, les modules et outils fournis, et quelques conseils importants d'organisation pour ce projet. Finalement, le chapitre 5 formalise le travail à rendre et les détails de l'évaluation.

L'objectif de ce projet est de réaliser en langage C un encodeur qui convertit les images au format brut (PPM) en un format compressé (JPEG). Il est nécessaire d'avoir une bonne compréhension du codec, qui reprend notamment des notions vues dans les enseignements Théorie de l'information et Bases de la Programmation Impérative. Mais l'essentiel de votre travail sera bien évidemment de concevoir et d'implémenter votre encodeur en langage C.

Bon courage à tou(te)s, et bienvenue dans le monde merveilleux du JPEG ! Enjoy !

## 2. Le JPEG pour les petits : mode séquentiel
Le JPEG (Joint Photographic Experts Group) est un comité de standardisation pour la compression d'image dont le nom a été détourné pour désigner une norme en particulier, la norme JPEG, que l'on devrait en fait appeler ISO/IEC IS 10918-1 | ITU-T Recommendation T.81.¹

Cette norme spécifie plusieurs alternatives pour la compression des images en imposant des contraintes uniquement sur les algorithmes et les formats du décodage. Notez que c'est très souvent le cas pour le codage source (ou compression en langage courant), car les choix pris lors de l'encodage garantissent la qualité de la compression. La norme laisse donc la réalisation de l'encodage libre d'évoluer. Pour une image, la qualité de compression est évaluée par la réduction obtenue sur la taille de l'image, mais également par son impact sur la perception qu'en a l'œil humain. Par exemple, l'œil est plus sensible aux changements de luminosité qu'aux changements de couleur. On préférera donc compresser les changements de couleur que les changements de luminosité, même si cette dernière pourrait permettre de gagner encore plus en taille. C'est l'une des propriétés exploitées par la norme JPEG.

Parmi les choix proposés par la norme, on trouve des algorithmes de compression avec ou sans perte (une compression avec pertes signifie que l'image décompressée n'est pas strictement identique à l'image d'origine) et différentes options d'affichage (séquentiel, l'image s'affiche en une passe pixel par pixel, ou progressif, l'image s'affiche en plusieurs passes en incrustant progressivement les détails, ce qui permet d'avoir rapidement un aperçu, quitte à attendre pour avoir l'image entière).

Dans son ensemble, il s'agit d'une norme plutôt complexe qui doit sa démocratisation à un format d'échange, le JFIF (JPEG File Interchange Format). En ne proposant au départ que le minimum essentiel pour le support de la norme, ce format s'est rapidement imposé, notamment sur Internet, amenant à la norme le succès qu'on lui connaît aujourd'hui. D'ailleurs, le format d'échange JFIF est également confondu avec la norme JPEG. Ainsi, un fichier possédant une extension .jpg ou .jpeg est en fait un fichier au format JFIF respectant la norme JPEG. Évidement, il existe d'autres formats d'échange supportant la norme JPEG comme les formats TIFF ou EXIF. La norme de compression JPEG peut aussi être utilisée pour encoder de la vidéo, dans un format appelé Motion-JPEG. Dans ce format, les images sont toutes enregistrées à la suite dans un flux. Cette stratégie permet d'éviter certains artefacts liés à la compression inter-images dans des formats types MPEG.

L'encodeur JPEG demandé dans ce projet doit supporter le mode dit "baseline sequential" (compression avec pertes, séquentiel, Huffman). Ce mode est utilisé dans le format JFIF, et il est décrit dans la suite de ce document.

### 2.1 Principe général du codec JPEG
Cette section détaille les étapes successives mises en œuvre lors de l'encodage, c'est-à-dire la conversion d'une image au format PPM vers une image au format JPEG.

Tout d'abord, l'image est partitionnée en macroblocs ou MCU pour Minimum Coded Unit. La plupart du temps, les MCUs sont de taille 8x8, 16x8, 8x16 ou 16x16 pixels selon le facteur d'échantillonnage (voir section sous-échantillonnage). Chaque MCU est ensuite réorganisée en un ou plusieurs blocs de taille 8x8 pixels.

La suite porte sur la compression d'un bloc 8x8. Chaque bloc est traduit dans le domaine fréquentiel par transformation en cosinus discrète (DCT). Le résultat de ce traitement, appelé bloc fréquentiel, est encore un bloc 8x8 mais dont les coordonnées ne sont plus des pixels, c'est-à-dire des positions du domaine spatial, mais des amplitudes à des fréquences données. On y distingue un coefficient continu DC aux coordonnées (0,0) et 63 coefficients fréquentiels AC.² Les plus hautes fréquences se situent autour de la case (7,7).

L'œil étant moins sensible aux hautes fréquences, il est plus facile de les filtrer avec cette représentation fréquentielle. Cette étape de filtrage, dite de quantification, détruit de l'information pour permettre d'améliorer la compression, au détriment de la qualité de l'image (d'où l'importance du choix du filtrage). Elle est réalisée bloc par bloc à l'aide d'un filtre de quantification, qui peut être générique ou spécifique à chaque image. Le bloc fréquentiel filtré est ensuite parcouru en zig-zag (ZZ) afin de transformer le bloc en un vecteur de 64x1 fréquences avec les hautes fréquences en fin. De la sorte, on obtient statistiquement plus de 0 en fin de vecteur.

Ce bloc vectorisé est alors compressé en utilisant successivement plusieurs codages sans perte : d'abord un codage RLE pour exploiter les répétitions de 0, un codage des différences plutôt que des valeurs, puis un codage entropique³ dit de Huffman qui utilise un dictionnaire spécifique à l'image en cours de traitement.

Les étapes ci-dessus sont appliquées à tous les blocs composant les MCUs de l'image. La concaténation de ces vecteurs compressés forme un flux de bits (bitstream) qui est stocké dans le fichier JPEG. Ces données brutes sont séparées par des marqueurs qui précisent la longueur et le contenu des données associées. D'autres informations générales (taille de l'image, etc.) sont aussi ajoutées dans l'en-tête du fichier. Le format et les marqueurs sont spécifiés dans l'annexe B.

Les opérations de codage/décodage sont résumées sur la figure ci-dessous, puis détaillées dans les sections suivantes (dans le sens du décodage). L'annexe A fournit un exemple numérique du codage d'une MCU.

![Alt text](./codec.png "HEADER struct")

## 2.2 Représentation des données

Il existe plusieurs manières de représenter une image. Une image numérique est en fait un tableau de pixels, chaque pixel ayant une couleur distincte. Dans le domaine spatial, le codec utilise deux types de représentation de l'image.

Le format RGB, le plus courant, est le format utilisé en entrée. Il représente chaque couleur de pixel en donnant la proportion de trois couleurs primitives: le rouge (R), le vert (G), et le bleu (B). Une information de transparence alpha (A) peut également être fournie (on parle alors de ARGB), mais elle ne sera pas utilisée dans ce projet. Le format RGB est le format utilisé en amont et en aval de l'encodeur.

Un deuxième format, appelé YCbCr, utilise une autre stratégie de représentation, en trois composantes: une luminance dite Y, une différence de chrominance bleue dite Cb, et une différence de chrominance rouge dite Cr. Le format YCbCr est le format utilisé en interne par la norme JPEG. Une confusion est souvent réalisée entre le format YCbCr et le format YUV.4

La stratégie de représentation YCbCr est plus efficace que le RGB (Red, Green, Blue) classique, car d'une part les différences sont codées sur moins de bits que les valeurs et d'autre part elle permet des approximations (ou de la perte) sur la chrominance à laquelle l'œil humain est moins sensible.

## 2.3 Découpage de l'image en MCUs
La première étape consiste à découper l'image en MCUs.

La taille de l'image n'étant par forcément un multiple de la taille des MCUs, le découpage en MCUs peut "déborder" à droite et en bas. A l'encodage, la norme recommande de compléter les MCUs en dupliquant la dernière colonne (respectivement ligne) contenue dans l'image dans les colonnes (respectivement lignes) en trop.

![Alt text](./mcu.png "HEADER struct")

La figure ci-dessus donne un exemple d’une image 46 × 35 (fond gris) à compresser sans sous-échantillonnage, soit avec des MCUs composées d’un seul bloc 8 × 8. Pour couvrir l’image en entier, 6 × 5 MCUs sont nécessaires. Les MCUs les plus à droite et en bas (les 6e, 12e, 18e, puis 24e à 30e) devront être complétées lors de la compression.

## 2.4 Conversion RGB vers YCbCr

La conversion RGB vers YCbCr s'effectue pixel par pixel à partir des trois composantes de chaque MCU (éventuellement composée de plusieurs blocs, voir ensuite). Ainsi, pour chaque pixel dans l'espace RGB, on effectue le calcul suivant pour obtenir la valeur du pixel dans l'espace YCbCr :

![Alt text](./mat.png "HEADER struct")

Ces calculs incluent des opérations arithmétiques sur des valeurs flottantes et signées. Sachant que les valeurs RGB sont forcément comprises entre 0 et 255, il conviendra de choisir le plus petit type de données entier permettant d'encoder toute la plage de valeurs possibles pour Y, Cb et Cr lors de l'écriture de votre encodeur.

## 2.5 Compression des MCUs
##   ----------------------
Dans le processus de compression, le JPEG peut exploiter la faible sensibilité de l'œil humain aux composantes de chrominance pour réaliser un sous-échantillonnage (subsampling) de l'image.

Le sous-échantillonnage est une technique de compression qui consiste en une diminution du nombre de valeurs, appelées échantillons, pour certaines composantes de l'image. Pour prendre un exemple, imaginons qu'on travaille sur une image couleur YCbCr partitionnée en MCUs de 2x2 blocs de 8x8 pixels chacun, pour un total de 256 pixels.

Ces 256 pixels ont chacun un échantillon pour chaque composante, le stockage nécessiterait donc 256x3 = 768 échantillons. On ne sous-échantillonne jamais la composante de luminance de l'image. En effet, l'œil humain est extrêmement sensible à cette information, et une modification impacterait trop la qualité perçue de l'image. Cependant, comme on l'a dit, la chrominance contient moins d'information. On pourrait donc décider que pour 2 pixels de l'image consécutifs horizontalement, un seul échantillon par composante de chrominance suffit. Il faudrait seulement alors 256 + 128 + 128 = 512 échantillons pour représenter toutes les composantes, ce qui réduit notablement la place occupée! Si on applique le même raisonnement sur les pixels de l'image consécutifs verticalement, on se retrouve à associer à 4 pixels un seul échantillon par chrominance, et on tombe à une occupation mémoire de 256 + 64 + 64 = 384 échantillons.
    **2.5.1 Sous-échantillonnage de l'image**
    Dans ce document, nous utiliserons une notation directement en lien avec les valeurs présentes dans les sections JPEG de l'en-tête, décrites en annexe B, qui déterminent le facteurs d'échantillonnages (sampling factors). Ces valeurs sont identiques partout dans une image. En pratique, on utilisera la notation \(h\times v)\ de la forme :

![Alt text](./mat1.png "HEADER struct")

    où hi et Vi représentent le nombre de blocs horizontaux et verticaux pour la composante i. Comme Y n'est jamais compressé, le facteur d'échantillonnage de Y donne les dimensions de la MCU en nombre de blocs. Les sous-échantillonnages les plus courants sont décrits ci-dessous. Votre encodeur devra supporter au moins ces trois combinaisons, mais nous vous encourageons à gérer tous les cas !

![Alt text](./sousech.png "HEADER struct")

    La figure ci-dessus illustre les composantes Y, Cb et Cr avec et sans sous-échantillonnage, pour une MCU de 2 × 2 blocs.

    **2.5.2 Ordre d'écriture des blocs**
    


## 2.6 Transformée en cosinus discrète (DCT)



## 2.7 Quantification et zig-zag
    **2.7.1 Zig-zag**
    **2.7.2 Quantification**
    **2.7.3 Ordre des opérations**


## 2.8 Compression d'un bloc fréquentiel
    **2.8.1 Le codage de Huffman**
    **2.8.2 Coefficient continu: DPCM, magnitude et arbre DC**
        **2.8.2.1 Représentation par magnitude**
        **2.8.2.2 Encodage dans le flux de bits: Huffman**
    **2.8.3 Arbres AC et codage RLE**
        **2.8.3.1 Codage des coefficients AC**
        **2.8.3.2 Encodage dans le flux**


## 2.9 Ecriture dans le flux JPEG
    **2.9.1 Structure d'un fichier JPEG**
    **2.9.2 Byte stuffing**
    





![Alt text](./ordo.png "HEADER struct")

![Alt text](./ordo1c.png "group_MCU / group_RGB struct")

![Alt text](./zig_then_zag.jpg "Architecture")

![Alt text](./huff.png "Architecture")

