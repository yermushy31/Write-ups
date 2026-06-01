# Write-Up : Root-Me – Programmation TCP – ROT13

## Le challenge

On doit se connecter à un serveur TCP, qui nous envoie une chaîne encodée en ROT13, et on a **2 secondes** pour lui renvoyer la version décodée. Clairement impossible à faire à la main, donc il faut automatiser.

---

## Comprendre ROT13

ROT13 c'est un simple décalage de 13 lettres dans l'alphabet. Le truc sympa c'est que c'est **son propre inverse** : encoder et décoder c'est la même opération.

```
'Uryyb' → décalage de 13 → 'Hello'
```

---

## La solution

J'ai écrit un script Python qui :
1. Se connecte au serveur
2. Attend le message
3. Extrait la chaîne encodée avec une regex
4. La décode et renvoie la réponse dans les 2 secondes

### Le décodage ROT13

```python
def decrypt_rot(self, message, shift=13):
    decipher = ""
    for b in message:
        if 65 <= b <= 90:                          # majuscules
            decipher += chr((b - 65 + shift) % 26 + 65)
        elif 97 <= b <= 122:                       # minuscules
            decipher += chr((b - 97 + shift) % 26 + 97)
        else:
            decipher += chr(b)                     # autres caractères inchangés
    return decipher
```

### La boucle de réception

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

On reçoit le message, on attrape la chaîne entre guillemets simples, on décode et on renvoie. Tout ça se fait bien en dessous des 2 secondes imposées.

---

## Résultat

Le serveur valide la réponse et renvoie le flag. La contrainte des 2 secondes ne pose aucun problème avec un script automatisé, Python répond quasi instantanément.
