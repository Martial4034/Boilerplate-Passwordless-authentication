# Bisourivage Authentication System

## Table des matières

-   [Introduction](#introduction)
-   [Technologies Utilisées](#technologies-utilis%C3%A9es)
-   [Structure du Projet](#structure-du-projet)
-   [Fonctionnalités Principales](#fonctionnalit%C3%A9s-principales)
-   [Configuration](#configuration)
-   [API Routes](#api-routes)
-   [Composants](#composants)
-   [Email Templates](#email-templates)

## Introduction

Ce projet implémente un système d'authentification sans mot de passe pour Bisourivage, en utilisant Next.js, NextAuth, Firebase Authentication, Firestore, et Resend pour l'envoi des emails.

## Technologies Utilisées

-   **Next.js**: Framework React pour le développement web.
-   **NextAuth.js**: Solution d'authentification pour Next.js.
-   **Firebase Authentication**: Service de gestion des utilisateurs.
-   **Firestore**: Base de données NoSQL pour stocker les informations des utilisateurs.
-   **Resend**: Service d'envoi d'emails.
-   **Tailwind CSS**: Framework CSS pour la stylisation.
-   **Shadcn UI**: Bibliothèque de composants UI.

## Structure du Projet



````
.
├── README.md
├── components.json
├── local.env
├── next-env.d.ts
├── next.config.js
├── package-lock.json
├── package.json
├── pages
│   └── api
│       └── auth
│           └── [...nextauth].ts
├── postcss.config.js
├── public
│   ├── next.svg
│   └── vercel.svg
├── src
│   ├── app
│   │   ├── SessionProvider.tsx
│   │   ├── api
│   │   │   ├── checkUser
│   │   │   │   └── route.ts
│   │   │   └── sendSignInEmail
│   │   │       └── route.ts
│   │   ├── auth
│   │   │   ├── signin
│   │   │   │   └── page.tsx
│   │   │   └── signin-confirm
│   │   │       └── page.tsx
│   │   ├── components
│   │   │   ├── EmailTemplates.tsx
│   │   │   └── ui
│   │   │       └── CustomAlertDialog.tsx
│   │   ├── favicon.ico
│   │   ├── firebase.ts
│   │   ├── firebaseAdmin.ts
│   │   ├── globals.css
│   │   ├── layout.tsx
│   │   └── page.tsx
│   ├── components
│   │   └── ui
│   │       ├── alert-dialog.tsx
│   │       └── button.tsx
│   └── lib
│       └── utils.ts
├── tailwind.config.js
├── tailwind.config.ts
└── tsconfig.json
```` 

## Fonctionnalités Principales

### Authentification

-   **Inscription sans mot de passe**: Un utilisateur s'inscrit uniquement avec son email, et un mot de passe aléatoire est généré côté serveur.
-   **Connexion sans mot de passe**: Un utilisateur se connecte en recevant un lien de connexion par email.

### Stockage des Informations

-   **Firestore**: Utilisé pour stocker des informations supplémentaires sur les utilisateurs, telles que l'email, la date de création du compte, le rôle, la dernière connexion, l'IP et un compteur d'emails envoyés.

### Envoi d'Emails

-   **Resend**: Utilisé pour envoyer des emails personnalisés pour l'inscription et la connexion.

### Vérification de l'Existence de l'Utilisateur

-   **API checkUser**: Vérifie si un utilisateur existe déjà dans Firebase Authentication avant de procéder à l'inscription ou à la connexion.

## Configuration

### Fichier .env

Créez un fichier `.env` à la racine du projet et ajoutez les variables d'environnement suivantes :

````
PROJECT_NAME=Bisou Rivage
NEXTAUTH_URL=http://localhost:3000
NEXTAUTH_SECRET=SuperSecret
NEXT_PUBLIC_BASE_URL=http://localhost:3000
RESEND_API_KEY=

#Firebase CLIENT CONFIGURATION
FIREBASE_CLIENT_API_KEY=
FIREBASE_CLIENT_AUTH_DOMAIN=
FIREBASE_CLIENT_PROJECT_ID=
FIREBASE_CLIENT_STORAGE_BUCKET=
FIREBASE_CLIENT_SMS_SENDER_ID=
FIREBASE_CLIENT_APP_ID=
FIREBASE_CLIENT_MEASUREMENT_ID=

#Firebase ADMIN SDK
FIREBASE_ADMIN_PROJECT_ID=your_firebase_project_id
FIREBASE_ADMIN_PRIVATE_KEY_ID=your_firebase_private_key_id
FIREBASE_ADMIN_PRIVATE_KEY=your_firebase_private_key
FIREBASE_ADMIN_CLIENT_EMAIL=your_firebase_client_email
FIREBASE_ADMIN_CLIENT_ID=your_firebase_client_id
FIREBASE_ADMIN_AUTH_URI=https://accounts.google.com/o/oauth2/auth
FIREBASE_ADMIN_TOKEN_URI=https://oauth2.googleapis.com/token
FIREBASE_ADMIN_AUTH_PROVIDER_X509_CERT_URL=https://www.googleapis.com/oauth2/v1/certs
FIREBASE_ADMIN_CLIENT_X509_CERT_URL=https://www.googleapis.com/robot/v1/metadata/x509/your_service_account_email
```` 

## API Routes

### Check User

**Fichier**: `src/app/api/checkUser/route.ts`

Cette route vérifie si un utilisateur existe déjà dans Firebase Authentication.



```js 
import { NextRequest, NextResponse } from 'next/server';
import { authAdmin } from '@/app/firebaseAdmin';

export async function POST(req: NextRequest) {
  const { email } = await req.json();

  try {
    const userRecord = await authAdmin.getUserByEmail(email);
    return NextResponse.json({ userExists: true, user: userRecord }, { status: 200 });
  } catch (error) {
    if (error.code === 'auth/user-not-found') {
      return NextResponse.json({ userExists: false }, { status: 200 });
    } else {
      return NextResponse.json({ message: 'Internal server error' }, { status: 500 });
    }
  }
}
```

### Send Sign In Email

**Fichier**: `src/app/api/sendSignInEmail/route.ts`

Cette route génère un lien de connexion ou d'inscription et envoie un email à l'utilisateur.

typescript

Copier le code

```js 
import { NextRequest, NextResponse } from 'next/server';
import { Resend } from 'resend';
import { authAdmin, firestoreAdmin, FieldValue } from '@/app/firebaseAdmin';
import { SignUpEmailTemplate, SignInEmailTemplate } from '@/app/components/EmailTemplates';

const resend = new Resend(process.env.RESEND_API_KEY);

export async function POST(req: NextRequest) {
  const { to, email, isSignUp } = await req.json();

  try {
    let link;
    let template;
    if (isSignUp) {
      // Générer un mot de passe aléatoire
      const password = Array(24).fill('0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz').map(x => x[Math.floor(Math.random() * x.length)]).join('');

      // Créer un nouvel utilisateur
      const userRecord = await authAdmin.createUser({
        email: email,
        password: password,
      });

      // Créer un document dans la collection users avec les informations nécessaires
      await firestoreAdmin.collection('users').doc(userRecord.uid).set({
        email: email,
        createdAt: FieldValue.serverTimestamp(),
        role: 'user',
        lastLogin: FieldValue.serverTimestamp(),
        lastEmailSent: FieldValue.serverTimestamp(), // Champs pour traquer le dernier mail envoyé
        emailCount: 1, // Compteur de mails envoyés
        ip: req.headers.get('x-forwarded-for') || req.headers.get('x-real-ip'),
      });

      // Utiliser le template d'email d'inscription
      template = SignUpEmailTemplate;

      // Générer le lien de connexion
      const actionCodeSettings = {
        url: `${process.env.NEXTAUTH_URL}/auth/signin-confirm`,
        handleCodeInApp: true,
      };

      link = await authAdmin.generateSignInWithEmailLink(email, actionCodeSettings);
    } else {
      // Générer le lien de connexion pour l'utilisateur existant
      const actionCodeSettings = {
        url: `${process.env.NEXTAUTH_URL}/auth/signin-confirm`,
        handleCodeInApp: true,
      };

      link = await authAdmin.generateSignInWithEmailLink(email, actionCodeSettings);

      // Mettre à jour le document de l'utilisateur avec les informations nécessaires
      const userRecord = await authAdmin.getUserByEmail(email);
      const userDocRef = firestoreAdmin.collection('users').doc(userRecord.uid);

      await userDocRef.update({
        lastEmailSent: FieldValue.serverTimestamp(),
        emailCount: FieldValue.increment(1),
        lastLogin: FieldValue.serverTimestamp(), // Mise à jour de la dernière connexion
      });

      // Utiliser le template d'email de connexion
      template = SignInEmailTemplate;
    }

    // Envoyer l'email avec Resend
    const { data, error } = await resend.emails.send({
      from: `${process.env.PROJECT_NAME} <onboarding@laubier.online>`,
      to: [to],
      subject: isSignUp ? 'Bienvenue sur Bisourivage' : 'Votre lien de connexion',
      text: '',
      react: template({ url: link, email }),
    });

    console.log('Sending Email with URL:', link);

    if (error) {
      return NextResponse.json(error, { status: 400 });
    }

    return NextResponse.json(data, { status: 200 });
  } catch (error) {
    console.error('Error sending email:', error);
    return NextResponse.json({ message: 'Internal server error' }, { status: 500 });
  }
}
```

## Composants

### CustomAlertDialog

**Fichier**: `src/app/components/ui/CustomAlertDialog.tsx`

Ce composant est une boîte de dialogue réutilisable pour afficher des messages d'alerte.

```ts import {
  AlertDialog,
  AlertDialogAction,
  AlertDialogCancel,
  AlertDialogContent,
  AlertDialogDescription,
  AlertDialogFooter,
  AlertDialogHeader,
  AlertDialogTitle,
} from "@/components/ui/alert-dialog";

interface CustomAlertDialogProps {
  isOpen: boolean;
  onClose: () => void;
  title?: string;
  description?: string;
  actionText?: string;
  onActionClick?: () => void;
}

export function CustomAlertDialog({
  isOpen,
  onClose,
  title,
  description,
  actionText,
  onActionClick,
}: CustomAlertDialogProps) {
  return (
    <AlertDialog open={isOpen} onOpenChange={onClose}>
      <AlertDialogContent>
        <AlertDialogHeader>
          <AlertDialogTitle>{title}</AlertDialogTitle>
          <AlertDialogDescription>{description}</AlertDialogDescription>
        </AlertDialogHeader>
        <AlertDialogFooter>
          <AlertDialogCancel onClick={onClose}>Cancel</AlertDialogCancel>
          <AlertDialogAction onClick={onActionClick}>{actionText}</AlertDialogAction>
        </AlertDialogFooter>
      </AlertDialogContent>
    </AlertDialog>
  );
}
```

## EmailTemplates

**Fichier**: `src/app/components/EmailTemplates.tsx`

Les templates d'emails pour l'inscription et la connexion.

```js 
import * as React from 'react';

interface EmailTemplateProps {
  url: string;
  email: string;
}

export const SignUpEmailTemplate: React.FC<Readonly<EmailTemplateProps>> = ({ url, email }) => (
  <div style={{ fontFamily: 'Arial, sans-serif', lineHeight: 1.6 }}>
    <h1>Bienvenue sur Bisourivage</h1>
    <p>Merci de vous être inscrit. Cliquez sur le lien ci-dessous pour vous connecter :</p>
    <a href={url} style={{ display: 'inline-block', padding: '10px 20px', margin: '10px 0', fontSize: '18px', color: '#fff', backgroundColor: '#007bff', textDecoration: 'none' }}>Se connecter</a>
    <p>Si vous n'avez pas demandé cette connexion, ignorez cet email.</p>
    <p>Email: {email}</p>
  </div>
);

export const SignInEmailTemplate: React.FC<Readonly<EmailTemplateProps>> = ({ url, email }) => (
  <div style={{ fontFamily: 'Arial, sans-serif', lineHeight: 1.6 }}>
    <h1>Votre lien de connexion</h1>
    <p>Utilisez le lien ci-dessous pour vous connecter :</p>
    <a href={url} style={{ display: 'inline-block', padding: '10px 20px', margin: '10px 0', fontSize: '18px', color: '#fff', backgroundColor: '#007bff', textDecoration: 'none' }}>Se connecter</a>
    <p>Si vous n'avez pas demandé cette connexion, ignorez cet email.</p>
    <p>Email: {email}</p>
  </div>
);
``` 

## Conclusion

Ce système d'authentification utilise plusieurs technologies pour offrir une expérience utilisateur fluide et sécurisée. L'architecture inclut des vérifications robustes pour l'existence des utilisateurs, des mécanismes de connexion sans mot de passe et des envois d'emails personnalisés pour différents scénarios (inscription et connexion).
