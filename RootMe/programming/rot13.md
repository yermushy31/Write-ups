# Write-Up : Root-Me – ELF x86 – Programmation TCP ROT

## Le challenge

On se connecte à un serveur TCP qui nous envoie un mot chiffré en ROT, et on doit lui renvoyer le mot déchiffré. Il faut automatiser ça car le serveur enchaîne les rounds rapidement.

Un message typique reçu ressemble à ça :

```
Decrypt this: 'Uryyb'
```

Et on doit répondre `Hello`.

---

## Ma solution

J'ai écrit un petit script Python qui se connecte au serveur, récupère chaque message, extrait le mot chiffré avec une regex, le déchiffre et renvoie la réponse.

### Le déchiffrement

Le serveur utilise du ROT13, donc j'ai codé une fonction qui parcourt chaque caractère et applique le décalage :

```python
def decrypt_rot(self, message, shift=13):
    decipher = ""
    for b in message:
        if 65 <= b <= 90:
            decipher += chr((b - 65 + shift) % 26 + 65)
        elif 97 <= b <= 122:
            decipher += chr((b - 97 + shift) % 26 + 97)
        else:
            decipher += chr(b)
    return decipher
```

Le truc pratique avec Python 3, c'est qu'itérer sur un objet `bytes` donne directement des entiers ASCII, donc pas besoin de conversion supplémentaire.

### La boucle principale

```python
def receive(self):
    while True:
        data = self.sock.recv(BUFFER_SIZE)
        if not data:
            break
        match = re.search(b"'([a-zA-Z0-9]+)'", data)
        if match:
            to_decode = match.group(1)
            a = self.decrypt_rot(to_decode, 13)
            self.msgsend(a)
```

À chaque message reçu, j'extrais le mot entre guillemets simples, je le déchiffre et je renvoie la réponse. Le `\n` dans `msgsend` est important, le serveur en a besoin pour valider.

---

## Résultat

Après quelques rounds, le serveur renvoie le flag. Rien de très compliqué une fois qu'on a identifié que c'était du ROT13 et qu'on automatise la boucle.
