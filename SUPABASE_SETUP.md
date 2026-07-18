# Activer la synchronisation multi-appareils

SoloEMDR peut synchroniser tes souvenirs, dossiers et réglages entre appareils via un compte, avec **chiffrement de bout en bout** : le contenu est chiffré dans ton navigateur avant d'être envoyé, avec une clé dérivée de ton mot de passe. Ni Supabase, ni personne d'autre que toi ne peut lire le contenu déchiffré.

Cette fonctionnalité est **désactivée par défaut**. Sans configuration, l'app continue de fonctionner exactement comme avant (stockage local uniquement).

## 1. Créer un projet Supabase (gratuit)

1. Va sur https://supabase.com et crée un compte / connecte-toi.
2. Crée un nouveau projet (choisis un mot de passe de base de données, région au choix).
3. Une fois le projet créé, va dans **Project Settings > API**. Note :
   - `Project URL` (ex: `https://xxxxxxxx.supabase.co`)
   - `anon public` key (une longue chaîne de caractères — c'est une clé publique, normal qu'elle soit visible côté client)

## 2. Créer la table et les règles de sécurité

Dans le tableau de bord Supabase, ouvre **SQL Editor**, colle ce qui suit, et exécute :

```sql
create table public.vaults (
  user_id uuid primary key references auth.users(id) on delete cascade,
  salt_pw text not null,
  iv_pw text not null,
  wrapped_dek_pw text not null,
  salt_rec text not null,
  iv_rec text not null,
  wrapped_dek_rec text not null,
  data_iv text not null,
  ciphertext text not null,
  updated_at timestamptz not null default now()
);

alter table public.vaults enable row level security;

create policy "select own vault" on public.vaults
  for select using (auth.uid() = user_id);

create policy "insert own vault" on public.vaults
  for insert with check (auth.uid() = user_id);

create policy "update own vault" on public.vaults
  for update using (auth.uid() = user_id);
```

Ces règles garantissent que chaque compte ne peut lire/écrire que sa propre ligne — jamais celle d'un autre utilisateur.

## 3. Configurer le lien de réinitialisation de mot de passe

Dans **Authentication > URL Configuration**, ajoute l'URL de ton site (ex: `https://sriracha-star.github.io/solo-emdr/`) dans **Redirect URLs**. C'est nécessaire pour que le lien "mot de passe oublié" reçu par email ramène bien vers l'app.

Optionnel : dans **Authentication > Providers > Email**, tu peux désactiver "Confirm email" si tu ne veux pas avoir à confirmer ton adresse par email avant de pouvoir te connecter (usage personnel).

## 4. Renseigner les clés dans le code

Dans `index.html`, cherche ces deux lignes (au début du `<script>`) :

```js
const SUPABASE_URL = 'SUPABASE_URL_HERE';
const SUPABASE_ANON_KEY = 'SUPABASE_ANON_KEY_HERE';
```

Remplace par tes vraies valeurs récupérées à l'étape 1, sauvegarde, et redéploie (commit + push). La synchronisation s'active automatiquement dès que ces deux valeurs sont renseignées.

## Comment ça marche

- **Créer un compte** (icône réglages > Compte & synchronisation) génère une clé de chiffrement unique, l'enveloppe une fois avec ton mot de passe et une fois avec une **phrase de récupération** affichée une seule fois — à noter en lieu sûr.
- Tes souvenirs/dossiers/réglages actuels sont automatiquement envoyés (chiffrés) vers ton compte à la création.
- Sur un autre appareil, se connecter avec le même compte télécharge et déchiffre tes données.
- Si un appareil a déjà des données locales différentes au moment de la connexion, l'app demande explicitement lesquelles garder (aucune perte silencieuse).
- Mot de passe oublié : le lien email permet de définir un nouveau mot de passe, mais la phrase de récupération reste indispensable pour redéchiffrer les données existantes. **Sans le mot de passe ET sans la phrase de récupération, les données sont perdues définitivement** — c'est le prix du chiffrement de bout en bout.
