# Build/Install Instructions

```bash
# Install the extension package creator
npm install -g @vscode/vsce

# Create the package
vsce package

# Install the extension from the package
code --install-extension [package-name-and-version].vsix

# Example
code --install-extension ./betterfountain-1.15.0.vsix
```