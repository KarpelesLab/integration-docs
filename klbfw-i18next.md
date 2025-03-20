# KLB Framework i18n Integration Guide

This guide explains how to enable internationalization (i18n) on your KLB Framework site using i18next.

## Overview

The KLB Framework automatically handles translation file generation from your source CSV files. This integration uses [i18next](https://www.i18next.com/) with our custom KLB backend to provide a simple yet powerful translation system.

## Setup Process

### 1. Prepare Translation Files

1. Create a directory structure for your translations:
   ```
   etc/
     └── i18n/
         └── your-translations.csv
   ```

2. Format your CSV files with the following structure:
   - First column **must** be named `token`
   - Add one column per language (e.g., `en-US`, `fr-FR`, `ja-JP`)
   - Each row represents a translation token and its values in different languages

   Example CSV:
   ```csv
   token,en-US,fr-FR,ja-JP
   welcome,"Welcome to our site","Bienvenue sur notre site","当サイトへようこそ"
   about_us,"About Us","À propos de nous","会社概要"
   contact,"Contact","Contact","お問い合わせ"
   ```

### 2. Install Required Packages

Add i18next and our KLB backend to your project:

```bash
npm install i18next @karpeleslab/i18next-klb-backend
```

### 3. Initialize i18next in Your Application

Add this to your main JavaScript file:

```javascript
import i18next from 'i18next';
import { Backend } from '@karpeleslab/i18next-klb-backend';
import { getLocale } from "@karpeleslab/klbfw";

// Initialize i18next
i18next
  .use(Backend)
  .init({
    lng: getLocale(),
    fallbackLng: false,
    load: 'currentOnly',
    interpolation: {
      escapeValue: false // not needed for react
    }
  });
```

### 4. Using Translations in Your Code

Basic usage:

```javascript
// Get translated text
const welcomeText = i18next.t('welcome');

// With variables
const greeting = i18next.t('hello_name', { name: 'John' });
// CSV: "hello_name","Hello, {{name}}!"

// Changing language (if needed)
i18next.changeLanguage('fr-FR');
```

## Advanced Features

### Language Detection

The system automatically uses the language set in `FW.Locale`. This is typically managed by the KLB Framework based on user preferences or browser settings.

## Troubleshooting

- **Missing Translations**: Check your CSV files for the correct format and ensure the token exactly matches what you're using in code. The server system will also apply tokens from the I18N files. Tokens which aren't defined will be returned as `[I18N:<token>]`.
- **Loading Issues**: The system will look for translations in `/l/<language>/_special/locale.json`. This is handled automatically by the KLB Framework.
- **Language Codes**: Always use the full 5-character language code format (e.g., `en-US`, not just `en`).

## Best Practices

1. Use meaningful token names grouped by feature
2. Keep translations concise and contextual
3. Always include at least the default language (usually `en-US`)
4. Test your site with different languages enabled
5. Consider using variables for dynamic content rather than creating multiple similar tokens
