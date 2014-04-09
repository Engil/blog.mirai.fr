---
title: Les optimisations LLVM dans la pratique
tags: llvm, cervoise, optimisations
author: jpdeplaix
---

Venant d'ajouter un option d'optimisation dans [Cervoise](https://github.com/jpdeplaix/cervoise) via ce [commit](https://github.com/jpdeplaix/cervoise/commit/a3685b10c7fc6fb),
j'ai pu voir quelles étaient les optimisations faites par LLVM dans le cas d'un langage fonctionnel qui supprime ces types après compilation (voir [Type erasure](https://en.wikipedia.org/wiki/Type_erasure)) et qui n'a pas de types natifs.

Avant de voir cela, quelques mots tout d'abord sur l'API OCaml (officielle) pour LLVM qui est ici utilisée. Celle-ci propose, via son module [Llvm.PassManager](https://github.com/llvm-mirror/llvm/blob/master/bindings/ocaml/llvm/llvm.mli#L2419), deux manières fines d'optimiser les fonctions.
À savoir:

 * Soit pour tout un module
 * Soit pour certaines fonctions seulement

Il propose aussi par différents modules (localisés dans le dossier [transforms](https://github.com/llvm-mirror/llvm/tree/master/bindings/ocaml/transforms)), des moyens d'ajouter des optimisations, soit à la carte, soit avec un menu (correspondant aux options -O1, -O2, -O3 et -O4 de [Clang](https://fr.wikipedia.org/wiki/Clang)).

Maintenant une note sur la manière dont LLVM marche et optimise le code.
LLVM est un framework de compilation. Il est en particulier doté:

 * D'une machine virtuelle qui exécute du bytecode LLVM
 * D'un outil (llc) qui transforme le LLVM-IR (une sorte d'assembleur générique) en code natif ou en bytecode LLVM
 * Une librairie C++ servant à générer ce LLVM-IR
 * Et quelques bindings (OCaml et Python) pour cette librairie.
À noter aussi que LLVM contient un gros nombre de backends pour différentes architectures, ce qui lui permet d'être très portable.
Donc, dans cervoise, je génère ce LLVM-IR via le binding OCaml et le passe ensuite à llc.
L'étape d'optimisation dans LLVM est fait entre les deux phases. C'est une optimisation qui prend (virtuellement) en entrée du LLVM-IR et qui retourne du LLVM-IR optimisé. Ce n'est *pas* réalisé durant la compilation du LLVM-IR vers l'assembleur natif.

Tout cela dit, nous allons pouvoir voir, quelles sont les optimisations de LLVM dans le cas de Cervoise.

 * Il va faire de tout les appels de fonctions, des appels tails call (en rajoutant ```tail``` devant l'instruction ```call```)
 * Ensuite il va enlever les instructions qui ne servent à rien (grâce au [SSA](https://fr.wikipedia.org/wiki/Static_single_assignment_form)) [1].
 * Il va aussi faire de l'inlining là ou il peut.
 * Ajout d'un attribut ```nounwind``` à chaque fonction. Cet attribut indique que la fonction ne lancera pas d'exceptions ou ne fera pas de choses bizarres avec son control flow (si j'ai bien saisi).
 * Ajout d'un attribut ```nocapture``` aux paramétres non-utilisés des fonctions (pour permettre la propagation de l'information).

Pour étayer l'exemple, voici,

Le fichier Cervoise source:
```
datatype Unit = Unit : Unit
let id = λf:Unit -> Unit.f Unit
```

La différence du code produit sans optimisations et avec optimisation au niveau 4:
```diff
--- before      2014-04-06 23:49:22.788797037 +0200
+++ after       2014-04-06 23:49:24.084797025 +0200
@@ -7,45 +7,42 @@
  @Unit = global i8* undef
  @id = global i8* undef

-define void @__init() {
+; Function Attrs: nounwind
+define void @__init() #0 {
  entry:
    %malloccall = tail call i8* @malloc(i32 0)
    %env = bitcast i8* %malloccall to [0 x i8*]*
    %env_cast = getelementptr [0 x i8*]* %env, i64 0, i64 0
    %malloccall1 = tail call i8* @malloc(i32 ptrtoint ({ i32, i8** }* getelementptr ({ i32, i8** }* null, i32 1) to i32))
    %variant = bitcast i8* %malloccall1 to { i32, i8** }*
-  %variant_loaded = load { i32, i8** }* %variant
-  %variant_with_idx = insertvalue { i32, i8** } %variant_loaded, i32 0, 0
-  %variant_with_vals = insertvalue { i32, i8** } %variant_with_idx, i8** %env_cast, 1
+  %variant_with_vals = insertvalue { i32, i8** } { i32 0, i8** undef }, i8** %env_cast, 1
    store { i32, i8** } %variant_with_vals, { i32, i8** }* %variant
-  %cast_variant = bitcast { i32, i8** }* %variant to i8*
-  store i8* %cast_variant, i8** @Unit
+  store i8* %malloccall1, i8** @Unit
    %malloccall2 = tail call i8* @malloc(i32 ptrtoint (i1** getelementptr (i1** null, i32 1) to i32))
    %env3 = bitcast i8* %malloccall2 to [1 x i8*]*
    %env_cast4 = getelementptr [1 x i8*]* %env3, i64 0, i64 0
    %malloccall5 = tail call i8* @malloc(i32 trunc (i64 mul nuw (i64 ptrtoint (i1** getelementptr (i1** null, i32 1) to i64), i64 2) to i32))
    %closure = bitcast i8* %malloccall5 to { i8* (i8*, i8**)*, i8** }*
-  %closure_loaded = load { i8* (i8*, i8**)*, i8** }* %closure
-  %closure_insert_f = insertvalue { i8* (i8*, i8**)*, i8** } %closure_loaded, i8* (i8*, i8**)* @__lambda, 0
-  %closure_insert_env = insertvalue { i8* (i8*, i8**)*, i8** } %closure_insert_f, i8** %env_cast4, 1
+  %closure_insert_env = insertvalue { i8* (i8*, i8**)*, i8** } { i8* (i8*, i8**)* @__lambda, i8** undef }, i8** %env_cast4, 1
    store { i8* (i8*, i8**)*, i8** } %closure_insert_env, { i8* (i8*, i8**)*, i8** }* %closure
-  %closure_cast = bitcast { i8* (i8*, i8**)*, i8** }* %closure to i8*
-  store i8* %closure_cast, i8** @id
+  store i8* %malloccall5, i8** @id
    ret void
    }

- declare noalias i8* @malloc(i32)
+ ; Function Attrs: nounwind
+ declare noalias i8* @malloc(i32) #0

- define i8* @__lambda(i8*, i8**) {
+ define i8* @__lambda(i8*, i8** nocapture) {
   entry:
-  %gep_env = getelementptr i8** %1, i32 0
-  store i8* %0, i8** %gep_env
+  store i8* %0, i8** %1
    %extract_f_cast = bitcast i8* %0 to { i8* (i8*, i8**)*, i8** }*
    %exctract_f = load { i8* (i8*, i8**)*, i8** }* %extract_f_cast
    %f = extractvalue { i8* (i8*, i8**)*, i8** } %exctract_f, 0
    %env = extractvalue { i8* (i8*, i8**)*, i8** } %exctract_f, 1
    %glob_extract = load i8** @Unit
-  %tmp = call i8* %f(i8* %glob_extract, i8** %env)
+  %tmp = tail call i8* %f(i8* %glob_extract, i8** %env)
    ret i8* %tmp
    }

+ attributes #0 = { nounwind }
+
```

Références:
 http://llvm.org/docs/LangRef.html
