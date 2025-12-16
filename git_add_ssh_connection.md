To use your SSH key for signing commits in GitHub, follow these steps:

1. Generate or Use an Existing SSH Key

If you don’t already have an SSH key, generate one using the following command:
```
ssh-keygen -t ed25519 -C "your_email@example.com"
```

This creates a private key (id_ed25519) and a public key (id_ed25519.pub) in your ~/.ssh directory. Keep the private key secure.

2. Add the Public Key to GitHub

Copy the contents of your public key to the clipboard:
```
cat ~/.ssh/id_ed25519.pub | pbcopy # macOS
cat ~/.ssh/id_ed25519.pub | clip # Windows
```

Log in to GitHub and navigate to Settings > SSH and GPG keys.

Click New SSH Key.

Paste the public key into the "Key" field, give it a descriptive title, and select Signing Key as the type.

Click Add SSH Key.
