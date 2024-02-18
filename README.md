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
