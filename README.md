# Gemini CLI Extension: Rocket.Chat Apps Generator

This is a Gemini CLI extension that provides natural language support and specialized skills for building Rocket.Chat Apps.

## Installation

If you are running the `gemini-cli` from source:

1. Clone this repository locally.
2. Navigate to your built `gemini-cli` project (e.g., `cd C:\dev\gemini-cli`).
3. Link this extension by providing its absolute path:
   ```bash
   node bundle/gemini.js extensions link C:\absolute\path\to\gemini-cli-rc-apps
   ```
4. Accept the security prompt:
   ```text
   Extensions can execute arbitrary code. Please review the extension's source code...
   Do you want to continue? [Y/n]: Y
   Extension "rc-apps" linked successfully and enabled.
   ```
5. Verify it loaded correctly by running:
   ```bash
   node bundle/gemini.js extensions list
   ```

## Included Skills

- **rc-slash-command**: Specialized knowledge for generating Rocket.Chat Apps Engine slash commands with accurate type coverage and robust error handling.
