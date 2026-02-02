---
author: "Felle"
title: "Valkyr: A simple password manager"
date: "2026-02-01"
description: "A straightforward password manager built with Flutter that keeps your credentials on your device without cloud sync or subscriptions."
tags: [
    "Flutter",
    "Privacy",
    "Security",
]
categories: [
    "Projects",
    "Mobile Development",
]
series: ["Projects"]
---

Valkyr is a password manager that stores your credentials locally on your device. No cloud sync, no account creation, no monthly fees. Just a simple app for managing passwords.
<!--more-->

## What It Does

The app lets you create, edit, and delete password entries. Each entry stores a title, username, password, and optional notes. That's the entire feature set. No browser extensions, no autofill integration, no password sharing between devices.

When you need a password, open the app, find the entry, and copy what you need. When you create a new account somewhere, add a new entry to Valkyr. When you change a password, edit the existing entry.

## Why Another Password Manager

Most password managers want you to store your passwords in their cloud, install their browser extensions, and often pay a subscription. Some of these services are good and worth the money. But sometimes you just want a basic password list that stays on your phone.

Valkyr exists for people who want local storage without the complexity of full-featured password managers. If you need cloud sync across devices, password sharing with team members, or automatic form filling, this isn't the right tool. But if you want a simple encrypted list of passwords on your phone, it works fine.

## Security Approach

The app encrypts your password database locally. Your master password unlocks the database when you open the app. The encryption happens on your device using standard cryptographic libraries.

This approach has obvious limitations. If you lose your device without backups, your passwords are gone. If you forget your master password, there's no recovery mechanism. If someone gets physical access to your unlocked phone, they can access your passwords.

These aren't bugs, they're design decisions. Local-only storage means local-only security. You're responsible for backing up your data and keeping your device secure.

## Using It

On first launch, you set a master password. This password encrypts your database, so choose something strong that you'll remember. The app doesn't validate password strength or require specific characters - that's your decision.

After setup, you see your password list. It's empty initially. Tap the add button to create your first entry. Fill in the title, username, password, and any notes you want to keep. Save it.

The entry appears in your list. Tap it to view details. From the detail view, you can copy the username or password to your clipboard, edit the entry, or delete it.

That's the entire workflow. There are no categories, no tags, no folders, no search filters. Just a list of passwords ordered by when you created them.

## Technical Implementation

Valkyr uses Flutter and Dart with Material Design 3 for the interface. The encryption library handles the cryptographic operations for securing the password database. Local storage keeps everything on device.

The architecture is straightforward. A master password unlocks the encrypted database. Password entries get encrypted before storage and decrypted when needed. The UI provides basic CRUD operations for managing entries.

## Backup Responsibility

Since everything stays local, you need to handle backups yourself. If you wipe your phone or lose your device, your passwords are gone unless you've backed them up.

The app stores its database in your device's app data directory. On Android, this typically isn't accessible without root unless you use backup tools. Regular device backups through your phone's backup system should include the app data, but verify this works before depending on it.

Some users export their database periodically and store it somewhere safe. Others rely on device-level backup solutions. Either way, it's your responsibility to ensure you don't lose your passwords.

## What's Missing

Valkyr doesn't have password generation built in. If you need to create a strong random password, use a separate tool or another app like Flipz. This might seem like an obvious missing feature for a password manager, but adding it would expand the scope beyond basic storage.

There's no biometric unlock. You enter your master password every time you open the app. This is more secure in some ways and less convenient in others.

No password strength indicators, no breach checking, no expiration reminders. The app stores what you give it without judgment or advice.

## Platform Support

Currently, Valkyr targets Android. Flutter supports multiple platforms, so iOS and desktop versions are possible. But the initial focus is Android because that's what I use.

Expanding to other platforms would be straightforward from a code perspective. The main considerations are platform-specific security implementations and ensuring the encryption works correctly across different environments.

## When to Use This

Valkyr makes sense if you want a simple password list on your phone without external dependencies. If you don't need passwords synced across devices, don't want browser integration, and prefer local storage over cloud services, this works.

It doesn't make sense if you need those features password managers typically provide. No judgment either way - use whatever fits your requirements.

## Contributing

The project is open source under GPL v3.0. If you find bugs, want to add features, or have suggestions, the repository accepts issues and pull requests.

Adding features like password generation, categories, or search functionality would improve the app for some users. Porting to other platforms would make it accessible to more people. Security enhancements are always welcome as long as they maintain the local-only approach.

## Download

Get the latest APK from the [GitHub releases page](https://github.com/felle-dev/valkyr-app/releases). Install it on your Android device, set up your master password, and start adding your passwords.

The source code is available in the repository if you want to verify the security implementation or build it yourself.

This is a basic password manager that does what it says: stores passwords on your device. Nothing more, nothing less.
