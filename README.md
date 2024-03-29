# Simulation de sable qui tombe

Le but de ce projet est de créer une fenêtre dans laquelle on peut faire
apparaitre des grains de sable en cliquant avec la souris. Les grains de sables
doivent tomber de manière « réaliste » et « glisser » sur les autres si l'angle
le permet.

## 1) Afficher une fenêtre

Il y a énormément de manière de créer des fenêtres quelque soit le langage.
Pour ce projet nous allons utiliser `minifb` qui est assez simple et nous laisse
envoyer un tableau de pixel a afficher.

### Ajouter une librairie au projet

```bash
cargo add minifb
```

### Ouvrir une fenêtre vide

On va créer un main tout simple avec le contenu du [README de `minifb`](https://github.com/emoon/rust_minifb):
```rust
use minifb::{Key, Window, WindowOptions};

const WIDTH: usize = 640;
const HEIGHT: usize = 360;

fn main() {
    let mut buffer: Vec<u32> = vec![0; WIDTH * HEIGHT];

    let mut window = Window::new(
        "Test - ESC to exit",
        WIDTH,
        HEIGHT,
        WindowOptions::default(),
    )
    .unwrap_or_else(|e| {
        panic!("{}", e);
    });

    // Limit to max ~60 fps update rate
    window.limit_update_rate(Some(std::time::Duration::from_micros(16600)));

    while window.is_open() && !window.is_key_down(Key::Escape) {
        for i in buffer.iter_mut() {
            *i = 0; // write something more funny here!
        }

        // We unwrap here as we want this code to exit if it fails. Real applications may want to handle this in a different way
        window
            .update_with_buffer(&buffer, WIDTH, HEIGHT)
            .unwrap();
    }
}
```

En lançant ce code une fenêtre noire devrait s'ouvrir. On va voir ensemble le but de chaque partie de ce programme :

#### Mise en place de la résolution de la fenêtre

```rust
const WIDTH: usize = 640;
const HEIGHT: usize = 360;
```

Ces deux lignes définissent la taille de la fenêtre, `640` pixels de largeur et `360` pixels de hauteur.

#### Création du buffer de la fenêtre

```rust
    let mut buffer: Vec<u32> = vec![0; WIDTH * HEIGHT];
```

Cette ligne donne beaucoup d'informations très importante sur le fonctionnement du programmes.
Elle initialise un buffer **mutable** qui représente l'image a afficher dans la fenêtre.
Si le vecteur est mutable c'est parce qu'on va pouvoir mettre à jour l'image à chaque fois qu'un grain de sable bouge.

##### Des images en 1D ou en 2D ??

Ensuite, il y a quelque chose qui devrait te sembler bizarre c'est que le type est un simple `Vec<_>` plutôt qu'un `Vec<Vec<_>>`.
Ce qui veut dire que l'ont va représenter une image en 2D avec un vecteur en 1D:
Par exemple pour représenter cette image:
```
.\o/.
..|..
./.\.
```

Naivement en rust on pourrait vouloir écrire :
```rust
vec![
  vec!['.', '\', 'o', '/', '.'], 
  vec!['.', '.', '|', '.', '.'], 
  vec!['.', '/', '.', '\', '.'], 
  ],
]
```

Mais pour économiser de la mémoire et rendre l'affichage plus rapide `minifb` demande à recevoir l'image formattée de cette manière :

```rust
vec!['.', '\', 'o', '/', '.', '.', '.', '|', '.', '.', '.', '/', '.', '\', '.']
```

Ou avec des retours à la ligne plus sympa :

```rust
vec![
  '.', '\', 'o', '/', '.', 
  '.', '.', '|', '.', '.', 
  '.', '/', '.', '\', '.', 
]
```

Ensuite, pour comprendre les « délimitations » de chaque ligne alors que tout est contigue ce sera a nous d'indiquer à `minifb` au moment d'afficher une nouvelle image la `WIDTH` et la `HEIGHT`.
Lui de son côté il pourra se débrouiller pour re découper le vecteur en une dimension en un vecteur en deux dimensions.
Par contre il est très important que le vecteur fasse contienne exactement autant de valeur qu'il y a de pixels.
C'est à dire `WIDTH * HEIGHT` pixels.
On retombe donc sur la taille initiale déclarée du buffer :
```rust
    let mut buffer: Vec<u32> = vec![0; WIDTH * HEIGHT];
    //                                 ^^^^^^^^^^^^^^
    //              Ici le vecteur contient bien WIDTH * HEIGHT éléments
```

##### Mais c'est quoi un pixel

Maintenant qu'on sait où est-ce qu'il faut écrire dans le vecteur pour afficher un pixel.
Il faut savoir **quoi** écrire dans le vecteur.
Dans le type déclaré sur l'exemple de `minifb` ils utilisent des `u32`.

```rust
    let mut buffer: Vec<u32> = vec![0; WIDTH * HEIGHT];
    //                  ^^^         ^
    //             Des `u32` qui valent zéro ???
```

C'est parce que ce `u32` représente en réalité 4 `u8` ensemble :
1. Rien
2. Le rouge
3. Le vert
4. Le bleu

En écrivant `0` dans le vecteur, il n'y a ni rouge, ni vert, ni bleu donc le pixel est noir.
Si on écrit `u32::MAX` a la place alors on aura le maximum de rouge, vert et bleu et ça fera un pixel blanc.

/!\ Attention, si tu testes tu dois le modifier a la fois à la ligne `30`
avec la valeur par défaut et `47` où les valeurs sont mises à jour pour chaque
images.

Voilà quelques exemple de couleurs en hexadécimal :
```rust
0x00_00_00_00 // Pareil que `0`, un pixel noir
0x00_ff_00_00 // Du rouge
0x00_00_ff_00 // Du vert
0x00_00_00_ff // Du bleu
0x00_12_34_56 // Un mélange de couleur
0x00_ff_ff_ff // Du blanc
0xff_ff_ff_ff // Du blanc aussi, pareil que `u32::MAX`
// Et on remarque que les deux premières valeur ne servent effectivement jamais a rien. En général ça sert a gérer la transparence, mais dans minifb ça ne sert a rien.
```

#### Création de la fenêtre


```rust
    let mut window = Window::new(
        "Test - ESC to exit", // le nom de la fenêtre
        WIDTH, // Ici on spécifie la taille initiale de la fenêtre, ça pourra changer dans le futur
        HEIGHT,
        WindowOptions::default(), // On peut spécifier des options a l'ouverture de la fenêtre mais on ne vas pas le faire
    )
    .unwrap_or_else(|e| { // Si ça ne fonctionne pas, on plante
        panic!("{}", e);
    });
```

#### On fixe le nombre maximul d'image par seconde

Dans le cas d'un programme qui s'exécute très vite on ne veut pas que la fenêtre
se mette a jour des millions de fois par seconde et fasse ramer tout l'ordi
donc en général on fixe une limite au nombre d'image que l'on peut afficher
par seconde pour être sûr qu'on ne fasse pas de travail inutile.


```rust
    // Limit to max ~60 fps update rate
    window.limit_update_rate(Some(std::time::Duration::from_micros(16600)));
```

Ici, ils indiquent qu'on peut afficher une image toutes les 16_600 micro secondes.
[Ça correspond a 60 images pour secondes](https://numbat.dev/?q=1s+%2F+60+-%3E+us%E2%8F%8E).

Une manière plus lisible de l'écrire en rust serait comme ça :
```rust
    // Limit to max ~60 fps update rate
    window.limit_update_rate(Some(std::time::Duration::from_secs(1) / 60));
```

Souvent on conseille de faire entre 30 images par seconde, ou le maximum que
l'écran peut faire. Souvent 60, mais des fois 120 ou 240 images par secondes.
Dans notre cas, 60 images par seconde c'est bien.

#### L'exécution du programme

Jusqu'à présent tout ce qu'on a fait c'est se préparer a exécuter un programme avec une fenêtre. Mais il ne s'est encore rien passé.
C'est ici que tout se passe :

```rust
    while window.is_open() && !window.is_key_down(Key::Escape) {
        for i in buffer.iter_mut() {
            *i = 0; // write something more funny here!
        }

        // We unwrap here as we want this code to exit if it fails. Real applications may want to handle this in a different way
        window
            .update_with_buffer(&buffer, WIDTH, HEIGHT)
            .unwrap();
    }
```

##### La boucle principale

Cette boucle `while` est vraiment la boucle principale d'exécution du programme,
c'est ici que tout se passe et lorsqu'on sort de la boucle alors le programme
s'arrête.
```rust
    while window.is_open() && !window.is_key_down(Key::Escape) {
```

Il y a deux conditions d'arrêts ici, il faut que :
- `window.is_open()` - La fenêtre soit encore ouverte (i.e.: l'utilisateur n'as pas cliqué sur la croix rouge en haut a gauche)
- `&& !window.is_key_down(Key::Escape)` - Et que l'utilisateur n'ait pas appuyé sur la touche `echap`

##### La mise à jour du buffer
```rust
        for i in buffer.iter_mut() {
            *i = 0; // write something more funny here!
        }
```

Pour cet exemple on se balade simplement sur toutes les valeurs du buffer et on les mets à zéro.

##### La mise à jour de la fenêtre

Lorsqu'on a mis à jour le buffer il ne s'est rien passé, c'est seulement en donnant le buffer à `minifb` qu'il mettra à jour la fenêtre :

```rust
        // We unwrap here as we want this code to exit if it fails. Real applications may want to handle this in a different way
        window
            .update_with_buffer(&buffer, WIDTH, HEIGHT)
            .unwrap();
```

Et comme on l'as vu plus tôt, `minifb` ne sait pas comment afficher le `buffer` tout seul et il faut qu'on lui re-précise les dimensions de la fenêtre avec la `WIDTH` et la `HEIGHT`.


## 2) Simplifier la gestion des couleurs

On a vu plus tôt comment écrire une couleur au format `u32` mais en l'état ce n'est pas très pratique a utiliser.
Le premier exercice sera donc de créer une fonction qui vas nous simplifier la vie dans le reste du projet dont voici la signature :
```rust
pub fn rgb(red: u8, green: u8, blue: u8) -> u32;
```

Je vais te fournir une suite de tests que tu devras copier coller à la fin de ton fichier et que tu pourras lancer avec `cargo test` :
```rust
#[cfg(test)]
mod test {
  use super::*;
  
  #[test]
  fn test_rgb() {
    assert_eq!(rgb(0, 0, 0), 0x00_00_00_00);
    assert_eq!(rgb(255, 255, 255), 0x00_ff_ff_ff);
    assert_eq!(rgb(0x12, 0x34, 0x56), 0x00_12_34_56);
  }
}
```

### `Left shift` et `OR`

Pour la première implémentation je veux que tu utilises l'opérateur [`left shift`](https://doc.rust-lang.org/stable/std/ops/trait.Shl.html#tymethod.shl)
qui permet de décaler des bits a gauche autant de fois que tu le souhaites.
Par exemple :
```rust
assert_eq!(0b00000001 << 1, 0b00000010);
assert_eq!(0b00000001 << 2, 0b00000100);
assert_eq!(0b00000001 << 3, 0b00001000);
assert_eq!(0x00_00_00_ff << 8, 0x00_00_ff_00);
```

#### /!\ La taille des nombres /!\

Fais **très attention** à la taille des nombres.
Si tu essaies de décaler un `u8` a gauche de 8 par exemple alors il vaudra zéro a la fin :
```rust
let a: u8 = 0b10101010;
assert_eq!(a << 1, 0b01010100);
assert_eq!(a << 2, 0b10101000);
assert_eq!(a << 3, 0b01010000);
assert_eq!(a << 4, 0b10100000);
assert_eq!(a << 5, 0b01000000);
assert_eq!(a << 6, 0b10000000);
assert_eq!(a << 7, 0b00000000);
assert_eq!(a << 8, 0b00000000);
```

Dans notre cas il va bien falloir penser a convertir les `u8` en `u32` **avant** de les _shifter_ à gauche puis de les ajouter aux `u32`.

#### Fusionner deux `u32` avec un `OR`

Une fois que tu auras réussi a positionner correctement les bits de chaque couleur dans un `u32` il te faudra tous les fusionner ensemble pour générer un seul `u32` final qui comprends toutes les couleurs.
La manière la plus simple de modifier les bits d'un nombre c'est d'utiliser l'opérateur [`BitOr`](https://doc.rust-lang.org/stable/std/ops/trait.BitOr.html).

Il agit de la même manière que le `or` des booléens, mais sur chaque bits des deux nombres :
```rust
assert_eq!(
  0b110110 ||
  0b001000,
  0b111110
//    ^
// ce bit a été modifié grace au or
);

assert_eq!(
  0b00000011 ||
  0b11000000,
  0b11000011
//    ^^^^
// ces bits étaient a zéro dans les deux cas donc ils sont resté a zéro
);
```


### `from_bytes`

La seconde approche n'existe pas dans tous les langages mais est un peu plus
simple a utiliser et consiste a donner directement les bytes du `u32` à rust
sans faire aucune opération.

Il existe trois fonctions en rust pour créer un nombre directement à partir de ses bytes et leurs différence c'est qu'elles ne reçoivent pas les bytes dans le même ordre :
- [`from_be_bytes`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.from_be_bytes) - `assert_eq!(u32::from_be_bytes([0xaa, 0xbb, 0xcc, 0xdd]), 0xaabbccdd)`
- [`from_le_bytes`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.from_le_bytes) - `assert_eq!(u32::from_le_bytes([0xaa, 0xbb, 0xcc, 0xdd]), 0xddccbbaa)`
- [`from_ne_bytes`](https://doc.rust-lang.org/stable/std/primitive.u32.html#method.from_ne_bytes) - Choisi l'ordre que ton ordinateur utilise par défaut

### Property testing

Une fois que tu auras créé une fonction `rgb` et `rgb2` avec chacunes des
approches on va faire du « property testing » juste pour s'assurer que les deux
fonctions sont bien parfaitement équivalente.
Pour rappel le property testing c'est le fait d'écrire un test où l'ont décrit
une propriété que le programme doit respecter puis on laisse la librairie de
test s'occuper de générer des tests toute seule.

#### Installer une librairie de proptest

Ce type de tests ne sont pas supporté par défaut en rust, on va donc avoir besoin d'une librairie pour le faire.
Dans notre cas on va installer `proptest` :
```bash
cargo add --dev proptest
```

Grace au `--dev` on peut préciser a rust que cette librairie ne sera utilisé que par les développeur mais qu'elle est inutile pour notre application principale.

Cette librairie a une documentation de l'API sur [`docs.rs`](https://docs.rs/proptest/latest/proptest/), mais le guide d'utilisation se trouve dans [le livre](https://proptest-rs.github.io/proptest/intro.html).

#### Écrire notre test

```rust
    proptest! {
        #[test]
        fn test_both_rgb(red in 0u8.., green in 0u8.., blue  in 0u8..) {
            assert_eq!(rgb(red, green, blue), rgb2(red, green, blue));
        }
    }
```

Dans leur livre on voit qu'il faut utiliser une macro `proptest!` pour que leurs tests fonctionnent.
Cette macro va nous permettre d'utiliser une nouvelle syntaxe dans notre test :
```rust
        fn test_both_rgb(red in 0u8.., green in 0u8.., blue in 0u8..) {
//                          ^^^^^^^^^        ^^^^^^^^       ^^^^^^^^
// Ici, au lieu de spécifier le type des paramètres, on spécifie à la fois le type ET les valeurs qu'ils peuvent avoir
// Dans notre cas j' ai permis d'utiliser toutes les valeurs possible pour un `u8`, de zéro jusqu'au maximum

            assert_eq!(rgb(red, green, blue), rgb2(red, green, blue));
/// Ici je vérifie que le résultat des fonctions `rgb` et `rgb2` sont bien égaux quelque soit les paramètres `red`, `green` et `blue`
```

On va insérer ce bloc de proptest dans notre bloc de test déjà existant à la fin du fichier :

```rust
#[cfg(test)]
mod test {
    use super::*;
    use proptest::prelude::*;

    #[test]
    fn test_rgb() {
        assert_eq!(rgb(0, 0, 0), 0x00_00_00_00);
        assert_eq!(rgb(255, 255, 255), 0x00_ff_ff_ff);
        assert_eq!(rgb(0x12, 0x34, 0x56), 0x00_12_34_56);
    }

    proptest! {
        #[test]
        fn test_both_rgb(red in 0u8.., green in 0u8.., blue  in 0u8..) {
            assert_eq!(rgb(red, green, blue), rgb2(red, green, blue));
        }
    }
}
```

## 3) Afficher un dégradé de couleur

Maintenant qu'on peut facilement générer des couleurs on va faire ensemble quelque chose d'un peu plus sympa qu'une image noire.

![assets/gradient.png](assets/gradient.png)

On va donc s'attaquer à cette boucle :
```rust
        for i in buffer.iter_mut() {
            *i = 0; // write something more funny here!
        }
```

Au lieu de mettre un zéro dans chaque pixel on va plutôt récupérer l'index au
quel il se trouve et l'utiliser pour mettre une teinte plus ou moins forte de
rouge.

```rust
        // On va avoir besoin de connaître la taille totale du buffer AVANT d'entrer dans la boucle
        let buffer_len = buffer.len();

        // Ici grace au `.enumerate()` on récupère l'index auquel on se trouve dans la boucle en plus du pixel a modifier (précédemment appelé `i`)
        for (idx, pixel) in buffer.iter_mut().enumerate() {
            // On commence par convertir l'index en une valeur qui va de `0` à `1` où `1` sera renvoyé lorsque l'index atteint la taille du buffer.
            // Si on veut c'est un simple pourcentage qui indique notre progression dans tous les pixels à modifier.
            let progression = idx as f64 / buffer_len as f64;

            // En multipliant la `progression` par `u8::MAX` on fait passer cette valeur de `0` à `u8::MAX` (`255`). On peut convertir le tout en `u8`.
            let color = (progression * u8::MAX as f64) as u8;

            // Pour notre dégradé on utilise seulement le canal du rouge
            *pixel = rgb(color, 0, 0);
        }
```

## 4) Simplifier la manière dont on se balade sur les pixels

Si maintenant on voulait faire aller le dégradé de gauche a droite on se rends
compte que c'est assez complexe parce qu'on a aucun moyen de spécifier « le
pixel de droite ».

Dans cette partie on va voir comment se simplifier la vie en créant un type qui
représente notre buffer et qui nous permet de se balader dessus en spécifiant
des coordonnés plutôt qu'un index de pixel :

```rust
buffer[(x, y)] = rgb(color);
```

De la même manière que `minifb` ne peut pas se débrouiller sans connaître la `WIDTH` et la `HEIGHT` de la fenêtre nous allons également en avoir besoin.
Donc la structure qui va représenter ce buffer va contenir au minimum ces deux paramètres et le vrai buffer qu'on est entrain d'abstraire :

```rust
pub struct WindowBuffer {
    width: usize,
    height: usize,

    buffer: Vec<u32>,
}
```

### Créer la structure

Au départ pour créer la structure on a simplement besoin de la `width` et `height` :
```rust
impl WindowBuffer {
    pub fn new(width: usize, height: usize) -> Self {
        Self {
            width,
            height,
            buffer: vec![0; width * height],
        }
    }
}
```

On peut ensuite remplacer la vieille création du buffer par notre nouveau type :
```diff
-   let mut buffer: Vec<u32> = vec![0; WIDTH * HEIGHT];
+   let mut buffer = WindowBuffer::new(WIDTH, HEIGHT);
```

### Afficher la structure

Pour pouvoir la suite des exercices et pour pouvoir déboguer on va implémenter
une manière de visualiser notre tableau au format texte avec des points (`.`)
quand un pixel vaut `0` et un `#` quand la valeur est différente de zéro.

Pour faire cela on va implémenter le trait [`Display`](https://doc.rust-lang.org/stable/std/fmt/trait.Display.html).
C'est le trait qu'utilise rust pour afficher les structures.
Il y a également le trait [`Debug`](https://doc.rust-lang.org/stable/std/fmt/trait.Debug.html) qui peut être intéressant à connaître.
Celui ci est aussi utilisé par rust pour afficher des structures mais plutôt lorsque l'on souhaite déboguer quelque chose.
C'est par exemple la visualisation qui est utilisée par la macro `dbg!`.
Ou lorsqu'on affiche des choses avec la macro `print!`, si on utilise `{}` ça utilisera l'implémentation de `Display` tandis que si on utilise `{:?}` c’est l'implémentation de `Debug` qui sera utilisé.

Voilà un template qui ne fonctionne pas de l'implémentation:
```rust
use std::fmt;

impl fmt::Display for WindowBuffer {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self.buffer[0] {
            0 => f.write_str(".")?,
            _ => f.write_str("#")?,
        }
        f.write_str("\n")?;

        Ok(())
    }
}
```
Ça va être a toi d'écrire le code qui fonctionne et tu pourrais t’aider avec une de ces fonctions :
- [`chunks`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.chunks)
- [`chunks_exact`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.chunks_exact)

Pour t'aider j’ai écris un test que tu peux copier coller dans ton module de `test`s. Il n’utilise pas `proptest` alors il n’y a pas besoin de le mettre dans la macro, il peut aller juste en dessous du `test_rgb()`.
```rust
    #[test]
    fn display_window_buffer() {
        let mut buffer = WindowBuffer::new(4, 4);
        assert_eq!(
            buffer.to_string(),
            "....
....
....
....
"
        );
        buffer.buffer[1] = 1;
        buffer.buffer[3] = 3;
        buffer.buffer[4] = 4;
        buffer.buffer[6] = 6;
        buffer.buffer[9] = 9;
        buffer.buffer[11] = 11;
        buffer.buffer[12] = 12;
        buffer.buffer[14] = 14;
        assert_eq!(
            buffer.to_string(),
            ".#.#
#.#.
.#.#
#.#.
"
        );
    }
```

/!\ Fait attention a l’indentation des chaînes de caractère, c’est très important de ne pas les indenter sinon cela rajoute des espaces dans la chaîne de caractère et elle devient invalide.

### Faire du snapshot testing

Comme on vient de le voir, écrire ce genre de tests est assez fastidieux et ce n’est pas hyper lisible a cause de l’indentation de la première ligne.
Pour simplifier leurs écriture et leurs mise à jour on utilise beaucoup une techinque qui s’appelle le « snapshot testing ».

#### En théorie

Le but des snapshot tests est exactement le même que les tests classiques qu’on écrit avec un `assert_eq!` sauf que la partie droite du tests est générée par rust directement.
Par exemple au lieu d’écrire :
```rust
assert_eq!(2 + 2, 4);
```

On pourrait écrire :
```rust
assert_display_snapshot!(2 + 2, @"4");
```

Sauf qu’ici, le contenu de la chaîne de caractère précédée d’un `@` à été auto-générée par la librairie de snapshot. Ce n’est pas nous qui l’avont écrit.
Et si on décide de changer la partie gauche alors on peut simplement demander a la librairie de re-générer la partie droite.

#### En pratique

Étant donnée que cette approche de test n’est pas supportée par rust directement on va devoir importer une librairie qui le fait pour nous.
La plus utilisée s’appelle `insta`. Je te laisse l’installer et me donner la commande que tu as utilisé.
L’auteur de cette librairie a écrit un mini livre sur le fonctionnement et l’utilisation de insta ici : <https://insta.rs/>

Une fois installée on va également avoir besoin d’un outil qui mets à jour les tests a notre place. Celui ci doit être installé comme une sous commande cargo :
```
cargo install cargo-insta
```

Une fois installé `cargo` va être augmenté d’une nouvelle commande.
On peut maintenant lancer la commande `cargo insta` par exemple pour voir tout ce que cette nouvelle commande peut faire.

Pour le moment nous n’avont rien a faire.
Le workflow général d’utilisation d’`insta` c’est :
1. J’écris mon test avec l’une des macro disponible et je laisse la partie droite vide :
    - `assert_snapshot!("hello", @"");` - `assert_snapshot` est utilisé quand la partie gauche est déjà une string
    - `assert_display_snapshot!(buffer, @"")` - `assert_display_snapshot` va utiliser l’implémentation de `Display` de la partie gauche pour générer la snapshot de droite
    - `assert_debug_snapshot!(buffer, @"")` - `assert_debug_snapshot` va utiliser l’implémentation de `Debug` de la partie gauche pour générer la snapshot de droite
2. Je lance `cargo insta` en mode test : `cargo insta test`
3. Je lance `cargo insta` en mode review : `cargo insta review`
4. Si j’ai le résultat attendu, j’accepte la snapshot. Sinon je la refuse, je corrige mon code puis je reprends a l’étape 2.

Avant d’ajouter plus de code on va commencer par réécrire le test précédent avec `insta` :
```rust
    use insta::assert_snapshot;

    #[test]
    fn display_window_buffer() {
        let mut buffer = WindowBuffer::new(4, 4);
        assert_display_snapshot!(
            buffer.to_string(),
            @""
        );
        buffer.buffer[1] = 1;
        buffer.buffer[3] = 3;
        buffer.buffer[4] = 4;
        buffer.buffer[6] = 6;
        buffer.buffer[9] = 9;
        buffer.buffer[11] = 11;
        buffer.buffer[12] = 12;
        buffer.buffer[14] = 14;
        assert_display_snapshot!(
            buffer.to_string(),
            @""
        );
    }
```

Je te laisse générer les nouvelles snapshots.

#### En conclusion

- C’est rends souvent l’écriture et la maintenance des tests plus rapide
- Il n’y a plus de problème d’indentation, `insta` se débrouille tout seul pour que ce soit le plus lisible possible puis retire les identations en trop au moment de comparer les snapshots
- `insta` n’est utile que quand on veut vérifier la valeur d’une variable. Si on doit comparer deux variables entre elle comme c’est le cas dans notre test `proptest` alors ça ne sert a rien
- Souvent, même sans utiliser proptest, `insta` n’as que peu d’intérêt quand on vérifie une propriété sur des valeurs dans une boucle


### Implémenter les crochets

Maintenant qu'on a la structure et une bonne manière d’écrire des tests il nous
faut une manière d'accéder au contenu du buffer facilement via les crochets.
En rust, les crochets sont des opérateurs :
- [`Index`](https://doc.rust-lang.org/stable/std/ops/trait.Index.html) si on est entrain de lire une valeur
- [`IndexMut`](https://doc.rust-lang.org/stable/std/ops/trait.IndexMut.html) si on est entrain d'écrire une valeur

On peut donc les implémenter sur n’importe quel type.
Dans notre cas on veut implémenter les opérateurs `Index` et `IndexMut` **sur le type** `WindowBuffer` et **pour le type** `(usize, usize)` :
```rust
// Le trait qu’on implémente            Le type sur lequel on l’implémente
//   vvvvvvvvvvvvvvv                     vvvvvvvvvvvv
impl std::ops::Index<(usize, usize)> for WindowBuffer {
//                   ^^^^^^^^^^^^^^
//                Le type sur lequel on l’implémente => Pour savoir a quel endroit mettre cette partie 
//                là il n’y a pas d’autre manière que de lire la documentation du trait `Index`

    type Output = u32; // Le type qui sera retourné par la méthode index. Encore une fois il faut lire la documentation pour savoir que ça existe

// Et finalement la signature de notre méthode avec le paramètre attendu et *une référence* vers le type de retour.
    fn index(&self, (x, y): (usize, usize)) -> &Self::Output {
        // Ici on fait ce qu'on veut mais on doit renvoyer une référence vers notre buffer interne
        &self.buffer[0]
    }
}

// Puis on implémente le second trait, tout est a peu près pareil a part que cette fois ci on renvoie une référence vers `Self::Output`
impl IndexMut<(usize, usize)> for WindowBuffer {
    fn index_mut(&mut self, (x, y): (usize, usize)) -> &mut Self::Output {
        // Ici on fait ce qu'on veut mais on doit renvoyer une référence **mutable** vers notre buffer interne
        &mut self.buffer[0]
    }
}
```

On peut déjà commencer par gérer les cas d’erreurs en ajoutant ce code aux deux méthodes :

```rust
        if x >= self.width {
            panic!(
                "Tried to index in a buffer of width {} with a x of {}",
                self.width, x
            );
        }
        if y >= self.height {
            panic!(
                "Tried to index in a buffer of height {} with a y of {}",
                self.height, y
            );
        }
```

Une fois fait on va pouvoir écrire encore un nouveau type de test. Les tests qui doivent échouer :
```rust
    #[test]
    #[should_panic]
    fn test_bad_index_width() {
        let mut buffer = WindowBuffer::new(4, 4);
        buffer[(0, 5)] = 0;
    }

    #[test]
    #[should_panic]
    fn test_bad_index_height() {
        let mut buffer = WindowBuffer::new(4, 4);
        buffer[(5, 0)] = 0;
    }
```

Ici en mettant allant chercher un `x` supérieur à la largeur de la grille (`width`) le programme doit planter.
Et grace a l’annotation
```rust
    #[should_panic]
```

On peut dire a rust que pour que le test soit considéré comme valide alors **il faut** qu’il plante.

----

Finalement, c’est à toi de finir d’implémenter ces deux fonctions. Voilà un nouveau test a faire passer qui vérifie que ton implémentation fonctionne bien :
```rust
    #[test]
    fn test_index() {
        let mut buffer = WindowBuffer::new(4, 4);
        buffer[(0, 1)] = 1;
        buffer[(0, 3)] = 3;
        buffer[(1, 0)] = 4;
        buffer[(1, 2)] = 6;
        buffer[(2, 1)] = 9;
        buffer[(2, 3)] = 11;
        buffer[(3, 0)] = 12;
        buffer[(3, 2)] = 14;
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        .#.#
        #.#.
        .#.#
        #.#.
        "###
        );
    }
```

Dans ce test cependant, comme je ne fais que donner des valeurs aux cases de la grille je n’utilise jamais l’implémentation d’`Index`, mais toujours celles d’`IndexMut`.
Quelle est la meilleure manière de s’assurer qu’`Index` et `IndexMut` fonctionnent pareil :
1. Quel type de test parmis tous ceux qu’on a vu
2. Écrit le test

### Faire un dégradé qui va de gauche a droite

Maintenant qu’on a toutes ces fonctions on peut enfin faire notre dégradé qui va de gauche a droite au lieu d’aller de haut en bas !

Mais un nouveau problème se pose, on va avoir besoin d’accéder a la `height` et la `width` de notre buffer pour savoir ou est-ce qu’on en est dans le dégradé mais ces valeurs ne sont pas accessible dans notre structure.
Il y a en général deux manières de régler ce problème.

#### Rendre les champs publique

La première manière, qui semble la plus simple et efficace c’est de simplement rendre ces deux champs publique dans notre structure :
```diff
pub struct WindowBuffer {
-    width: usize,
-    height: usize,
+    pub width: usize,
+    pub height: usize,

    buffer: Vec<u32>,
}
```

Maintenant dans le code on peut très simplement accéder a la `width` en écrivant `buffer.width`.
Et en bonus on peut même modifier les dimensions en hauteur et en largeur de notre buffer, top non ?

Et bien non parce que si quelqu’un venait a modifier la `width` sans que l’on ait modifié notre `buffer` alors tout ce qu’on a écrit au dessus va arrêter de fonctionner.
C’est pour ça qu’on choisi assez rarement cette approche finalement.

#### Utiliser des getters et des setters

L’autre approche consiste a définir des `setters` et des `getters`.
Les `getters` sont des méthodes qui, dans notre cas, nous renvoient la `width` et la `height`.
Et les `setters` nous laissent la modifier.

Dans notre cas on ne sait pas trop quoi faire si elles sont modifiée donc on va seulement définir les `getters`.
Voilà à quoi ça pourrait ressembler :
```rust
    pub fn width(&self) -> usize {
        self.width
    }

    pub fn height(&self) -> usize {
        self.height
    }
```

Dans certains langages on pourrait appeler les méthodes `get_width` et `get_height`.
Mais en rust on préfère mettre le nom du champ plutôt que de mettre un `get_` devant toutes nos méthodes.

Pour les `setters` par contre on les aurait plutôt appelé `set_width` et `set_height` je pense. Ça dépends un peu du contexte des fois on voit aussi des `with_width` / `with_height` ou autre.

-----

Une fois ce problème réglé, on peut repartir écrire notre dégradé.
La première étape c’est de retirer le vecteur qu’on utilisait comme buffer avant pour utiliser notre nouveau type :
```diff
-    let mut buffer: Vec<u32> = vec![0; WIDTH * HEIGHT];
+    let mut buffer = WindowBuffer::new(WIDTH, HEIGHT);
```

Et ensuite, pour rappel, actuellement la boucle qui faisait le dégradé ressemblait à ça :
```rust
        // On va avoir besoin de connaître la taille totale du buffer AVANT d'entrer dans la boucle
        let buffer_len = buffer.len();

        // Ici grace au `.enumerate()` on récupère l'index auquel on se trouve dans la boucle en plus du pixel a modifier (précédemment appelé `i`)
        for (idx, pixel) in buffer.iter_mut().enumerate() {
            // On commence par convertir l'index en une valeur qui va de `0` à `1` où `1` sera renvoyé lorsque l'index atteint la taille du buffer.
            // Si on veut c'est un simple pourcentage qui indique notre progression dans tous les pixels à modifier.
            let progression = idx as f64 / buffer_len as f64;

            // En multipliant la `progression` par `u8::MAX` on fait passer cette valeur de `0` à `u8::MAX` (`255`). On peut convertir le tout en `u8`.
            let color = (progression * u8::MAX as f64) as u8;

            // Pour notre dégradé on utilise seulement le canal du rouge
            *pixel = rgb(color, 0, 0);
        }
```

On va la réécrire ligne par ligne :
```rust
        // On va avoir besoin de connaître la taille totale du buffer AVANT d'entrer dans la boucle
        let buffer_len = buffer.len();
```

Maintenant on a plus besoin de connaître la taille totale du buffer mais juste la largeur totale de la fenêtre étant donné qu’on ne va plus qu’aller de gauche a droite.
On peut retirer cette ligne entièrement on utilisera notre nouvelle méthode `width()` pour récupérer la largeur maximale plus tard.

----

```rust
        // Ici grace au `.enumerate()` on récupère l'index auquel on se trouve dans la boucle en plus du pixel a modifier (précédemment appelé `i`)
        for (idx, pixel) in buffer.iter_mut().enumerate() {
```

On ne va plus itérer sur tous les éléments sans savoir ou on en est mais directement sur les `x` et les `y` comme cela :
```rust
        for y in 0..buffer.height() {
            for x in 0..buffer.width() {
```

----

```rust
            // On commence par convertir l'index en une valeur qui va de `0` à `1` où `1` sera renvoyé lorsque l'index atteint la taille du buffer.
            // Si on veut c'est un simple pourcentage qui indique notre progression dans tous les pixels à modifier.
            let progression = idx as f64 / buffer_len as f64;

            // En multipliant la `progression` par `u8::MAX` on fait passer cette valeur de `0` à `u8::MAX` (`255`). On peut convertir le tout en `u8`.
            let color = (progression * u8::MAX as f64) as u8;

            // Pour notre dégradé on utilise seulement le canal du rouge
            *pixel = rgb(color, 0, 0);
```

On doit redéfinir la progression uniquement sur l’axe des `x`.
```rust
            // On commence par convertir l'index en une valeur qui va de `0` à `1` où `1` sera renvoyé lorsqu’on est a droite de l’écran.
            // Si on veut c'est un simple pourcentage qui indique notre progression de la gauche vers la droite.
            let progression = x as f64 / buffer.width() as f64;

            // En multipliant la `progression` par `u8::MAX` on fait passer cette valeur de `0` à `u8::MAX` (`255`). On peut convertir le tout en `u8`.
            let color = (progression * u8::MAX as f64) as u8;

            // Pour notre dégradé on utilise seulement le canal du rouge
            buffer[(x, y)] = rgb(color, 0, 0);
```

----

Et voilà, tout est presque écrit.

Mais si tu essaies de lancer le code tu vas te rendre compte qu’il reste une erreur :
```rust
error[E0308]: mismatched types
   --> src/main.rs:41:35
    |
41  |         window.update_with_buffer(&buffer, WIDTH, HEIGHT).unwrap();
    |                ------------------ ^^^^^^^ expected `&[u32]`, found `&WindowBuffer`
    |                |
    |                arguments to this method are incorrect
    |
    = note: expected reference `&[u32]`
               found reference `&WindowBuffer`
```

Effectivement, `minifb` ne sait pas ce qu’est un `WindowBuffer` et il ne peut pas accéder a son contenu.
À toi de proposer et d’implémenter une solution.

-----

Une fois ce problème résolu tadaaa :

![left to write gradient](assets/left_write_gradient.png)


## 5) Simuler un grain de sable

Ça y est, on à tous les outils pour essayer de simuler notre premier grain de sable !

Encore une fois, la première question a se poser c’est :

### C’est quoi un grain de sable

Ou, plus précisément, « de quoi j’ai besoin pour afficher un grain de sable ».

Pour l’instant dans notre première version on pourrait se dire qu’un grain de sable est un pixel blanc sur un écran noir qui tombe **vers le bas**.
On aurait alors seulement besoin de stocker sa position a l’écran.

```rust
struct Sand {
    x: usize,
    y: usize,
}
```

Et il va falloir une structure qui stockes tous ces grains de sables. En général dans les jeux vidéos on appelle ça le monde, ou « world » en anglais :
```rust
struct World {
    world: Vec<Sand>,
}
```

Ensuite, notre boucle de jeu va se dérouler en trois étapes :
1. On mets à jour le monde
2. On mets à jour le buffer d’affichage
3. On mets à jour l’écran à partir du buffer

### 5.1) Mettre à jour le monde

Dans notre cas mettre à jour le monde ça veut dire : faire descendre les grains de sables.

Autrement dit, on passe sur chaque grain de sable contenu dans le monde, et on **augmente** leurs y.
Et oui parce que attention :
- Quand `x` augmente, le pixel va vers la **droite**.
- Mais quand `y` augmente, le pixel va vers le **bas**.

![l’évolution des `x` et `y` sur une grille en 2D](assets/grid.png)

En sachant ça il devient assez simple de mettre à jours tous les grains de sable :

```rust
impl World {
    pub fn update(&mut self) {
        // on va modifier les grains donc on doit itérer en mode mutable sur les grains de sable
        for sand in self.world.iter_mut() {
            sand.y += 1;
        }
    }
}
```

### 5.2) Mettre à jour le buffer d’affichage

Une fois qu’on a mis à jour la position de tous les grains de sable il faut qu’on mette à jour le buffer d’affichage.
On est dans un cas où on a deux objets qui ont besoin l’un de l’autre.
Il n’y a pas de règle sur comment les faires intéragir ensemble, j’ai décidé arbitrairement que c’était le monde qui allait prendre en paramètre le buffer :
```rust
    pub fn display(&self, buffer: &mut WindowBuffer) {
        todo!();
    }
```

On a pas besoin de se modifier soit même donc on peut prendre `self` par simple référence.
Mais on doit modifier le buffer donc on le prends avec une référence mutable.

----

Le but de cette fonction va être de parcourir chaque grain de sable et mettre un pixel blanc à la position correspondante dans le buffer :
```rust
    pub fn display(&self, buffer: &mut WindowBuffer) {
        for sand in self.world.iter() {
            buffer[(sand.x, sand.y)] = u32::MAX;
        }
    }
```

### Mettre à jour le programme principal

On a tous les outils nécessaires. Il ne reste plus qu’à connecter tous les bouts ensemble dans la boucle principale :

#### Créer le monde

Pour l’instant pour faire simple on va simplement créer un monde qui contient un seul grain de sable :

```rust
    let mut world = World {
        world: vec![Sand { x: WIDTH / 2, y: 0 }],
    };
```

À insérer **avant** la boucle de jeu à côté de la création du buffer et de la fenêtre.
J’ai positionné un grain de sable en haut de l’écran et au centre.

#### Mettre à jour la boucle de jeu

On peut maintenant retirer tout le code lié au dégradé :
```rust
        for y in 0..buffer.height() {
            for x in 0..buffer.width() {
                // On commence par convertir l'index en une valeur qui va de `0` à `1` où `1` sera renvoyé lorsque l'index atteint la taille du buffer.
                // Si on veut c'est un simple pourcentage qui indique notre progression dans tous les pixels à modifier.
                let progression = x as f64 / buffer.width() as f64;

                // En multipliant la `progression` par `u8::MAX` on fait passer cette valeur de `0` à `u8::MAX` (`255`). On peut convertir le tout en `u8`.
                let color = (progression * u8::MAX as f64) as u8;

                // Pour notre dégradé on utilise seulement le canal du rouge
                buffer[(x, y)] = rgb(color, 0, 0);
            }
        }
```

À la place on va mettre à jour le monde :
```rust
        world.update();
```

Puis mettre à jour le buffer a partir du nouveau monde :
```rust
        world.display(&mut buffer);
```

Et finalement on garde le code qu’on avait pour mettre à jour la fenêtre.
Ce qui nous donne au total cette boucle de jeu :
```rust
    while window.is_open() && !window.is_key_down(Key::Escape) {
        world.update();
        world.display(&mut buffer);

        // We unwrap here as we want this code to exit if it fails. Real applications may want to handle this in a different way
        window
            .update_with_buffer(buffer.buffer(), WIDTH, HEIGHT)
            .unwrap();
    }
```

On peut maintenant lancer le code !

![Le premier grain de sable](assets/first_sand_drop.gif)

### Les problèmes

On voit plusieurs problème avec cette vidéo.
1. Au lieu de voir **un grain** de sable tomber on voit une ligne se former.
2. Une fois arrivé en bas de l’écran, la fenêtre se ferme et le programme panique avec cette erreur:
```rust
thread 'main' panicked at src/main.rs:163:19:
Tried to index in a buffer of height 360 with a y of 360
stack backtrace:
   0: rust_begin_unwind
             at /rustc/82e1608dfa6e0b5569232559e3d385fea5a93112/library/std/src/panicking.rs:645:5
   1: core::panicking::panic_fmt
             at /rustc/82e1608dfa6e0b5569232559e3d385fea5a93112/library/core/src/panicking.rs:72:14
   2: <sand_simulation::WindowBuffer as core::ops::index::IndexMut<(usize,usize)>>::index_mut
             at ./src/main.rs:130:13
   3: sand_simulation::World::display
             at ./src/main.rs:163:19
   4: sand_simulation::main
             at ./src/main.rs:31:9
   5: core::ops::function::FnOnce::call_once
             at /rustc/82e1608dfa6e0b5569232559e3d385fea5a93112/library/core/src/ops/function.rs:250:5
note: Some details are omitted, run with `RUST_BACKTRACE=full` for a verbose backtrace.
```

La première chose quand on observe ce genre de problème c’est d’écrire des tests qui nous permette de les reproduire comme ça quoi qu’il arrive :
- On oublie pas ce comportement, bien ou mauvais
- Si on essaie de régler le problème le test est déjà là et nous dira si notre approche a fonctionné
- Il arrive assez fréquemment qu’on essayant de reproduire le bug on n’y arrive pas parce qu’il n’a pas été causéé par ce qu’on croyait. Écrire des tests maintenant nous assures qu’on va s’attaquer a la partie vraiment buggé du code.

#### Reproduire le premier problème

Le test pour le premier problème est simple :
1. On créé le monde avec un grain de sable en haut au centre
2. On affiche le monde dans le buffer
3. On `snapshot` le buffer avec `insta`
4. On répète les étapes `2` et `3` quelque fois

```rust
    #[test]
    fn simple_sand_drop() {
        let mut buffer = WindowBuffer::new(5, 4);
        let mut world = World {
            world: vec![Sand { x: 3, y: 0 }],
        };
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###""###
        );

        world.update();
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###""###
        );

        world.update();
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###""###
        );
    }
```

Je te laisse générer les snapshots, mettre à jour le test et vérifier qu’on observe bien une ligne au lieu d’un seul grain de sable.

#### Reproduire le second problème

Ce qui a causé le second problème est moins clair par contre. Je te laisse écrire le test le plus petit possible qui reproduise le problème.
Tu peux le faire avec un seul appel a `update` et `display` après avoir initialisé le `World`.

#### Résoudre le premier problème

Une fois qu’on a nos deux tests on est prêt a régler les problèmes une fois pour toute.
Le problème de la ligne vient du fait que quand on mets à jour la position d’un grain de sable on ne supprime pas son pixel dans le buffer.
Donc a l’étape suivante on affiche le grain de sable une seconde fois alors que son ancienne pixel est toujours intact dans le buffer.


Il y a plusieurs manière de régler ce problème, je vais en proposer une mais **c’est à toi** d’en trouver une autre.

------

Une solution pourrait être de supprimer **tout le contenu** du buffer **avant** de commencer a écrire les nouveaux grain de sable dedans.
Pour cela on pourrait utiliser une double boucle comme ça :
```rust
    pub fn display(&self, buffer: &mut WindowBuffer) {
        // On remets le buffer a zero avant d’écrire quoi que ce soit dedans
        for x in 0..buffer.width() {
            for y in 0..buffer.height() {
                buffer[(x, y)] = 0;
            }
        }

        for sand in self.world.iter() {
            buffer[(sand.x, sand.y)] = u32::MAX;
        }
    }
```

Mais en général je préfère définir une méthode sur le buffer qui s’en occupe pour moi :

```rust
    pub fn reset(&mut self) {
        self.buffer.fill(0);
    }
```

Quand on l’implémente de ce côté on peut en plus bénéficier des méthodes définie sur les vecteurs ce qui rends la tache plus facile a coder ET plus rapide a exécuter.
Ici j’utilise la méthode [`.fill(0)`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.fill) qui permet de donner à **toutes les valeurs** du vecteur la valeur spécifiée.
Et ensuite la méthode `display` définie précédemment devient aussi plus simple a comprendre et écrire :
```rust
    pub fn display(&self, buffer: &mut WindowBuffer) {
        // On remets le buffer a zero avant d’écrire quoi que ce soit dedans
        buffer.reset();

        for sand in self.world.iter() {
            buffer[(sand.x, sand.y)] = u32::MAX;
        }
    }
```

Une fois implémenté tu peux mettre à jour le test et relancer le code pour vérifier que ça fonctionne.

-----

Maintenant c’est a toi de trouver une autre manière de faire.
Avant de passer trop de temps a l’implémenter n’hésite pas a me demander si ton approche est bonne.

#### Résoudre le second problème

Le second problème vient du fait que dans notre fonction `World::update()` peut faire sortir les grains de sable de l’écran.
```rust
    pub fn update(&mut self) {
        for sand in self.world.iter_mut() {
            sand.y += 1; // ici, si on est déjà en bas de l’écran alors on devrait juste laisser le grain de sable en place
        }
    }
```

Le problème c’est que dans cette fonction on ne connait pas la taille de l’écran et le `y` maximum qui peut exister.
Encore une fois plusieurs solutions sont possible, pour en citer quelques unes :
- On pourrait prendre la `height` en paramètre de la fonction
- On pourrait stocker la `height` dans `self` pour pouvoir y accéder de partout
- On pourrait prendre le buffer en entier en paramètre et en lecture seule (la fonction qui mets à jour le monde ne doit pas mettre à jour le buffer aussi)

Ici j’ai décidé de prendre le buffer en paramètre :
```rust
    pub fn update(&mut self, buffer: &WindowBuffer) {
        for sand in self.world.iter_mut() {
            if sand.y < buffer.height() - 1 {
                sand.y += 1;
            }
        }
    }
```

Et maintenant on peut relancer le tout :
![Second lâché de grain de sable](assets/second_sand_drop.gif)

Et surtout n’oublie pas de mettre à jour ton test avant de passer a la suite !

## 6) Simuler plusieurs grains de sable

Regarder un seul grain de sable tomber c’est pas passionnant.
Au lieu d’en faire mettre un seul a l’initialisation du monde, on va partir d’un monde vide :
```diff
-    let mut world = World {
-        world: vec![Sand { x: WIDTH / 2, y: 0 }],
-    };
+    let mut world = World { world: Vec::new() };
```

Puis les faire tomber a l’infini dans la boucle de jeu :
```rust
    while window.is_open() && !window.is_key_down(Key::Escape) {
        world.world.push(Sand { x: WIDTH / 2, y: 0 });
```

![](assets/inifinite_sand_drop.gif)

### Simuler un grain de sable qui rencontre un autre grain

Un premier problème dans cette simulation c’est que quand deux grains de sables se recontre il ne se passe _rien_.
Celui du haut traverse celui du bas.

Pour mieux comprendre ce qu’il se passe on va faire un nouveau test beaucoup plus simple que la vidéo ci dessus.
Je vais placer 3 grains de sables les uns au dessus des autres et on va les regarder se rencontrer :
```rust
    #[test]
    fn sand_physic() {
        let mut buffer = WindowBuffer::new(5, 4);
        let mut world = World {
            world: vec![
                Sand { x: 2, y: 2 },
                Sand { x: 2, y: 1 },
                Sand { x: 2, y: 0 },
            ],
        };
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        ..#..
        ..#..
        ..#..
        .....
        "###
        );
        world.update(&buffer);
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        .....
        ..#..
        ..#..
        ..#..
        "###
        );
        world.update(&buffer);
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        .....
        .....
        ..#..
        ..#..
        "###
        );
        world.update(&buffer);
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        .....
        .....
        .....
        ..#..
        "###
        );
    }
```

Le test est un peu long mais il permet de bien visualiser le problème.
Au lieu de s’empiler les grains de sables se rentrent dedans et c’est tout bonnement dégueulasse.
Notre objectif c’est qu’ils se touchent mais pas plus avant le marriage et la fonction problématique c’est la mise à jour du monde :
```rust
    pub fn update(&mut self, buffer: &WindowBuffer) {
        for sand in self.world.iter_mut() {
            // ici en plus de vérifier qu’on ne vas pas sortir de l’écran il faudrait
            // aussi vérifier qu’on ne va pas rentrer dans un autre grain de sable
            if sand.y < buffer.height() - 1 {
                sand.y += 1;
            }
        }
    }
```

On pourrait être tenté de rajouter une boucle qui va vérifier que le monde ne contient pas d’autres grain de sable a la position juste en dessous :
```rust
    pub fn update(&mut self, buffer: &WindowBuffer) {
        for sand in self.world.iter_mut() {
            if self
                .world
                .iter()
                // ici on doit donner la _future_ position de grain de sable. On incrémente donc `y`
                .any(|s| (sand.x, sand.y + 1) == (s.x, s.y))
            {
                continue;
            }
            if sand.y < buffer.height() - 1 {
                sand.y += 1;
            }
        }
    }
```

Malheureusement, ce genre de double boucle ne sont pas possible en rust :
```rust
error[E0502]: cannot borrow `self.world` as immutable because it is also borrowed as mutable
   --> src/main.rs:151:16
    |
150 |           for sand in self.world.iter_mut() {
    |                       ---------------------
    |                       |
    |                       mutable borrow occurs here
    |                       mutable borrow later used here
151 |               if self
    |  ________________^
152 | |                 .world
    | |______________________^ immutable borrow occurs here

For more information about this error, try `rustc --explain E0502`.
```

C’est un problème avec les itérateurs de rust, tant qu’on est dans la boucle `for` qui a démarrée avec le `.iter_mut()` il est impossible d’accéder en lecture aux autres éléments du monde.

Une manière simple de régler le problème c’est d’utiliser des index dans le vecteur. Ça fonctionne **tant qu’on ajoute pas d’éléments dans le monde**.
Dans notre cas donc il n’y aura pas de problème.

```rust
    pub fn update(&mut self, buffer: &WindowBuffer) {
        for index in 0..self.world.len() {
            // Je récupère le grain de sable que je veux mettre à jour dès le début. Tu vas devoir dériver `Clone` sur le `Sand`.
            let mut sand = self.world[index].clone();
            // J’en profite pour mettre à jour sa position dès le début. Ce n’est pas un problème puisque je ne modifie qu’une copie.
            sand.y += 1;
            // Ici plus besoin de mettre à jour le y donc
            if self.world.iter().any(|s| (sand.x, sand.y) == (s.x, s.y)) {
                continue;
            }
            // Et ici plus besoin de faire un `-1` sur la `height` puisqu’on l’as déjà fais sur le `y` en amont
            if sand.y < buffer.height() {
                // Et finalement on mets tout le grain de sable à jour plutôt que seulement son `y`
                self.world[index] = sand;
            }
        }
    }
```

On peut relancer les tests et mettre à jour les `snapshots`.
Après toutes les itérations dans notre test on se retrouve dans cette position :
```rust
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        .....
        ..#..
        ..#..
        ..#..
        "###
        );
```

C’est bien mieux, plus de fornication en cachette sous un même pixel !
Mais maintenant on aimerait bien que les grains de sables tombent a droite et a gauche s’il y a de la place de disponible.

### Mais attention au test qu’on écrit

Ici on aurait pu continuer et passer à côté d’un problème subtil assez compliqué a voir en dehors d’un test.
Si l’on change l’initialisation du monde avec les trois même grains de sable mais pas dans le même ordre comme cela :
```rust
        let mut world = World {
            world: vec![
                Sand { x: 2, y: 0 },
                Sand { x: 2, y: 1 },
                Sand { x: 2, y: 2 },
            ],
        };
```

En relançant le tests et en mettant à jour les snapshots on se rends compte qu’elles changent, et pas comme on en a envie :
```rust
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        ..#..
        ..#..
        ..#..
        .....
        "###
        );
        world.update(&buffer);
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        ..#..
        ..#..
        .....
        ..#..
        "###
        );
```

Ici les grains deviennent trop pudique et refusent de se suivre de trop près.
Le problème vient encore une fois de la fonction `World::update`.
Au lieu de mettre à jour les grains de sables dans l’ordre d’insertion (ou l’ordre dans lequel ils apparaissent dans le vecteur).
Il faudrait qu’on les mettes à jour *par le bas*.
C’est à dire qu’on veut d’abord mettre à jour ceux qui ont les `y` les plus **grands**.

![l’évolution des `x` et `y` sur une grille en 2D](assets/grid.png)

Une manière d’implémenter ça pourrait être de parcourir les `y` (ou les lignes) de bas en haut puis pour chaque grains de sable,
s’il n’est pas sur la ligne observée on ne le mets pas à jour.
```rust
    pub fn update(&mut self, buffer: &WindowBuffer) {
        // On parcours les `y` de bas en haut en faisant un itérateur qui va de la `height` jusqu’à 0.
        // Comme on ne peut pas faire de range inversée `buffer.height()..0` on utilise le `.rev()`.
        for y in (0..buffer.height()).rev() {
            for index in 0..self.world.len() {
                let mut sand = self.world[index].clone();
                // On ne mets à jour que les grains de sable qui sont sur la ligne observée
                if sand.y != y {
                    continue;
                }
                sand.y += 1;
                if self.world.iter().any(|s| (sand.x, sand.y) == (s.x, s.y)) {
                    continue;
                }
                if sand.y < buffer.height() {
                    self.world[index] = sand;
                }
            }
        }
    }
```

Maintenant on peut relancer nos tests et constater que les grains de sables se suivent sans jamais se rentrer dedans :
```rust
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        .....
        ..#..
        ..#..
        ..#..
        "###
        );
```

### Faire glisser un grain de sable sur un autre

L’étape suivante serait de faire glisser les grains de sable les uns sur les autres.
Une règle simple est de dire que lorsqu’un grain de sable rencontre un grain de sable en dessous de lui.
Il regarde en bas à gauche ou à droite puis occupe l’espace s’il est disponible.

```
1)
.#.
...
.#.
---
2)
...
.#.
.#.
---
3)  OU
...    ...
...    ...
##.    .##
```

Cette fois ci c’est a toi de le faire, le test qu’on a déjà écrit devrait être suffisant pour vérifier que ça fonctionne.

/!\ Fait bien attention aux bords de l’écran !

Quand on regarde si on peut aller a gauche on risque de sortir de rendre un `usize` négatif et ça crash.
N’hésite pas a utiliser une de ces fonctions pour vérifier le bord gauche :
- [`saturating_sub`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.saturating_sub) qui retire `1` à la valeur OU la laisse a zéro si elle y était déjà.
- [`checked_sub`](https://doc.rust-lang.org/stable/std/primitive.usize.html#method.checked_sub) qui renvoie une option si jamais l’opération n’as pas fonctionné.

Pour vérifier qu’on ne sort pas du bord droit il n’y a pas d’autre manière que de regarder la `width` du buffer.

![Sablier sympa](assets/inifinite_hourglass.gif)

## 7) Ajouter de l’interaction

Le sablier c’est sympa mais on aimerait bien que ce soit un peu plus interactif
donc on va essayer de récupérer les clics de l’utilisateur sur l’écran et les
utiliser pour faire apparaître de nouveaux grains de sable.

### Comprendre l’exemple de `minifb`

Au lieu de lire la documentation j’ai décidé d’aller d’abord voir les exemples disponible sur le repo : <https://github.com/emoon/rust_minifb/tree/master/examples>
Et top, un exemple s’appelle [`mouse.rs`](https://github.com/emoon/rust_minifb/blob/master/examples/mouse.rs).
On va probablement trouver tout ce dont on a besoin dedans!

Ils font pleins de trucs compliqué, j’ai pas lu parce qu’il y avait des maths, et puis soudainement, un bout de code semble parler de la souris [ici](https://github.com/emoon/rust_minifb/blob/ef07f55834d711a88676f011f96f97aae98f3be2/examples/mouse.rs#L43-L53):
```rust
        if let Some((x, y)) = window.get_mouse_pos(MouseMode::Discard) {
            let screen_pos = ((y as usize) * (width / 2)) + x as usize;

            if window.get_mouse_down(MouseButton::Left) {
                buffer[screen_pos] = 0x00ffffff; // white
            }

            if window.get_mouse_down(MouseButton::Right) {
                buffer[screen_pos] = 0x00000000; // black
            }
        }
```

#### `get_mouse_pos`

Maintenant on va devoir lire le code ligne par ligne et regarder ce que chaque fonctions font :
```rust
        if let Some((x, y)) = window.get_mouse_pos(MouseMode::Discard) {
```

La première méthode utilisée est [`get_mouse_pos`](https://docs.rs/minifb/latest/minifb/struct.Window.html#method.get_mouse_pos)
avec le paramètre [`MouseMode`](https://docs.rs/minifb/latest/minifb/enum.MouseMode.html).
La documentation de la méthode `get_mouse_pos` dit :
> Get the current position of the mouse relative to the current window The coordinate system is as 0, 0 as the upper left corner

« Relative to the current window » veut dire que ça nous donne les coordonnée par rapport a notre fenêtre, et avec `0, 0` comme le coin en haut a gauche.
C’est bien c’est exactement le modèle qu’on a suivi actuellement.

Et la signature de la fonction c’est :
```rust
pub fn get_mouse_pos(&self, mode: MouseMode) -> Option<(f32, f32)>;
```

Une première chose étonnante c’est que ça renvoie une `Option`, dans quel cas est-ce qu’on ne pourrait pas trouver les coordonnés de la souris?
L’autre chose étonnante c’est que le tuple de `x`/`y` sont des `f32`. Donc ils n’indiquent pas un pixel précis il semblerait.
Et ça pose la question de savoir ce qu’il représente en réalité ? Quel est le maximum, est-ce qu’ils peuvent être négatif ?


Ensuite on va jeter un œil au paramètre [`MouseMode`](https://docs.rs/minifb/latest/minifb/enum.MouseMode.html).
On voit que c’est un `enum` :
```rust
pub enum MouseMode {
    Pass,
    Clamp,
    Discard,
}
```

Les valeurs possiblent ne nous disent pas grand chose mais en dessous on trouve la documentation pour chaque variante. Je te conseille de les lires.

> Discard
Discared if the mouse is outside the window

Nous, comme pour l’exemple, c’est ce mode qui nous intéresse.
Si la souris n’est pas dans la fenêtre alors ça ne nous intéresse pas et on l’ignore.

----

Moi avec ces informations je comprends que je veux utiliser le paramètre `MouseMode::Discard`.

Mais je ne comprends pas la valeur renvoyée.

Il y a deux possibilités à ce moment là, soit on s’arrête et on teste la fonction, soit on continue de lire l’exemple pour essayer de comprendre.

#### `screen_pos`

```rust
            let screen_pos = ((y as usize) * (width / 2)) + x as usize;
```

Cette ligne converti le couple `x`/`y` en une position qu’ils peuvent écrire dans leurs buffer.
Là on retrouve presque le code qu’on avait écrit au départ. On se serait attendu a quelque chose comme ça plutôt :
```rust
            let screen_pos = ((y as usize) * width) + x as usize;
```

D’où vient ce `/ 2`. Si on regarde un peu le reste de l’exemple alors on se rends compte qu’il y a plusieurs fois des commentaires indiquant :
> Divide by / 2 here as we use 2x scaling for the buffer

Ok donc il semblerait que ça ne nous concerne pas. On peut retirer ce `/ 2`.
Et ce qu’on peut en déduire donc c’est que le `x` et `y` sont des `floats` mais on peut simplement les convertir en `usize` sans plus de difficulté pour récupérer des `x` et `y` en `usize`.

#### `get_mouse_down`

```rust
            if window.get_mouse_down(MouseButton::Left) {
                buffer[screen_pos] = 0x00ffffff; // white
            }

            if window.get_mouse_down(MouseButton::Right) {
                buffer[screen_pos] = 0x00000000; // black
            }
```

Une fois qu’on sait si on se trouve dans la fenêtre ou non et qu’on a les paramètres nécessaire ils utilisent une nouvelle méthode avec un nouveau paramètre.

Encore une fois on va lire la documentation de la méthode [`get_mouse_down`](https://docs.rs/minifb/latest/minifb/struct.Window.html#method.get_mouse_down) indique :
> Check if a mouse button is down or not

Et du paramètre [`MouseButton`](https://docs.rs/minifb/latest/minifb/enum.MouseButton.html) :

> The various mouse buttons that are availible

Là tu pourrais faire une pause pour faire une PR où tu corriges leurs documentation déjà.
Ensuite les différents variants possible sont :
```rust
pub enum MouseButton {
    Left,
    Middle,
    Right,
}
```

C’est assez explicite, avec cette méthode on peut donc savoir si, **en ce moment**, l’utilisateur est entrain de faire un clic droit, un clic gauche ou un clic molette.


### Mettre tout ce qu’on a appris en pratique

On a maintenant toutes les infos necessaires pour faire apparaître un grain de sable à chaque fois que l’utilisateur clique quelque part.
Mais il faut qu’on décide a quel endroit du code on va récupérer les interactions de l’utilisateur.
On va modifier notre boucle de jeu et rajouter ça en première étape :
1. On récupère et on applique les actions utilisateur
2. On mets à jour le monde
3. On mets à jour le buffer d’affichage
4. On mets à jour l’écran à partir du buffer

Encore une fois, j’ai décidé complètement arbitrairement (et je pense que c’est du mauvais code) de gérer les interactions de l’utilisateur dans le `World` directement.
On va donc définir cette nouvelle méthode qui a à la fois besoin du `World` et de la `Window` de `minifb` :
```rust
    pub fn handle_user_input(&mut self, window: &Window) {
        if let Some((x, y)) = window.get_mouse_pos(MouseMode::Discard) {
            if window.get_mouse_down(MouseButton::Left) {
                self.world.push(Sand {
                    x: x as usize,
                    y: y as usize,
                });
            }
        }
    }
```

Puis dans la boucle de jeu :
```rust
    while window.is_open() && !window.is_key_down(Key::Escape) {
        world.handle_user_input(&window);
        world.update(&buffer);
        world.display(&mut buffer);

        // We unwrap here as we want this code to exit if it fails. Real applications may want to handle this in a different way
        window
            .update_with_buffer(buffer.buffer(), buffer.width(), buffer.height())
            .unwrap();
    }
```

Et tadaaa

![](assets/interaction.gif)

## 8) Rendre tout un peu plus réactif

Malgré tous nos efforts on se fait toujours un peu chier dans cette simulation de grain de sable.
Peut être que c’est parce que c’est foncièrement pas très rigolo mais je pense
que mon premier problème c’est qu’il n’y a pas assez de grain de sable qui tombe
quand on clique quelque part donc ici on va voir comment on pourrait créer plusieurs grains de sables à chaque fois qu’on clique quelque part.

Une première version du code pourrait ressembler a ça :
```rust
    pub fn handle_user_input(&mut self, window: &Window) {
        if let Some((x, y)) = window.get_mouse_pos(MouseMode::Discard) {
            if window.get_mouse_down(MouseButton::Left) {
                let (x, y) = (x as usize, y as usize);
                let thickness = 2;
                // Ici on si la personne clique sur les coordonnés (10, 10) on va se balader sur les indexes
                // [
                //  (8, 8),  (9, 8),  (10, 8),  (11, 8),  (12, 8),
                //  (8, 9),  (9, 9),  (10, 9),  (11, 9),  (12, 9),
                //  (8, 10), (9, 10), (10, 10), (11, 10), (12, 10),
                //  (8, 11), (9, 11), (10, 11), (11, 11), (12, 11),
                //  (8, 12), (9, 12), (10, 12), (11, 12), (12, 12),
                // ]
                // Ce qui correspond a un carré autour du coordonné (10, 10).
                // Ça nous permet aussi de facilement régler l’épaisseur du trait si l’on veut le rendre configurable plus tard.
                for x in (x - thickness)..(x + thickness) {
                    for y in (y - thickness)..(y + thickness) {
                        self.world.push(Sand { x, y });
                    }
                }
            }
        }
    }
```

Mais quand je vais trop sur les bords de l’écran j’ai encore des crashs.
À toi de corriger ce code et d’ajouter les conditions nécessaires pour que ça arrête de crasher.

### Pourquoi ça rame

Tu as du te rendre compte en testant qu’assez rapidement le code se mets a ramer.
Si tu ne le fais pas déjà, n’oublie pas que tu peux lancer ton programme en mode release avec la commande :
```
cargo run --release
```
Cette « solution » repousse le problème mais ne le fait pas disparaitre.

----

Les problèmes de performances sont souvent les plus dur a résoudre pour plusieurs raisons :
- Ils peuvent nécessiter de modifier du code déjà complexe pour retirer  des opérations
- Ça prends beaucoup du temps de s’assurer que le code est bien devenu rapide dans *tous les cas* et pas seulement dans le cas qu’on a regardé
- C’est très dur de comprendre **quoi** optimiser

Comme rendre du code plus rapide prends du temps on ne peut pas **tout** rendre plus rapide.
C’est pourquoi la chose la plus important quand on réalise qu’il y a un problème de performance c’est d’en trouver la source !

Une méthode vraiment simpliste mais très chiante pour localiser les problèmes de performances c’est de simplement mesurer a la main le temps qu’on perds dans chaque partie du code.
Par exemple on peut commencer au niveau le plus haut, directement dans la boucle de jeu et regarder laquelle de nos 4 étape prends le plus de temps.
En rust on peut utiliser le type [`Instant`](https://doc.rust-lang.org/stable/std/time/struct.Instant.html) pour représenter un moment dans le temps.
Puis on peut demander combien de temps s’est écoulé depuis qu’on a créé ce type.
Voilà a quoi ressemble notre boucle de jeu si l’on affiche le temps passé dans chaque partie :

```rust
    while window.is_open() && !window.is_key_down(Key::Escape) {
        let now = Instant::now();
        world.handle_user_input(&window);
        println!("get user input took: {:?}", now.elapsed());

        let now = Instant::now();
        world.update(&buffer);
        println!("updating the world took: {:?}", now.elapsed());

        let now = Instant::now();
        world.display(&mut buffer);
        println!("displaying the world: {:?}", now.elapsed());

        let now = Instant::now();
        // We unwrap here as we want this code to exit if it fails. Real applications may want to handle this in a different way
        window
            .update_with_buffer(buffer.buffer(), buffer.width(), buffer.height())
            .unwrap();
        println!("refreshing the screen took: {:?}", now.elapsed());

        println!();
    }
```

Et maintenant lorsqu’on lance le code on voit ou est passé le temps :
```
get user input took: 167ns
updating the world took: 0ns
displaying the world: 107.834µs
refreshing the screen took: 36.975375ms
```

De base on voit que tout vas très vite sauf la fonction qui rafraichit l’écran.
La raison c’est qu’on a dit a `minifb` qu’on ne voulait pas plus de `60` images par secondes donc si on essaie de rafraichir l’écran trop vite il va attendre a notre place.
Une fois que le code se mets a ramer on voit quelque chose de bien différent :
```
get user input took: 167ns
updating the world took: 164.166834ms
displaying the world: 29.75µs
refreshing the screen took: 5.093458ms
```

/!\ Pour rappel, on a un budget d’environ `16ms` pour maintenir du 60 images par secondes.

- Récupérer les interactions utilisateur est toujours plus ou moins instantané et ne pose aucun problème.
- Mettre à jour le monde explose notre budget et prends 10 fois le temps nécessaire a l’affichage. Si c’était notre seule opération on n’afficherait que 6 images par secondes.
- Afficher le monde est instantané aussi.
- Mettre à jour l’écran prends 5ms, étant donné que nous étions en retard c’est probablement le temps minimum pour rafraichir l’écran pour `minifb`, et donc une durée incompressible pour nous.

Ici on a de la chance, le problème est localisé a un seul endroit il semblerait.
Il n’y que la mise à jour du monde qui prends du temps, tout le reste est plutôt rapide.

-----

```rust
    pub fn update(&mut self, buffer: &WindowBuffer) {
        // On parcours les `y` de bas en haut
        for y in (0..buffer.height()).rev() {
            for index in 0..self.world.len() {
                let mut sand = self.world[index].clone();
                // On ne mets à jour que les grains de sable qui sont sur la ligne observée
                if sand.y != y {
                    continue;
                }
                sand.y += 1;
                if sand.y >= buffer.height() {
                    continue;
                }

                // ...
            }
        }
    }
```

Voilà le code sur lequel on s’était arrêté.
Ce qu’il se passe c’est que pour chaque ligne dans notre fenêtre, donc 360 fois (notre fenêtre fait 640x360 pixels), on regarde *TOUS* les grains de sables.
Si l’on pouvait simplement retirer cette première boucle `for` on rendrait donc notre code 360 fois plus rapide ce qui serait suffisant pour revenir à 60 images par seconde dans l’exemple que j’ai mesuré.

À ce moment là il faut bien se souvenir pourquoi nous avions besoin de cette boucle :
On s’était rendu compte que si on ne mettait pas à jour les grains de sable du bas vers le haut alor certains risquaient de ne pas bouger parce qu’ils avaient un autre grains en dessous d’eux : 
```rust
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        ..#..
        ..#..
        ..#..
        .....
        "###
        );
        world.update(&buffer);
        world.display(&mut buffer);
        assert_display_snapshot!(
            buffer.to_string(),
            @r###"
        ..#..
        ..#..
        .....
        ..#..
        "###
        );
```

### Trier les éléments en fonction de leurs hauteur

Il y a probablement plein de solution a ce problème, moi j’ai choisi de simplement trier les éléments en fonction de leurs `y` quand on commence a mettre à jour le monde :
```rust
        self.world.sort_unstable_by_key(|sand| Reverse(sand.y));

        for index in 0..self.world.len() {
            let mut sand = self.world[index].clone();
            // On ne mets à jour que les grains de sable qui sont sur la ligne observée
            sand.y += 1;
            if sand.y >= buffer.height() {
                continue;
            }
```

- [`sort_unstable_by_key`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.sort_unstable_by_key) - Cette fonction permet de trier des éléments mais en spécifiant sur quoi `rust` doit se baser pour trier. Ici on lui indique qu’on veut qu’il utilise `Reverse(sand.y)`
- [`Reverse`](https://doc.rust-lang.org/stable/std/cmp/struct.Reverse.html) - Une structure qui inverse l’ordre de ce qu’on lui donne en paramètre. Pour rust `Reverse(5) < Reverse(10)`.

Question : Pourquoi est-ce qu’il y a des méthodes `sort_unstable_*` et des méthodes `sort_*`, que veut dire le `unstable` et à quoi il sert ?

----

Maintenant le code va beaucoup plus vite et on arrive a afficher beaucoup plus d’éléments très vite.
Mais on se rends compte que le code se mets a ramer encore plus vite puisqu’on est super rapide a rajouter les éléments.

On va donc *encore* ajouter des prints pour voir ou est-ce qu’on perds notre temps.
Pour faire simple j’en ai mis un autour du sort et un autour du reste avec toute la boucle :
```rust
        let now = Instant::now();
        self.world.sort_unstable_by_key(|sand| Reverse(sand.y));
        println!("\tsorting: {:?}", now.elapsed());

        let now = Instant::now();
        for index in 0..self.world.len() {
            let mut sand = self.world[index].clone();
            // ...
        }
        println!("\teverything else: {:?}", now.elapsed());
```

Le `\t` est une indentation, c’est juste pour marquer que ces lignes là sont reliée a celle qui va suivre.
Voilà le résultat quand j’attends que ça rame :
```
get user input took: 125ns
	sorting: 80.792µs
	everything else: 249.679542ms
updating the world took: 249.772417ms
displaying the world: 37.292µs
refreshing the screen took: 367.542µs
```

Le problème vient encore de la boucle. Le tri est plutôt rapide.

### Supprimer les duplicats

Une autre chose qui pourrait ralentir notre code c’est le fait que quand on
passe avec la souris, on passe souvent deux fois au même endroit ce qui génère
des grains de sables exactement identique mais qui doivent être mis à jour deux
fois.

Je ne sais pas a quel point ça prends du temps mais je sais que ça consomme de
la mémoire et que ça ralentit le code donc je vais corriger ça maintenant.

Lorsque l’on insère des éléments on va maintenant vérifier que toutes les positions qu’on utilises ne sont pas déjà utilisée :
```rust
                for x in (x - thickness)..(x + thickness) {
                    for y in (y - thickness)..(y + thickness) {
                        let sand = Sand { x, y };
                        // On ne veut insérer le grain de sable QUE s’il n’est pas déjà présent
                        if !self.world.contains(&sand) {
                            self.world.push(sand);
                        }
                    }
                }
```

Rust se plaint qu’on ne peut pas comparer le type qui représente les grains de sable.
Comme le type ne contient que les coordonnés alors on peut simplement dériver `PartialEq` et `Eq` comme rust le demande :
```rust
#[derive(Debug, Clone, PartialEq, Eq)]
struct Sand {
    x: usize,
    y: usize,
}
```

Avec cette optimisation le code devient encore plus rapide mais j’arrive toujours a le faire ramer assez facilement.

```
get user input took: 125ns
	sorting: 52.875µs
	everything else: 110.765167ms
updating the world took: 110.825ms
displaying the world: 26.667µs
refreshing the screen took: 3.083125ms
```

Et la fonction de gestion des entrées utilisateur n’as pas spécialement ralenti.
En tout cas pas au point de devenir problématique.

### Mesurer les problèmes de performances dans une boucle

Mesurer les performances de *toute* la boucle n’est pas si dur, on regarde l’heure **avant** la boucle puis on regarde combien de temps s’est écoulé **après** la boucle.
Ça nous a permis d’identifier que c’était cette boucle le problème, mais maintenant pour optimiser le contenu de la boucle c’est plus complexe.
Si j’essaie de mettre un print autour de la première opération par exemple :
```rust
            let now = Instant::now();
            let mut sand = self.world[index].clone();
            println!("\t\tgetting the sand grain: {:?}", now.elapsed());
```

(J’ai mis deux identation (`\t`) parce que cette durée représente l’intérieur du print suivant)

On se retrouve avec quelque chose complètement illisible et dont on ne peut rien tirer :
```
...
		getting the sand grain: 41ns
		getting the sand grain: 0ns
		getting the sand grain: 42ns
		getting the sand grain: 0ns
		getting the sand grain: 0ns
		getting the sand grain: 42ns
		getting the sand grain: 0ns
		getting the sand grain: 0ns
		getting the sand grain: 42ns
		getting the sand grain: 41ns
		getting the sand grain: 0ns
		getting the sand grain: 42ns
		getting the sand grain: 0ns
		getting the sand grain: 42ns
		getting the sand grain: 42ns
		getting the sand grain: 0ns
		getting the sand grain: 0ns
		getting the sand grain: 0ns
		getting the sand grain: 0ns
		getting the sand grain: 0ns
		getting the sand grain: 42ns
		getting the sand grain: 0ns
		getting the sand grain: 42ns
		getting the sand grain: 0ns
		getting the sand grain: 0ns
		getting the sand grain: 42ns
		getting the sand grain: 0ns
		getting the sand grain: 0ns
		getting the sand grain: 0ns
	everything else: 85.263041ms
```

C’est normal, puisqu’on fait ça pour chaque grains de sable et qu’on génère des milliers de grains de sables très rapidement on va printer un millier de fois cette ligne.

Une stratégie souvent utilisée pour régler ce problème est de d’additionner tous le temps qui a été passé dans chaque itération de la boucle :

```rust
        let mut get_sand = Duration::default();

        let now = Instant::now();
        for index in 0..self.world.len() {
            let now = Instant::now();
            let mut sand = self.world[index].clone();
            get_sand += now.elapsed();
            // ...
        }
        println!("\t\tgetting the sand grains: {:?}", get_sand);
        println!("\teverything else: {:?}", now.elapsed());
```

Et maintenant on affiche seulement une seule ligne qui représente le temps total dépensé a récupérer des grains de sables :
```
get user input took: 208ns
	sorting: 23.458µs
		getting the sand grains: 78.035µs
	everything else: 89.314083ms
updating the world took: 89.339583ms
displaying the world: 1.140541ms
refreshing the screen took: 298.625µs
displayed 3357 elements
```

Ça ne semble pas être le problème.
On va rajouter des prints en suivant cette technique pour identifier le problème :

À ce moment là on est sur un bout de code que je ne t’ai pas donné donc tu vas devoir vraiment rajouter des prints un peu partout jusqu’à identifier la vraie cause de ton problème.

Mais il y a de bonne chances qu’on ait le même problème.
Dans mon code a un endroit je regarde si un grain de sable est présent a la position où je veux aller, j’ai rajouté des prints comme on a dit autour de cet endroit là :
```rust
                let now = Instant::now();
                // dès qu’on trouve une position libre on s’arrête
                if self.world.iter().all(|s| (sand.x, sand.y) != (s.x, s.y)) {
                    self.world[index] = sand;
                    sand_exists += now.elapsed();
                    break;
                }
                sand_exists += now.elapsed();
```

Et ça m’as permis d’identifié l’endroit qui occupe presque tout mon temps :
```
get user input took: 250ns
	sorting: 35.875µs
		getting the sand grains: 119.587µs
		does the sand already exists: 223.073668ms
	everything else: 225.692875ms
updating the world took: 225.730583ms
displaying the world: 1.079375ms
refreshing the screen took: 3.285875ms
displayed 5318 elements
```

On voit que sur toute la boucle, `223ms` / `225ms` sont perdue a regarder si un grain de sable occupe déjà la position qui m’intéresse.

-----

C’est assez logique puisque ce code là :
```rust
                if self.world.iter().all(|s| (sand.x, sand.y) != (s.x, s.y)) {
```

Peut possiblement itérer sur *TOUS* les grains de sables existant.
Et celà pour *CHAQUE* grain de sable qu’on est entrain de mettre à jour.

C’est ce qu’on appelle un algorithme en `O(n²)` où `n` représente le nombre total de grain de sable.
Donc au plus on ajoute de grain de sable et au plus on doit faire de vérification **de manière exponentielle**.

> La notation `O(...)` indique qu’on mesure la complexité d’un algorithme et qu’on va donner plus ou moins la manière dont le temps d’exécution va varier et en fonction de quoi

![](assets/exponential_notation.png)

Si on trace le graph de `n²` on se rends compte que le temps d’exécution **explose** littéralement avec le nombre de grain de sable qui augmentent.

Comme indiqué par la couleur du graph, c’est de la merde.

---

Notre but maintenant ça va être de faire chuter cette complexité.
Il y a plusieurs manière de faire.
Si tu veux lire quelque chose de bien écrit, mon chercheur préféré a écrit un article que je trouve très réaliste a ce propos : <https://tratt.net/laurie/blog/2023/four_kinds_of_optimisation.html>

Mais pour résumer voilà les 4 types d’optimisations possibles dans l’ordre de la plus simple a la plus complexe :
- Utiliser un meilleur algorithme
- Utiliser une meilleure structure de donnée
- Écrire du code plus bas niveau / proche de la machine
- Accepter une solution moins précise (par exemple ne regarder que quelques grains de sable au lieu de tous les regarder)

Dans notre algorithme, on est donc en `O(n * n)` avec:
1. Le premier `n` qui représente le parcours de TOUS les grains de sable pour les mettres à jour
2. Le second `n` qui représente le parcours de TOUS les grains de sable pour savoir si une place est libre

#### Simplifier le premier `n`

```rust
        for index in 0..self.world.len() {
```

Je ne vois pas beaucoup de manière de simplifier le premier `n`, il est nécessaire de mettre à jour tous les grains de sable pour les voir bouger.
Mais si on prends un peu de recul sur le problème est-ce qu’on veut _vraiment_ mettre à jour **tous** les grains de sable ?
En réalité on sait que lorsqu’un grain de sable touche le sol, il ne sera plus jamais mis à jour.
Une idée pour réduire de beaucoup le premier `n` pourrait être de faire un vecteur avec les grains de sable qui ne seront plus jamais mis à jour et un autre vecteur avec les grains de sable qui doivent être mis à jour.

Cependant, même avec cette approche le second `n` reste inchangé, on doit toujours regarder dans les deux vecteurs pour savoir si une place est libre.
J’ai décidé de ne pas poursuivre cette approche parce que j’arrive a faire ramer la simulation avant même qu’un seul grain de sable atteigne le sol.


/!\ Si a ce moment là tu penses à une autre solution je suis très intéressé de l’entendre. J’en ai une seconde qui serait vraiment efficace mais trop compliqué a implémenter pour le moment.


#### Simplifier le second `n`

```rust
                if self.world.iter().all(|s| (sand.x, sand.y) != (s.x, s.y)) {
```

Je vois beaucoup plus de manière de travailler autour du second `n`.

##### En changeant l’algorithme

En changement uniquement l’algorithme on pourrait par exemple utiliser le fait que la liste de grain de sable est déjà triée sur les `y` pour aller chercher directement le `y` qui nous intéresse sans parcourir tous les grains de sable.
On peut utiliser une technique qui s’appelle la « dichotomie » ou la « binary search » en anglais qui consiste a chercher de la même manière qu’on chercherais un mot dans un dictionnaire :
1. J’ouvre le dictionnaire au centre et je regarde si le mot que je cherche est a droite ou a gauche
2. J’ouvre les pages restante au centre et je regarde encore une fois si je dois aller dans la partie gauche ou droite
3. Et ainsi de suite jusqu’à ce que je trouve le bon mot

C’est par exemple la technique qu’on utilise pour gagner au jeu où il faut deviner un nombre entre `0` et `100`.
Si le nombre est `72` par exemple alors on va demander les valeurs dans cet ordre :
1. `50` - la moitié de la range `0-100`
2. `75` - la moitié de la range `50-100`
3. `62` - la moitié de la range `50-75`
4. `68` - la moitié de la range `62-75`
5. `71` - la moitié de la range `68-75`
6. `73` - la moitié de la range `71-75`
7. `72` - la moitié de la range `71-73`

Cet exemple représente le **pire** cas possible.
Souvent on croise le nombre pendant qu’on fait la recherche avant que la range ne contienne plus qu’un seul élément.

Ce qui est important d’observer avec cet exemple c’est qu’entre chaque itération on **divise par deux** la quantité de valeur a tester.
D’une certaine manière on fait l’inverse d’une puissance de 2 :
Au lieu de faire `n * 2 * 2 * 2 * 2` pour `n⁴` on fait `n / 2 / 2 / 2 / 2`.
C’est ce qu’on appelle un logarithme en base 2.
L’inverse de la puissance de 2.

Et on peut le vérifier assez facilement : [`log2(100) = 6.6`](https://numbat.dev/?q=log2%28100%29%E2%8F%8E)
Nous étions dans le pire cas possible et on a bien eu besoin de regarder `6` valeurs avant de trouver la bonne.

-----

Si on utilisait cette technique notre code pour chercher le grain de sable qui nous intéresse alors on pourrait faire passer notre complexité de :
`O(n * n)` à `O(n * log2(n))`. C’est bien mieux mais ça monte toujours assez vite :
![](assets/n_log2_n.png)

C’est toujours un peu compliqué de se rendre compte a quel point une courbe monte vite en fonction du niveau de zoom et d’autre choses.
J’ai dessiné les deux sur le même graph et la différence n’est toujours pas flagrante :

![](assets/n_n_vs_n_log2_n_zoomed.png)

Mais si l’on dézoome beaucoup. Par exemple jusqu’à ce que l’une des deux courbes atteigne `20_000` grains de sable ce qui semble être une valeur raisonnable alors voilà ce que ça donne :

![](assets/n_n_vs_n_log2_n_dezoomed.png)

On se rends compte que l’algorithme en `O(n²)` n’as presque pas bougé et ne vas jamais attendre les `20_000`.
Même si le `O(n * log2(n))` semble toujours augmenter très vite il est infiniement meilleur que notre premier algorithme.

----


Pour savoir si cette optimisation est suffisante pour nous j’ai essayé de trouver le temps que prenais une comparaison a s’exécuter environ dans notre cas.
J’ai ajouté un print dans le code et je me suis rendu compte que quand j’avais environ `20_000` grains de sable cette boucle prenait environ `120ms` à s’exécuter.
À partir de ces deux valeurs une simple division nous donne l’information qu’on veut : [`120ms / 20_000² -> ns = 0.3ns`](https://numbat.dev/?q=120ms+%2F+20_000%C2%B2+-%3E+ns%E2%8F%8E)
Une comparaison prends environ `0.3ns`.

Ça nous donne une base pour travailler.

Nous on voudrait que notre logiciel soit capable de supporter le fait que la moitié de l’écran soit rempli de grain de sable.
Étant donné que notre résolution est du `640 * 360`, la moitié de cette valeur nous donne `115_200` :
Notre code explose alors qu’on est même pas a un cinquième de ce qu’on aimerait supporter !!!

Maintenant qu’on connait le temps d’exécution d’une compairaison on peut d’ailleur mesurer combien de temps l’algorithme actuel prendrait a gérer autant de points :
[`115_200 * 115_200 * 0.3ns -> s = 3.9s`](https://numbat.dev/?q=115_200+*+115_200+*+0.3ns+-%3E+s%E2%8F%8E)

C’est complètement impossible de laisser ça comme ça. 
Avec notre nouvel algorithme qui utilise la dichotomie en `O(n * log2(n))` ça nous donnerait : [`115_200 * log2(115_200) * 0.3ns -> ms = 0.5ms`](https://numbat.dev/?q=115_200+*+log2%28115_200%29+*+0.3ns+-%3E+s%E2%8F%8E115_200+*+log2%28115_200%29+*+0.3ns+-%3E+ms%E2%8F%8E)
Un temps complètement raisonnable.


Avec une résolution d’écran beaucoup plus grande comme celle de l’écran de mon mac par exemple : `3456 × 2234`.
La moitié de l’écran représente `3_860_352` grains de sable et prendrait un total de [`3_860_352 * log2(3_860_352) * 0.3ns -> ms = 25.3ms`](https://numbat.dev/?q=115_200+*+log2%28115_200%29+*+0.3ns+-%3E+s%E2%8F%8E115_200+*+log2%28115_200%29+*+0.3ns+-%3E+ms%E2%8F%8E3456+%C3%97+2234%E2%8F%8E3456+%C3%97+2234+%2F+2%E2%8F%8E3_860_352+*+log2%283_860_352%29+*+0.3ns%E2%8F%8E3_860_352+*+log2%283_860_352%29+*+0.3ns+-%3E+ms%E2%8F%8E).

Un peu trop long mais on va partir sur ça puisque c’est notre première idée et qu’elle est simple a implémenter.

#### Implémentation

En rust l’algorithme de dichotomie est déjà présent dans la librairie standard, on ne vas pas avoir besoin de le coder.
Il se trouve sur les slice sous le nom de [`binary_search`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.binary_search).

Pour pouvoir chercher dans tous les éléments ceci dit, on va avoir besoin que notre tableau de grain de sable soit _complètement_ trié.
Et pas uniquement les `y`.
Parce que sinon une fois qu’on trouve l’endroit dans le tableau ou un `y` nous intéresse la dichotomie ne saura plus si elle doit aller a gauche ou a droite pour trouver le `x` correspondant :

```
[{ y: 0, x: 1000 }, { y: 0, x: 543 }, { y: 0, x: 0 }]
```

En ne triant que sur les `y` on peut se retrouver avec ce tableau genre de tableau.
Et imaginons qu’on cherche a savoir si le grain de sable `{ y: 0, x: 5 }` existe, une fois que la dichotomie arrive au centre elle n’as aucune information sur s’il faut aller a gauche ou a droite pour trouver un `x` plus petit ou plus grand.

On va donc devoir modifier cette ligne au début de notre fonction pour y inclure la gestion du `x` :
```rust
        self.world.sort_unstable_by_key(|sand| Reverse(sand.y));
```

On va abandonner notre méthode `sort_unstable_by_key` pour une méthode plus bas niveau : [`sort_unstable_by`](https://doc.rust-lang.org/stable/std/primitive.slice.html#method.sort_unstable_by).
Celle ci nous permet de complètement maitriser la manière dont on compare deux éléments plutôt que de simplement renvoyer un objet et laisser rust faire la comparaison.
Elle nous passe en paramètre deux objets et on renvoie si celui de gauche est plus grand, plus petit ou égal a celui de droite.

```rust
        self.world.sort_unstable_by(|left, right| {
            left.y.cmp(&right.y).reverse().then(left.x.cmp(&right.x))
        });
```

Rust nous permet de comparer des éléments s’ils implémentent le trait [`Ord`](https://doc.rust-lang.org/stable/std/cmp/trait.Ord.html).
Et la plupart des types de base de rust dont tous les nombres entiers implémentent ce trait.
1. Comme avant, on commence par comparer les `y` et on les inverses pour avoir les éléments du bas en premier : `left.y.cmp(&right.y).reverse()`
2. *Puis*, SI les `y` étaient identique, alors on compare les `x` : `.then(left.x.cmp(&right.x))`

Ensuite on peut supprimer notre boucle pour utiliser la dichotomie, une fois encore on doit utiliser le `by` avec les même arguments pour comparer deux grains de sables :
```rust
                if self
                    .world
                    .binary_search_by(|s| s.y.cmp(&sand.y).reverse().then(s.x.cmp(&sand.x)))
                    .is_err()
```

Cette fois ci la closure ne prends qu’un seul paramètre puisque c’est nous qui spécifiont le grain de sable qu’on observe.
Ici chez moi il s’appelait `sand`.

#### Implémenter `Ord` soit même

Ici on a répété du code très important, si le `sort` du début devait un jour se retrouver désynchroniser avec la `binary_search` plus bas on aurait des erreurs très dure a déboguer.
Une manière simple de régler ce problème est d’implémenter nous même le trait `Ord` sur notre type puis d’utiliser les fonctions de trie et de dichotomie normale sans aucun paramètre :

```rust
// Rust nous demande d’implémenter `PartialOrd` pour avoir le droit d’implémenter `Ord`
impl PartialOrd for Sand {
    fn partial_cmp(&self, other: &Self) -> Option<std::cmp::Ordering> {
        Some(self.cmp(other))
    }
}

impl Ord for Sand {
    fn cmp(&self, other: &Self) -> std::cmp::Ordering {
        self.y.cmp(&other.y).reverse().then(self.x.cmp(&other.x))
    }
}
```

Une fois implémenté on peut tout simplement retirer nos `_by` dans le reste du code :
```diff
-        self.world.sort_unstable_by(|left, right| {
-            left.y.cmp(&right.y).reverse().then(left.x.cmp(&right.x))
-        });
+        self.world.sort_unstable();
```

Puis :
```diff
-                if self
-                    .world
-                    .binary_search(|s| s.y.cmp(&sand.y).reverse().then(s.x.cmp(&sand.x)))
-                    .is_err()
+                if self.world.binary_search(&sand).is_err() {
```

### On a fini d’implémenter tout ce qu’on voulait !

Et voilà, avec cette optimisation on ne devrait plus avoir de problème avant un moment n’est-ce pas ?

Et bien non, de mon côté j’arrive toujours a faire ramer le code relativement vite :
```
get user input took: 54.257ms
	sorting: 3.876875ms
		getting the sand grains: 1.966495ms
		does the sand already exists: 39.817456ms
	everything else: 78.6265ms
updating the world took: 82.508708ms
displaying the world: 2.011958ms
refreshing the screen took: 3.379292ms
displayed 85413 elements
```

Mais au lieu de devenir lent après `20_000` éléments le code est devenu lent au bout de `85_000` éléments !
On peut se dire qu’il est devenu 4 fois plus rapide en seulement une dizaine de ligne de modifiée.

Mais on ne peut s’empêcher d’être déçu puisque dans nos estimations notre code n’aurait plus jamais eu besoin d’optimisation.
C’est quelque chose de très commun en informatique quand on optimise quelque chose.
Il faut toujours bien vérifier qu’on obtient le résultat auquel on s’attends.

Ici on peut faire quelques observations sur ce qu’il s’est passé :
- Faire apparaitre de nouveau grains de sable est devenu super lent : 54ms
- Trier tous les grains de sable est également devenu beaucoup plus lent qu’avant sans être problématique pour autant : 3.8ms
- Le morceau de boucle qu’on a optimisée est toujours trop lente a : 39ms
- Le reste de la boucle a également pris `78ms - 39ms = 39ms` alors qu’avant c’était plus ou moins instantané 

La manière dont on optimise du code se fait souvent comme ça :
1. On repère le bout de code qui ralentit tout
2. Après avoir réfléchi très fort on arrive a s’en débarrasser ou le rendre beaucoup plus rapide
3. Le code n’est pas beaucoup plus rapide qu’avant mais de nouveau bout de code sont devenu lent

En fait on se retrouve a faire une chasse aux « bottleneck ».
Le bout le plus lent du code. Et à chaque fois qu’on en supprime un, un autre apparaît.

----

On pourrait passer des heures a essayer d’optimiser ce code et il y a énormément de choses a faire.
Si jamais tu as des idées d’optimisation parles en moi et je pourrais te guider pour les implémenter et voir a quel point elle nous rende plus rapide.

Mais pour ce projet on va s’arrêter a ce niveau d’optimisation et passer au dernier chapitre.

## 9) Rendre le tout joli

La dernière étape du projet sera de revenir aux bases et de rendre le tout plus joli.
Si tu te souviens bien, au tout début je t’ai fais implémenter des fonctions qui servaient a la gestion de couleur mais ensuite on ne les as jamais utilisée.

Tout ça c’était pour maintenant, on va rendre les grains de sable coloré !

La première chose a faire c’est d’ajouter la couleur dans la structure qui représente un grain de sable :
```rust
#[derive(Debug, Clone, PartialEq, Eq)]
struct Sand {
    x: usize,
    y: usize,

    color: u32,
}
```

Cela va casser beaucoup de bout de code, je te laisse les corriger.
Pour le moment on va rendre les grains de sable jaune.
Le jaune est un mélange de rouge et de vert ce qui veut dire que la valeur associée doit être : `0x00ffff00`
Ou, en utilisant ta fonction : `rgb(u8::MAX, u8::MAX, 0)`.
Utilise la valeur que tu préfères partout pour corriger ton code.

Ensuite, tu vas constater que quand tu lances le code, les grains de sables sont toujours blanc.
C’est parce qu’on a oublié de mettre à jour la fonction qui représente le monde dans le buffer.
Je te laisse mettre à jour cette fonction en affichant la couleur stocké dans les grains de sable.

### Faire varier les couleurs

Le jaune c’est sympa mais ça ne rends pas le tout bien plus joli.
On va maintenant faire varier la couleur des grains de sable au fil du temps.
Il y a énormément de manière de faire ça mais j’ai décidé que je voulais faire quelque chose comme ça :
- Au départ les grains de sable sont rouge : `rgb(u8::MAX, 0, 0)`
- Au fur et a mesure ils deviennent jaune => on incrémente le vert
- Puis ils deviennent vert => on décrémente le rouge
- Puis on incrémente le bleu
- Puis on décrémente le vert
- Puis on incrémente le rouge
- Puis on décrémente le bleu

Et on recommence a l’infini.

Dans ce genre de cas ma technique c’est de découper le problème en plus petit problème.
On sait qu’on va avoir un cycle qui se répète a l’infini et que ce cycle sera le même pour toutes les couleurs. Simplement désynchronisé.

Si on regarde uniquement le rouge alors il commence a `u8::MAX` puis :
1. Il doit rester rouge pendant `u8::MAX` itération : `std::iter::repeat(u8::MAX).take(u8::MAX as usize)`
2. Ensuite on fait redescendre le rouge a zéro : `(0..=u8::MAX).rev()`
3. On reste a zéro le temps que le bleu augmente, puis que le vert descende : `std::iter::repeat(0).take(u8::MAX as usize * 2)`
4. On augmente jusqu’à `u8::MAX` : `(0..u8::MAX)` 
5. On reste a `u8::MAX` le temps que le bleu redescende : `std::iter::repeat(u8::MAX).take(u8::MAX as usize)`

Étant donné que la première et la dernière opération sont les même on peut les mélanger en : `std::iter::repeat(u8::MAX).take(u8::MAX as usize * 2)`.

Ensuite, grace a l’opérateur [`.chain()`](https://doc.rust-lang.org/stable/std/iter/trait.Iterator.html#method.chain) on peut relier des itérateurs ensemble les uns après les autres et ça nous donne :
```rust
    let channel = (0..u8::MAX) // On monte jusqu’à u8::MAX de 1 en 1
        .chain(std::iter::repeat(u8::MAX).take(u8::MAX as usize * 2)) // On reste a u8::MAX PENDANT u8::MAX itération pour que l’autre channel puisse nous rejoindre
        .chain((0..=u8::MAX).rev()) // On redescend jusqu’à 0
        .chain(std::iter::repeat(0).take(u8::MAX as usize * 2)) // On reste a 0 pendant u8::MAX * 2
        .cycle(); // On répète tout ça a l’infini
```

Maintenant il faut relier cet itérateur sur trois couleurs de manière désynchronisée :
- Le rouge commence après `u8::MAX` itération : `channel.skip(u8::MAX as usize * 2)`
- Le vert commence immédiatement et augmente : `channel`
- Le bleu commence en plein milieu de la pause et doit rester a zéro pendant un moment : `channel.skip(u8::MAX as usize * 4)`

Ensuite pour mélanger tout ça on va utiliser la méthode [`zip`](https://doc.rust-lang.org/stable/std/iter/fn.zip.html) qui lie les valeurs de deux itérateurs dans un tuple.
Ici on va d’abord lier le rouge et le vert ce qui vas nous donner un itérateur de `(u8, u8)`.
Puis on va relire le bleu a cet itérateur ce qui va finalement générer un `((u8, u8), u8)`.
Comme ce n’est pas très pratique a manipuler on va le transformer en un seul tuple de 3 éléments grace a map : `.map(|((r, g), b)| (r, g, b))`

```rust
    let colors = channel
        .clone()
        .skip(u8::MAX as usize * 2)
        .zip(channel.clone())
        .zip(channel.clone().skip(u8::MAX as usize * 4))
        .map(|((r, g), b)| (r, g, b))
```

Ensuite on va stocker cet itérateur dans le monde, le seul qui connait l’état de tout le logiciel mais un nouveau problème se pose :

#### Quel est le type de cet itérateur

Si j’essaie de définir une fonction qui créé tout cet itérateur pour moi et que je laisse rust me donner le type de retour alors voilà ce qu’on observe :
```rust
fn color_generator() {
    let channel = (0..u8::MAX) // On monte jusqu’à u8::MAX de 1 en 1
        .chain(std::iter::repeat(u8::MAX).take(u8::MAX as usize * 2)) // On reste a u8::MAX PENDANT u8::MAX itération pour que l’autre channel puisse nous rejoindre
        .chain((0..=u8::MAX).rev()) // On redescend jusqu’à 0
        .chain(std::iter::repeat(0).take(u8::MAX as usize * 2)) // On reste a 0 pendant u8::MAX * 2
        .cycle(); // On répète tout ça a l’infini

    let colors = channel
        .clone()
        .skip(u8::MAX as usize * 2)
        .zip(channel.clone())
        .zip(channel.clone().skip(u8::MAX as usize * 4))
        .map(|((r, g), b)| (r, g, b));

    colors
}
```

L’erreur renvoyée indique :
```rust
error[E0308]: mismatched types
   --> src/main.rs:271:5
    |
257 | fn color_generator() {
    |                     - help: a return type might be missing here: `-> _`
...
269 |         .map(|((r, g), b)| (r, g, b));
    |              ------------- the found closure
270 |
271 |     colors
    |     ^^^^^^ expected `()`, found `Map<Zip<Zip<Skip<...>, ...>, ...>, ...>`
    |
    = note: expected unit type `()`
                  found struct `Map<Zip<Zip<Skip<Cycle<Chain<Chain<Chain<Range<u8>, Take<Repeat<u8>>>, Rev<RangeInclusive<u8>>>, Take<...>>>>, ...>, ...>, ...>`
            the full type name has been written to '/Users/irevoire/sand_simulation/target/debug/deps/sand_simulation-e0a21a8a3878e2a3.long-type-8255841897920051058.txt'
```

Un type tellement compliqué que même rust abandonne son écriture : `Map<Zip<Zip<Skip<Cycle<Chain<Chain<Chain<Range<u8>, Take<Repeat<u8>>>, Rev<RangeInclusive<u8>>>, Take<...>>>>, ...>, ...>, ...>`

Pourtant nous on sait que ce qui nous intéresse c’est simplement quelque chose qui implémente le trait `Iterator<Item = (u8, u8, u8)`.
Et bien c’est ce qu’on va dire a rust en écrivant exactement ce type de retour a la fonction

```rust
fn color_generator() -> impl Iterator<Item = (u8, u8, u8)> {
```

Cette écriture nous permet de dire à rust qu’on a aucune idée du type qu’on renvoie, mais qu’à la fin il va bien implémenter ce que le trait qu’on souhaite.


Mais ensuite nouveau problème, lorsqu’on essaie d’écrire la même chose dans la structure `World` cette fois ci on se retrouve avec un message d’erreur beaucoup plus clair :
```rust
struct World {
    world: Vec<Sand>,
    colors: impl Iterator<Item = (u8, u8, u8)>,
}
```

Et l’erreur :

```rust
error[E0562]: `impl Trait` only allowed in function and inherent method argument and return types, not in field types
   --> src/main.rs:165:13
    |
165 |     colors: impl Iterator<Item = (u8, u8, u8)>,
    |             ^^^^^^^^^^^^^^
```

En effet, il est interdit d’utiliser cette syntaxe dans une structure.
Et il y a en fait une bonne raison, rust doit pouvoir stocker l’itérateur dans la structure, mais puisqu’on ne connait pas nous même son type, lui non plus ne le connait pas et il ne sait pas de combien de mémoire il aura besoin dans la structure.

La manière la plus simple de résoudre le problème c’est de donner un type qui a une taille connue mais qui peut contenir des types dont on ne connait pas la taille : La [`Box`](https://doc.rust-lang.org/stable/std/boxed/struct.Box.html).
Un type bien pratique qui permet juste de pointer vers une structure quelque part quelque soit sa taille.

Le `World` devient alors :
```rust
struct World {
    world: Vec<Sand>,
    colors: Box<dyn Iterator<Item = (u8, u8, u8)>>,
    //          ^^^ ici le `dyn` est obligatoire et signifie qu’on ne connait pas la taille de ce qui suit
}
```

Ensuite on change l’initialisation :
```rust
    let mut world = World {
        world: Vec::new(),
        colors: Box::new(color_generator()),
    };
```

Puis on change l’ajout de grain de sable par l’utilisateur :
```rust
                        let (r, g, b) = self.colors.next().unwrap();
                        let sand = Sand {
                            x,
                            y,
                            color: rgb(r, g, b),
                        };
```
